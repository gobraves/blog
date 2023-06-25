---
title: "Rust_async_await"
date: 2023-06-26T01:00:48+08:00
---

## Rust Async Await

协程并不是抢占式任务模型，所以不是在任意时间点强制暂停正在运行的任务，而是让每个任务一直运行，直到它自愿放弃对cpu的控制，这样任务可以在合适时间点暂停。那么有三个问题，  
	1. 在rust中异步代码是怎么执行的？  
	2. 切换不同的任务时，被切换的任务的状态信息怎么保存？  
	3. 任务什么时候再次被执行?   
#### 1. 在rust中异步代码是怎么执行的？
在rust中写异步代码时就像写同步代码一样，但是rust为了实现这一机制，会将async code block 编译为一个状态机，使用下面这个例子来说明。  
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};


struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

async fn sleep_some_millis() {
    let array = [1,2,3];
    let num = &array[2];
  
    let when = Instant::now() + Duration::from_millis(num);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```
以上是正常编写的异步代码，但是在编译器编译后，会生成类似下面的代码
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum SleepSomeMillisStateMachine {
    State0,
    State1,
    Terminated,
}

struct SleepSomeMillisHandler {
	array: [i32: 3]
    num: &i32
    out_fut: Delay
    state: SleepSomeMillisStateMachine
}

impl Future for SleepSomeMillisHandler {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use SleepSomeMillisStateMachine::*;

        loop {
            match self.state {
                SleepSomeMillisStateMachine::State0 => {
                    let num = &array[2];
                    let when = Instant::now() +
                        Duration::from_millis(num);
                    self.out_fut = Delay { when };
                    self.state = SleepSomeMillisStateMachine::State1;
                }
                SleepSomeMillisStateMachine::State1 => {
                    match Pin::new(delay_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            self.state = SleepSomeMillisStateMachine::Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                SleepSomeMillisStateMachine::Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```
可以看到在实际执行时，每个task会这样被执行，如果遇到pending，则会暂时让出cpu。   
#### 2.任务状态信息怎么保存?
可以看到上面类似编译后的代码，会生成`struct SleepSomeMillisHandler`, 所以task的状态信息都会存在这个`struct`中。同时看下`std::future::Future`的定义
```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```
可以看到`Future trait`的`poll`方法，`self`类型是`Pin<&mut Self>`, `Pin<&mut Self>`和普通的`&mut Self`行为类似，但是会固定在内存的某个位置，这样做的原因是什么呢？，可以看到`struct SleepSomeMillisHandler`包含`array`和`num`，`num`是`array`最后一个元素的引用，这是自引用结构，但这样带来一个问题，如果`array`的地址变了，那么`num`还是指向原先的地址，这样`num`就变成了一个悬垂指针，因此对于自引用的结构，rust提供了`pin<T>`机制，可以固定T在内存中位置，使其不变

#### 3. 任务什么时候再次被执行
这时就要看`future` `trait` `poll`方法中的第二个参数`Context`。`Context`有一个`waker()`方法,可以返回一个`Waker`, `Waker`有`wake()`方法。这个方法的作用就是在任务为pending状态，但资源都准备好了，由`executor`或者`scheduler`周期性检查调用`wake()`方法，将任务标记为需要唤醒，等待后续执行
