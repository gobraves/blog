---
type: post
title: tokio internal(1)
date: 2023-06-12
categories: Rust
---
## tokio runtime 基本组件
```rust
  // tokio/src/runtime/runtime.rs
  #[cfg(feature = "rt-multi-thread")]
          #[cfg_attr(docsrs, doc(cfg(feature = "rt-multi-thread")))]
          pub fn new() -> std::io::Result<Runtime> {
              Builder::new_multi_thread().enable_all().build()
          }
```
```rust
  // tokio/src/runtime/builder.rs
  pub fn build(&mut self) -> io::Result<Runtime> {
    match &self.kind {
      Kind::CurrentThread => self.build_current_thread_runtime(),
      #[cfg(all(feature = "rt-multi-thread", not(tokio_wasi)))]
      Kind::MultiThread => self.build_threaded_runtime(),
    }
  }
```
```rust
  // tokio/src/runtime/builder.rs
  fn build_threaded_runtime(&mut self) -> io::Result<Runtime> {
      use crate::loom::sys::num_cpus;
      use crate::runtime::{Config, runtime::Scheduler};
      use crate::runtime::scheduler::{self, MultiThread};
  
      let core_threads = self.worker_threads.unwrap_or_else(num_cpus);
  
      let (driver, driver_handle) = driver::Driver::new(self.get_cfg())?;
  
      // Create the blocking pool
      let blocking_pool =
      blocking::create_blocking_pool(self, self.max_blocking_threads + core_threads);
      let blocking_spawner = blocking_pool.spawner().clone();
  
      // Generate a rng seed for this runtime.
      let seed_generator_1 = self.seed_generator.next_generator();
      let seed_generator_2 = self.seed_generator.next_generator();
  
      let (scheduler, handle, launch) = MultiThread::new(
        core_threads,
        driver,
        driver_handle,
        blocking_spawner,
        seed_generator_2,
        Config {
            before_park: self.before_park.clone(),
            after_unpark: self.after_unpark.clone(),
            global_queue_interval: self.global_queue_interval,
            event_interval: self.event_interval,
            #[cfg(tokio_unstable)]
            unhandled_panic: self.unhandled_panic.clone(),
            disable_lifo_slot: self.disable_lifo_slot,
            seed_generator: seed_generator_1,
            metrics_poll_count_histogram: self.metrics_poll_count_histogram_builder(),
        },
      );
  
      let handle = Handle { inner: scheduler::Handle::MultiThread(handle) };
  
      // Spawn the thread pool workers
      let _enter = handle.enter();
      launch.launch();
  
      Ok(Runtime::from_parts(Scheduler::MultiThread(scheduler), handle, blocking_pool))
  }
```
从这段代码可以看到
1. 创建了线程池
2. 创建drive，drive handle，猜测这个就是和网络，时间控制器绑定，用于后续的事件通知,之后看猜的对不对
3. 生成了两个随机种子
4. 创建scheduler handle和lunch
5. 包装了下handle，然后执行`handle.enter()`
6. 执行了`launch.launch()`

但其实并不明白以上操作的目的是什么？以及之后有什么用？但最后几行代码似乎是返回了runtime的handle，以及将所有的线程先运行起来。先详细看第五和第六步具体操作
1. 这些`handle`各自都是什么？大概有什么作用？
2. `launch.launch`具体做了什么事情?

对于第一个问题:
1. 这里有几个handle，先看看每个handle具体指什么？  
2. 从`let (scheduler, handle, launch) = MultiThread::new(...)`可以看到第一个handle, 所以`MultiThread::new(...)` 获取`scheduler::Handle`, 第二个`handle`是`runtime`的`handle`。以下是`scheduler handle`结构体和`runtime handle`结构体。

```rust
// tokio/src/runtime/scheduler/multi_thread/handle.rs
/// Handle to the multi thread scheduler
pub(crate) struct Handle {
    /// Task spawner
    pub(super) shared: worker::Shared,

    /// Resource driver handles
    pub(crate) driver: driver::Handle,

    /// Blocking pool spawner
    pub(crate) blocking_spawner: blocking::Spawner,

    /// Current random number generator seed
    pub(crate) seed_generator: RngSeedGenerator,
}
```

```rust
  //tokio/src/runtime/handle.rs
  
  /// Handle to the runtime.
  ///
  /// The handle is internally reference-counted and can be freely cloned. A 
  /// handle can be obtained using the [`Runtime::handle`] method.
  ///
  /// [`Runtime::handle`]: crate::runtime::Runtime::handle()
  #[derive(Debug, Clone)]
  // When the `rt` feature is *not* enabled, this type is still defined, but not
  // included in the public API.
  pub struct Handle {
      pub(crate) inner: scheduler::Handle,
  }
```
对于第二个问题:  
`launch`是在`MultiThread::new(...)`中通过`worker::create`创建的
```rust
// tokio/src/runtime/scheduler/multi_thread/worker.rs
pub(super) fn create(
    size: usize,
    park: Parker,
    driver_handle: driver::Handle,
    blocking_spawner: blocking::Spawner,
    seed_generator: RngSeedGenerator,
    config: Config,
) -> (Arc<Handle>, Launch) {
    let mut cores = Vec::with_capacity(size);
    // remotes 是什么
    let mut remotes = Vec::with_capacity(size);
    let mut worker_metrics = Vec::with_capacity(size);

    // Create the local queues
    for _ in 0..size {
        let (steal, run_queue) = queue::local();

        let park = park.clone();
        let unpark = park.unpark();
        let metrics = WorkerMetrics::from_config(&config);

        // NOTE: 这里已经确定了每个worker在数组中的index
        cores.push(Box::new(Core {
            tick: 0,
            lifo_slot: None,
            lifo_enabled: !config.disable_lifo_slot,
            run_queue,
            is_searching: false,
            is_shutdown: false,
            park: Some(park),
            metrics: MetricsBatch::new(&metrics),
            rand: FastRand::new(config.seed_generator.next_seed()),
        }));

        // NOTE: 这里worker在remotes中的顺序和在cores中的顺序一样
        remotes.push(Remote { steal, unpark });
        worker_metrics.push(metrics);
    }

    let handle = Arc::new(Handle {
        shared: Shared {
            remotes: remotes.into_boxed_slice(),
            inject: Inject::new(),
            idle: Idle::new(size),
            owned: OwnedTasks::new(),
            shutdown_cores: Mutex::new(vec![]),
            config,
            scheduler_metrics: SchedulerMetrics::new(),
            worker_metrics: worker_metrics.into_boxed_slice(),
            _counters: Counters,
        },
        driver: driver_handle,
        blocking_spawner,
        seed_generator,
    });

    let mut launch = Launch(vec![]);

    for (index, core) in cores.drain(..).enumerate() {
        launch.0.push(Arc::new(Worker {
            handle: handle.clone(),
            index,
            core: AtomicCell::new(Some(core)),
        }));
    }

    (handle, launch)
}

/// Starts the workers
pub(crate) struct Launch(Vec<Arc<Worker>>);

impl Launch {
    pub(crate) fn launch(mut self) {
        for worker in self.0.drain(..) {
            runtime::spawn_blocking(move || run(worker));
        }
    }
}

/// A scheduler worker
pub(super) struct Worker {
    /// Reference to scheduler's handle
    handle: Arc<Handle>,

    /// Index holding this worker's remote state
    index: usize,

    /// Used to hand-off a worker's core to another thread.
    core: AtomicCell<Core>,
}

/// Core data
struct Core {
    /// Used to schedule bookkeeping tasks every so often.
    tick: u32,

    /// When a task is scheduled from a worker, it is stored in this slot. The
    /// worker will check this slot for a task **before** checking the run
    /// queue. This effectively results in the **last** scheduled task to be run
    /// next (LIFO). This is an optimization for improving locality which
    /// benefits message passing patterns and helps to reduce latency.
    lifo_slot: Option<Notified>,

    /// When `true`, locally scheduled tasks go to the LIFO slot. When `false`,
    /// they go to the back of the `run_queue`.
    lifo_enabled: bool,

    /// The worker-local run queue.
    run_queue: queue::Local<Arc<Handle>>,

    /// True if the worker is currently searching for more work. Searching
    /// involves attempting to steal from other workers.
    is_searching: bool,

    /// True if the scheduler is being shutdown
    is_shutdown: bool,

    /// Parker
    ///
    /// Stored in an `Option` as the parker is added / removed to make the
    /// borrow checker happy.
    park: Option<Parker>,

    /// Batching metrics so they can be submitted to RuntimeMetrics.
    metrics: MetricsBatch,

    /// Fast random number generator.
    rand: FastRand,
}
```
这里的worker的数量和cpu核数相同，launch中全是worker，worker中包含`scheduler handle`， index（之后调度排除自身时使用），根据`core` struct，`core`属于每个系统线程用于管理异步任务的执行，存储和任务调度、性能指标相关的数据的结构。
`launch.launch` 中就是让每个`worker` `run`起来，有了任务就执行，如果当前`worker`上没有任务了，就从其他`worker`上获取。具体细节后面再看

基于目前看过的代码，可以清楚这几个结构体之间的关系
- `scheduler handle`
- `scheduler handle`
- `core`
- `worker`
- `launch`

![tokio compenent structure](/images/tokio_component_structure.png)
