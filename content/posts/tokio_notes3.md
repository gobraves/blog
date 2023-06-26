---
title: "Tokio源码阅读3"
date: 2023-06-26T23:02:21+08:00
draft: true
---
首先看`let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();` `bind`->`to_socket_addrs`-> `impl sealed::ToSocketAddrsPriv for str` 中的`to_socket_addrs()`。`MaybeReady`实现了`Future`，所以`await`时实际执行的是`MaybeReady`的`poll()`。
不过这里的重点是创建了一个`TcpListener`，看下`TcpListener`创建过程
```rust
fn bind_addr(addr: SocketAddr) -> io::Result<TcpListener> {
    let listener = mio::net::TcpListener::bind(addr)?;
    TcpListener::new(listener)
}

pub(crate) fn new(listener: mio::net::TcpListener) -> io::Result<TcpListener> {
    let io = PollEvented::new(listener)?;
    Ok(TcpListener { io })
}

pub(crate) fn new(io: E) -> io::Result<Self> {
    PollEvented::new_with_interest(io, Interest::READABLE | Interest::WRITABLE)
}

pub(crate) fn new_with_interest(io: E, interest: Interest) -> io::Result<Self> {
    Self::new_with_interest_and_handle(io, interest, scheduler::Handle::current())
}

pub(crate) fn new_with_interest_and_handle(
  	mut io: E,
  	interest: Interest,
  	handle: scheduler::Handle,
) -> io::Result<Self> {
    let registration = Registration::new_with_interest_and_handle(&mut io, interest, handle)?;
    Ok(Self {
      io: Some(io),
      registration,
    })
}

pub(crate) fn new_with_interest_and_handle(
    io: &mut impl Source,
    interest: Interest,
    handle: scheduler::Handle,
) -> io::Result<Registration> {
    let shared = handle.driver().io().add_source(io, interest)?;

    Ok(Registration { handle, shared })
}

pub(super) fn add_source(
    &self,
    source: &mut impl mio::event::Source,
    interest: Interest,
) -> io::Result<slab::Ref<ScheduledIo>> {
    let (address, shared) = self.allocate()?;

    let token = GENERATION.pack(shared.generation(), ADDRESS.pack(address.as_usize(), 0));

    self.registry
    .register(source, mio::Token(token), interest.to_mio())?;

    self.metrics.incr_fd_count();

    Ok(shared)
}
```

这里`self.registry.register(...)`, `source`代表事件源，`token`是原来标记后续返回的事件来自哪个事件源，interest代表要监听的行为。同时这里可以看到`ScheduledIo`被返回，并成了`TcpListener`一部分。

`let (mut socket, _) = listener.accept().await.unwrap();`开始等待请求，由`accept`可以看到
```rust
// impl TcpListener
pub async fn accept(&self) -> io::Result<(TcpStream, SocketAddr)> {
    let (mio, addr) = self
    .io
    .registration()
    .async_io(Interest::READABLE, || self.io.accept())
    .await?;

    let stream = TcpStream::new(mio)?;
    Ok((stream, addr))
}

// impl Registration
pub(crate) async fn async_io<R>(&self, interest: Interest, mut f: impl FnMut() -> io::Result<R>) -> io::Result<R> {
    loop {
        let event = self.readiness(interest).await?;

        match f() {
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
              self.clear_readiness(event);
            }
            x => return x,
        }
    }
}

pub(crate) async fn readiness(&self, interest: Interest) -> io::Result<ReadyEvent> {
    let ev = self.shared.readiness(interest).await;

    if ev.is_shutdown {
      	return Err(gone())
    }

    Ok(ev)
}

// imple ScheduledIo
pub(crate) async fn readiness(&self, interest: Interest) -> ReadyEvent {
  	self.readiness_fut(interest).await
}

// This is in a separate function so that the borrow checker doesn't think
// we are borrowing the `UnsafeCell` possibly over await boundaries.
//
// Go figure.
fn readiness_fut(&self, interest: Interest) -> Readiness<'_> {
    Readiness {
        scheduled_io: self,
        state: State::Init,
        waiter: UnsafeCell::new(Waiter {
            pointers: linked_list::Pointers::new(),
            waker: None,
            is_ready: false,
            interest,
            _p: PhantomPinned,
        }),
    }
}

struct Readiness<'a> {
    scheduled_io: &'a ScheduledIo,

    state: State,

    /// Entry in the waiter `LinkedList`.
    waiter: UnsafeCell<Waiter>,
}

impl Future for Readiness<'_> {
    type Output = ReadyEvent;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        use std::sync::atomic::Ordering::SeqCst;

        let (scheduled_io, state, waiter) = unsafe {
            let me = self.get_unchecked_mut();
            (&me.scheduled_io, &mut me.state, &me.waiter)
        };

        loop {
            match *state {
              	State::Init => {
                    // Optimistically check existing readiness
                    let curr = scheduled_io.readiness.load(SeqCst);
                    let ready = Ready::from_usize(READINESS.unpack(curr));
                    let is_shutdown = SHUTDOWN.unpack(curr) != 0;

                    // Safety: `waiter.interest` never changes
                    let interest = unsafe { (*waiter.get()).interest };
                    let ready = ready.intersection(interest);

                    if !ready.is_empty() || is_shutdown {
                        // Currently ready!
                        let tick = TICK.unpack(curr) as u8;
                        *state = State::Done;
                        return Poll::Ready(ReadyEvent { tick, ready, is_shutdown });
                    }

                    // Wasn't ready, take the lock (and check again while locked).
                    let mut waiters = scheduled_io.waiters.lock();

                    let curr = scheduled_io.readiness.load(SeqCst);
                    let mut ready = Ready::from_usize(READINESS.unpack(curr));
                    let is_shutdown = SHUTDOWN.unpack(curr) != 0;

                    if is_shutdown {
                          ready = Ready::ALL;
                    }

                    let ready = ready.intersection(interest);

                    if !ready.is_empty() || is_shutdown {
                        // Currently ready!
                        let tick = TICK.unpack(curr) as u8;
                        *state = State::Done;
                        return Poll::Ready(ReadyEvent { tick, ready, is_shutdown });
                    }

                    // Not ready even after locked, insert into list...

                    // Safety: called while locked
                    unsafe {
                        (*waiter.get()).waker = Some(cx.waker().clone());
                    }

                    // Insert the waiter into the linked list
                    //
                    // safety: pointers from `UnsafeCell` are never null.
                    waiters
                    .list
                    .push_front(unsafe { NonNull::new_unchecked(waiter.get()) });
                    *state = State::Waiting;
              	}
              	State::Waiting => {
                    // Currently in the "Waiting" state, implying the caller has
                    // a waiter stored in the waiter list (guarded by
                    // `notify.waiters`). In order to access the waker fields,
                    // we must hold the lock.

                    let waiters = scheduled_io.waiters.lock();

                    // Safety: called while locked
                    let w = unsafe { &mut *waiter.get() };

                    if w.is_ready {
                        // Our waker has been notified.
                        *state = State::Done;
                    } else {
                        // Update the waker, if necessary.
                        if !w.waker.as_ref().unwrap().will_wake(cx.waker()) {
                          w.waker = Some(cx.waker().clone());
                        }

                        return Poll::Pending;
                    }

                    // Explicit drop of the lock to indicate the scope that the
                    // lock is held. Because holding the lock is required to
                    // ensure safe access to fields not held within the lock, it
                    // is helpful to visualize the scope of the critical
                    // section.
                    drop(waiters);
              	}
                State::Done => {
                    // Safety: State::Done means it is no longer shared
                    let w = unsafe { &mut *waiter.get() };

                    let curr = scheduled_io.readiness.load(Acquire);
                    let is_shutdown = SHUTDOWN.unpack(curr) != 0;

                    // The returned tick might be newer than the event
                    // which notified our waker. This is ok because the future
                    // still didn't return `Poll::Ready`.
                    let tick = TICK.unpack(curr) as u8;

                    // The readiness state could have been cleared in the meantime,
                    // but we allow the returned ready set to be empty.
                    let curr_ready = Ready::from_usize(READINESS.unpack(curr));
                    let ready = curr_ready.intersection(w.interest);

                    return Poll::Ready(ReadyEvent {
                        tick,
                        ready,
                        is_shutdown,
                    });
              	}
            }
        }
    }
}
```
如果此时被`pending`，将会返回到`if let Ready(v) = crate::runtime::coop::budget(|| f.as_mut().poll(&mut cx))`这里，此时主线程会进入休眠，等待被唤醒
