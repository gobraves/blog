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

