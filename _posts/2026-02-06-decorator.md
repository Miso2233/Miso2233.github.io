---
layout: post
title: "Python函数装饰器语法"
subtitle: "关于高阶函数"
date: 2026-02-06
author: "Miso"
header-img: "img/post-bg-0.jpg"
tags: ["Python", "编程语言", "高阶函数", "装饰器"]
---

### 0. 前言 关于高阶函数

---

高阶函数的定义：能够接收其他函数作为参数，或者将函数作为返回值的函数。它是“函数式编程”的灵魂组成部分。

```python
def add(a: int,b: int) -> int:
    return a + b

def sub(a: int,b: int) -> int:
    return a - b

def do_operation(a: int, b: int, operation: Callable[[int, int], int]) -> int:
    return operation(a, b)
    
do_operation(1,2,add) # 返回 3
```

在这里，我们定义了两个函数`add`和`sub`。一旦经由`def`定义，`add`和`sub`两个函数名便成为了“可调用对象”，类型提示为`: Callable[[int, int], int]`，是接受两个整数、返回整数的函数。

请注意，这里我们讨论可调用对象时，没有在函数名后面加上括号填入参数。因为`add(1,2)`是`3: int`。在一个可调用对象后面加上括号传入参数的行为便是大名鼎鼎的“调用”，此时整个表达式等于函数的返回值。

### 1. 举个栗子：计时装饰器

---

```python
def f_interface(n: int) -> int:
    if n <= 2:
        return 1
    return f_recursion(n)

def f_recursion(n: int) -> int:
    if n <= 2:
        return 1
    return f_recursion(n - 1) + f_recursion(n - 2)
```

这是一套运用递归方法求出斐波那契数列特定项的函数，其时间复杂度为$O(2^n)$

当n取30到40时，函数运行便需要较长的时间。我们希望有一种方法来计算函数运行的用时。

#### 朴素的方法

用朴素的方法实现，我们需要修改`f_interface`，如下：

```python
def f_interface_v1(n: int) -> int:
    start_time = time.time()
    if n <= 2:
        result = 1
    else:
        result = f_recursion(n)
    end_time = time.time()
    print(f"Function f_interface_v1 took {end_time - start_time} seconds to run")
    return result
```

这个改动……太过伤筋动骨了。

为了实现统计时间的功能，我们统一了函数出口，放弃了卫语句。我们还要环绕着核心业务写三行代码，把计时功能和原有业务紧紧耦合在一起，不便于观看，也不便于删除。

#### 让高阶函数上场

与其在函数内部进行修改，不如尝试在函数外面修改。怎么做呢？

我们设想一下下面这样的函数：

```python
def add_time_logger(func: Callable[[int], int])
	do_what?
	func_with_time_logger: Callable[[int], int] = what?
	return func_with_time_logger
	
f_interface = add_time_logger(f_interface)
```

这样，我们用带有时间统计的**新函数覆盖了旧函数名**。函数签名不变，意味着任何使用新旧函数的地方都不需要有任何修改。

现在，我们只需要专注实现这个函数。

do_what?

#### 建立嵌套的def

显然，我们要在`add_time_logger`内定义一个新的函数，这个函数接受单个`int`参数，返回`int`. 此时我们需要嵌套的`def`。

```python
def add_time_logger(func: Callable[[int], int]):
    def func_with_time_logger(arg: int):
        do_what?
        return what?
    return func_with_time_logger
```

现在我们面临两个问题。新函数要做什么？返回什么？

#### 新函数的内部逻辑

新旧函数的参数列表和返回值含义是完全一致的，因此，必须调用旧函数来获取返回值。

旧函数便是最外层`def`形参中的`func`

```python
def add_time_logger(func: Callable[[int], int]):
    def func_with_time_logger(arg: int):
        result = func(arg)
        return result
    return func_with_time_logger
```

此时，时间统计的逻辑可以环绕形参的调用`func(arg)`展开

```python
def add_time_logger(func: Callable[[int], int]):
    def func_with_time_logger(arg: int):
        start = time.time()
        result = func(arg)
        end = time.time()
        print(f"Function {func.__name__} took {end - start} seconds to run")
        return result
    return func_with_time_logger
```

其中`func.__name__`用以获取函数名，使打印更清晰。

至此，高阶函数`add_time_logger`构建完成。

#### 语法糖

直接写`f_interface = add_time_logger(f_interface)`两次书写了原函数名，显得非常臃肿。

并且，允许函数名覆盖以这种形式写在原`def`下游的任何位置是非常危险的。两个函数的签名和返回值完全一致，如果找不到这一行赋值，我们很难发现函数是否被覆盖。

两个函数少有的区别在于：`__doc__`会丢失。（这可不是什么好消息）

为了解决这一问题，Python提供了一种好用的语法糖。我们只需：

```python
@add_time_logger
def f_interface(n: int) -> int:
    if n <= 2:
        return 1
    return f_recursion(n)
```

直接将@加上高阶函数名（我们称之为装饰器）置于需修饰的函数上方，这一操作等价于`f_interface = add_time_logger(f_interface)`，但可保留`__doc__`

恭喜我们学会了函数装饰器。语法如此。

### 2. 上点硬菜：记忆化搜索装饰器

---

众所周知，$O(2^n)$时间复杂度在绝大多数情况下都是不可接受的。我们尝试引入记忆化搜索技术，将时间复杂度压缩至$O(n)$

我们需要用一个哈希表来存储已有的计算结果，其存取操作和包含判断均为$O(1)$

装饰器构造如下：

```python
def memorize_result(func: Callable[[int], int]):
    result_dict = {1:1, 2:1}
    def wrapper(n: int) -> int:
        if n not in result_dict:
            result = func(n)
            result_dict[n] = result
        else:
            result = result_dict[n]
        return result
    return wrapper
```

用`wrapper`指代新函数为业界惯例。新函数只在没有搜索到结果的情况下才会执行旧函数，同时存储结果。

运用时，只需：

```python
@memorize_result
def f_recursion(n: int) -> int:
    if n <= 2:
        return 1
    return f_recursion(n - 1) + f_recursion(n - 2)
```

即可对这个函数施加记忆化搜索功能，显著提高其计算速度。

### 3. 拓展：将装饰器应用到更多函数

---

刚才提到的装饰器只能应用于一个`: Callable[[int], int]`显得有些不足。尝试将其应用于所有类型组合的函数：

```python
def add_time_logger(func: Callable[..., Any]):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"Function {func.__name__} took {end - start} seconds to run")
        return result
    return wrapper
```

我们引入了`*args, **kwargs`来接受任意参数，同时使用`: Callable[..., Any]`注解来适配任何返回值的函数。