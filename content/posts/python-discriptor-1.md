---
title: python描述器（1）
date: 2020-04-06 15:55:55
tags: ["python"]
---


先看一段代码，这段代码来自《流畅的python》

```python
class Quantity:

    def __init__(self, storage_name):
        self.storage_name =storage_name

    def __set__(self, instance, value):
        if value > 0:
            # setattr(instance, self.storage_name, value)
            instance.__dict__[self.storage_name] = value
        else:
            raise ValueError('value must be > 0')

    def __get__(self, instance, owner):
        return getattr(instance, self.storage_name)


class LineItem:
    weight1 = Quantity('weight')
    price1 = Quantity('price')

    def __init__(self, description, weight, price):
        self.description = description
        self.weight1 = weight
        self.price1 = price

    def subtotal(self):
        return self.weight1 * self.price1
```

如果用`setattr(instance, self.storage_name, value)`替换 `instance.__dict__[self.storage_name] = value`会出现什么结果呢？答案是无限循环。那为什么呢？

根据[描述器协议](https://docs.python.org/zh-cn/3/howto/descriptor.html)

```python
descr.__get__(self, obj, type=None) -> value

descr.__set__(self, obj, value) -> None

descr.__delete__(self, obj) -> None

以上就是全部。定义这些方法中的任何一个的对象被视为描述器，并在被作为属性时覆盖其默认行为。

如果一个对象定义了 __set__() 或 __delete__()，则它会被视为数据描述器。 仅定义了 __get__() 的描述器称为非数据描述器（它们通常被用于方法，但也可以有其他用途）。

数据和非数据描述器的不同之处在于，如何计算实例字典中条目的替代值。如果实例的字典具有与数据描述器同名的条目，则数据描述器优先。如果实例的字典具有与非数据描述器同名的条目，则该字典条目优先。

为了使数据描述器成为只读的，应该同时定义 __get__() 和 __set__() ，并在 __set__() 中引发 AttributeError 。用引发异常的占位符定义 __set__() 方法使其成为数据描述器。
```

可以知道Quantity是一个数据描述符，LineItem实例有和数据描述符同名的属性，因此按描述器协议会优先计算描述器字典中同名属性。所以setattr 所作的赋值操作其实是给数据描述符对应的属性赋值，又会调用__set__方法，所以才导致无限循环
