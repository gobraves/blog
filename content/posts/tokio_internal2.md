---
title: "Tokio源码阅读2"
date: 2023-06-12T14:31:52+08:00
---

## main thread execution

```rust
  #[track_caller]
      pub fn block_on<F: Future>(&self, future: F) -> F::Output {
          #[cfg(all(tokio_unstable, feature = "tracing"))]
          let future = crate::util::trace::task(
              future,
              "block_on",
              None,
              crate::runtime::task::Id::next().as_u64(),
          );
  
          let _enter = self.enter();
  
          match &self.scheduler {
              Scheduler::CurrentThread(exec) => exec.block_on(&self.handle.inner, future),
              #[cfg(all(feature = "rt-multi-thread", not(tokio_wasi)))]
              Scheduler::MultiThread(exec) => exec.block_on(&self.handle.inner, future),
          }
      }
```
执行`multithread::block_on`
```rust
/// Blocks the current thread waiting for the future to complete.
///
/// The future will execute on the current thread, but all spawned tasks
/// will be executed on the thread pool.
pub(crate) fn block_on<F>(&self, handle: &scheduler::Handle, future: F) -> F::Output
where
    F: Future,
{
    let mut enter = crate::runtime::context::enter_runtime(handle, true);
    enter
        .blocking
        .block_on(future)
        .expect("failed to park thread")
}
```
然后执行`BlockingRegionGuard`的`block_on`
```rust
pub(crate) fn block_on<F>(&mut self, f: F) -> Result<F::Output, AccessError>
    where
        F: std::future::Future,
    {
        use crate::runtime::park::CachedParkThread;

        let mut park = CachedParkThread::new();
        park.block_on(f)
    }
```
`CachedParkThread`的`block_on`
```rust
/// Blocks the current thread using a condition variable.
#[derive(Debug)]
pub(crate) struct CachedParkThread {
    _anchor: PhantomData<Rc<()>>,
}
```
```rust
// tokio-1.28.2/src/runtime/park.rs
pub(crate) fn block_on<F: Future>(&mut self, f: F) -> Result<F::Output, AccessError> {
	use std::task::Context;
    use std::task::Poll::Ready;

	// `get_unpark()` should not return a Result
	let waker = self.waker()?;
	let mut cx = Context::from_waker(&waker);

	pin!(f);

	loop {
		if let Ready(v) = crate::runtime::coop::budget(|| f.as_mut().poll(&mut cx)) {
		  return Ok(v);
		}

		// Wake any yielded tasks before parking in order to avoid
		// blocking.
		#[cfg(feature = "rt")]
		crate::runtime::context::with_defer(|defer| defer.wake());

		self.park();
	}
}
```
这里调用了`self.waker()?` ，创建了`CURRENT_PARKER` -> `UnparkThread` -> `waker` -> `context`
```rust
// tokio-1.28.2/src/runtime/park.rs
fn unpark(&self) -> Result<UnparkThread, AccessError> {
        self.with_current(|park_thread| park_thread.unpark())
}

pub(crate) fn park(&mut self) {
    self.with_current(|park_thread| park_thread.inner.park()).unwrap();
}

//...

/// Gets a reference to the `ParkThread` handle for this thread.
fn with_current<F, R>(&self, f: F) -> Result<R, AccessError>
where
    F: FnOnce(&ParkThread) -> R,
{
    CURRENT_PARKER.try_with(|inner| f(inner))
}

tokio_thread_local! {
    static CURRENT_PARKER: ParkThread = ParkThread::new();
}

#[derive(Debug)]
pub(crate) struct ParkThread {
    inner: Arc<Inner>,
}

/// Unblocks a thread that was blocked by `ParkThread`.
#[derive(Clone, Debug)]
pub(crate) struct UnparkThread {
    inner: Arc<Inner>,
}

#[derive(Debug)]
struct Inner {
    state: AtomicUsize,
    mutex: Mutex<()>,
    condvar: Condvar,
}

impl UnparkThread {
    pub(crate) fn into_waker(self) -> Waker {
        unsafe {
            let raw = unparker_to_raw_waker(self.inner);
            Waker::from_raw(raw)
        }
    }
}

unsafe fn unparker_to_raw_waker(unparker: Arc<Inner>) -> RawWaker {
    RawWaker::new(
        Inner::into_raw(unparker),
        &RawWakerVTable::new(clone, wake, wake_by_ref, drop_waker),
    )
}
```
继续上面的代码`let mut cx = Context::from_waker(&waker);`
```rust
/// core/src/task/wake.rs
pub struct Context<'a> {
    waker: &'a Waker,
    // Ensure we future-proof against variance changes by forcing
    // the lifetime to be invariant (argument-position lifetimes
    // are contravariant while return-position lifetimes are
    // covariant).
    _marker: PhantomData<fn(&'a ()) -> &'a ()>,
    // Ensure `Context` is `!Send` and `!Sync` in order to allow
    // for future `!Send` and / or `!Sync` fields.
    _marker2: PhantomData<*mut ()>,
}

impl<'a> Context<'a> {
    /// Create a new `Context` from a [`&Waker`](Waker).
    #[stable(feature = "futures_api", since = "1.36.0")]
    #[rustc_const_unstable(feature = "const_waker", issue = "102012")]
    #[must_use]
    #[inline]
    pub const fn from_waker(waker: &'a Waker) -> Self {
        Context { waker, _marker: PhantomData, _marker2: PhantomData }
    }

    /// Returns a reference to the [`Waker`] for the current task.
    #[stable(feature = "futures_api", since = "1.36.0")]
    #[rustc_const_unstable(feature = "const_waker", issue = "102012")]
    #[must_use]
    #[inline]
    pub const fn waker(&self) -> &'a Waker {
        &self.waker
    }
}
```
进入了一个loop，在其中执行了future poll，如果结束就直接返回，否则就继续后面的代码。
`crate::runtime::context::with_defer(|defer| defer.wake());`，这里需要执行CONTEXT defer中waker的wake方法，但是看起来只有调用了defer的defer方法才可以增加,但是只有yield_now方法中调用了。
`self.park()` 最终调用的是`Inner::park()`, 可以看到这里其实是会等待`unpark` `drop mutex.lock()`唤醒线程的，否则该线程就会一直`self.condvar.wait`
```rust
fn park(&self) {
    // If we were previously notified then we consume this notification and
    // return quickly.
    if self
        .state
        .compare_exchange(NOTIFIED, EMPTY, SeqCst, SeqCst)
        .is_ok()
    {
        return;
    }

    // Otherwise we need to coordinate going to sleep
    let mut m = self.mutex.lock();

    match self.state.compare_exchange(EMPTY, PARKED, SeqCst, SeqCst) {
        Ok(_) => {}
        Err(NOTIFIED) => {
            // We must read here, even though we know it will be `NOTIFIED`.
            // This is because `unpark` may have been called again since we read
            // `NOTIFIED` in the `compare_exchange` above. We must perform an
            // acquire operation that synchronizes with that `unpark` to observe
            // any writes it made before the call to unpark. To do that we must
            // read from the write it made to `state`.
            let old = self.state.swap(EMPTY, SeqCst);
            debug_assert_eq!(old, NOTIFIED, "park state changed unexpectedly");

            return;
        }
        Err(actual) => panic!("inconsistent park state; actual = {}", actual),
    }

    loop {
        m = self.condvar.wait(m).unwrap();

        if self
            .state
            .compare_exchange(NOTIFIED, EMPTY, SeqCst, SeqCst)
            .is_ok()
        {
            // got a notification
            return;
        }

        // spurious wakeup, go back to sleep
    }
}

fn unpark(&self) {
    // To ensure the unparked thread will observe any writes we made before
    // this call, we must perform a release operation that `park` can
    // synchronize with. To do that we must write `NOTIFIED` even if `state`
    // is already `NOTIFIED`. That is why this must be a swap rather than a
    // compare-and-swap that returns if it reads `NOTIFIED` on failure.
    match self.state.swap(NOTIFIED, SeqCst) {
        EMPTY => return,    // no one was waiting
        NOTIFIED => return, // already unparked
        PARKED => {}        // gotta go wake someone up
        _ => panic!("inconsistent state in unpark"),
    }

    // There is a period between when the parked thread sets `state` to
    // `PARKED` (or last checked `state` in the case of a spurious wake
    // up) and when it actually waits on `cvar`. If we were to notify
    // during this period it would be ignored and then when the parked
    // thread went to sleep it would never wake up. Fortunately, it has
    // `lock` locked at this stage so we can acquire `lock` to wait until
    // it is ready to receive the notification.
    //
    // Releasing `lock` before the call to `notify_one` means that when the
    // parked thread wakes it doesn't get woken only to have to wait for us
    // to release `lock`.
    drop(self.mutex.lock());

    self.condvar.notify_one()
}
```
总结一下就是循环中会先poll 传入的`future`，如果执行完就返回，否则就进入`park`，等待信号量
