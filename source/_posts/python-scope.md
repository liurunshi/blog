---
title: python的作用域
date: 2024-08-17 09:32:32
tags: python
categories: it
---

# 作用域，global和nonlocal

## 作用域

python中有四类作用域，分别是

1. 局部（`Local`）作用域）
2. 封闭（`Enclosing`）作用域
3. 全局（`Global`）作用域
4. 内置（`Built-in`）作用域

在作用域中按照名称查找对象（python中一切皆对象），也是按照LEGB的规则查找，那个作用域先找到就用那个作用域的对象。

LEGB规则如下：

`Local -> Enclosing -> Global -> Built-in`

一下例子很好解释这个规则的应用

```go
len = len([])       # (1)

def a():
    len = 1         # (2)

    def b():
        len = 2     # (3)
        print(len)  # (4)

    b()

a()
```

`len`原本是Python的一个内置函数，在以上代码中被重新定义了三次。

在(1)中，右边的`len([])`，就是内置的`len`函数。 而左边的`len =`，则是新定义的全局变量，值为`0`。

在(2)中，又重新定义了`len = 1`。 这里是函数`a`的本地作用域，而对嵌套的函数`b`来说，则是Enclosing作用域。

在(3)中，再次重新定义了`len = 2`。 这是函数`b`的本地作用域。

在(4)中，打印`len`时，遵循LEGB规则，会打印Local的`2`。 如果没有(3)，会打印Enclosing，也就是函数`a`里的`1`。 如果没有(2)，会打印Global里的`0`。 如果没有(1)，会打印`<built-in function len>`。

## global

global关键字主要用于对全局变量的写入

```go
i = 0

def a():
    **global** i
    i = 1
    print('local:', i)

a()
print('global:', i)

-------
local: 1
global: 1
```

## nonlocal

nonlocal关键字主要修改`Enclosing`变量问题

在多重嵌套中，`nonlocal`只会上溯一层； 而如果上一层没有，则会继续上溯。

如果一直上溯到最后一层（再往上就是全局变量了）都没有该变量，则`nonlocal`会报错。

```go
i = 0

def a():
    i = 1

    def b():
        **nonlocal** i
        i = 2

    b()
    print('a:', i)

a()

-------
a: 2
```