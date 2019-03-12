---
type: post
title: 协程
date: 2018-12-20
tags: Python
---
# coroutine

- 协程是什么？
- 官方是如何实现协程
  - native coroutine
  - generator-based coroutine
- 其他协程库
  - greelet
- 目前还有哪些坑？

## 背景

### 处理io密集问题一般有三种模式：

- 多进程
- 多线程
- io复用+单线程回调

有了协程后，可以使用io复用+单线程搭配协程。

### 为什么不用回调呢？

步骤多或者嵌套深，捕捉异常或者阅读代码都会很困扰。而使用协程，让原来要使用异步+回调方式写的复杂代码,可以用看似同步的方式写出来。堆栈信息也很清晰，便于编写和维护

## 协程是什么

> 下文是根据官方的实现所定义，与greenlet, golang的goroutine不同

协程直接给它一个定义似乎不太容易，我会从它的运行方式及外沿来描述它。

在单线程中，过程A可被过程B打断，并保存此时过程A的上下文，且执行过程B,稍后过程B交回控制权，恢复过程A的上下文并从过程A的中断处继续执行。这是协程的运行方式。
换句话说，协程就像一个可被打断后继续执行的函数。一个线程中可以开多个协程，但某一时刻只有一个协程在运行。

## 如何实现协程

目前Python官方的协程实现方式有两种：

- generator-based coroutine
- native coroutine 

### generator-based coroutine

#### yield

要明白基于生成器的协程的工作方式，得先弄清生成器是怎么工作的，要说清楚生成器是怎么工作还得说明Python代码基于cpython是怎么执行的。

1. python代码会被编译成bytecode。
2. C程序中执行Python函数的函数叫`PyEval_EvalFrameEx`，当其遇到bytecode为`CALL_FUNCTION`，会在Python堆栈上新建一个栈帧

```python
>>> def bar():
...     pass
... 
>>> def foo():
...     bar()
...

>>> dis.dis(foo)
  2           0 LOAD_GLOBAL              0 (bar)
              2 CALL_FUNCTION            0
              4 POP_TOP
              6 LOAD_CONST               0 (None)
              8 RETURN_VALUE

```

而这个栈帧其实是c程序堆上的一个变量。里面存储了需要执行的字节码。如果这个是生成器函数，这个对象还会有`gi_frame`属性，`gi_frame`的`f_lasti`会记录当前执行到第几条字节码

举个例子  

```python
>>> import dis

>>> def a():
...     c = 10
...     print("haha") 
...     yield 1 
...     print("wawa") 
...     yield 10 
...     print("kaka")
...

>>> dis.dis(b)
  2           0 LOAD_CONST               1 (10)
              2 STORE_FAST               0 (c)

  3           4 LOAD_GLOBAL              0 (print)
              6 LOAD_CONST               2 ('haha')
              8 CALL_FUNCTION            1
             10 POP_TOP

  4          12 LOAD_CONST               3 (1)
             14 YIELD_VALUE
             16 POP_TOP

  5          18 LOAD_GLOBAL              0 (print)
             20 LOAD_CONST               4 ('wawa')
             22 CALL_FUNCTION            1
             24 POP_TOP

  6          26 LOAD_CONST               1 (10)
             28 YIELD_VALUE
             30 POP_TOP

  7          32 LOAD_GLOBAL              0 (print)
             34 LOAD_CONST               5 ('kaka')
             36 CALL_FUNCTION            1
             38 POP_TOP
             40 LOAD_CONST               0 (None)
             42 RETURN_VALUE

>>> b = a()
>>> b.send(None)
>>> b.gi_frame.f_lasti
haha
14
>>> b.send(None)
>>> b.gi_frame.f_lasti
wawa
28
>>> b.gi_frame.f_locals
{'c': 10}
```

上例中，a是一个用yield实现的生成器。并且生成器拥有一些属性，可以看到`f_lasti`记录了执行到了第几条指令, `f_locals`记录当前的变量，也就是说s生成器对象中完整的保存了当前环境的上下文。而且这个栈帧也就是c程序中一个普通的堆上的变量，可以在任何时候调起。这样就可以达到操作协程的需求了。

#### yield from

`yield from`有两个很明显的作用：

1. 简化 for 循环中的 yield 表达式

```python
>>> def gen():
...     for c in 'AB':
...         yield c
...     for i in range(1, 3):
...         yield i
...
>>> list(gen())
['A', 'B', 1, 2]

可以改写为
>>> def gen():
...     yield from 'AB'
...     yield from range(1, 3)
```

2. 打开双向通道,把最外层的调用方与最内层的子生成器连接起来

```python
def gen():
    while true:
        guess = yield "fail"
        if guess == 10:
            return "win"
```

### native coroutine

#### async/await 
未完待续
