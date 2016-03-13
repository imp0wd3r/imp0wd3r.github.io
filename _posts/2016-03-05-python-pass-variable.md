---
layout: post
title: Python 函数参数传递
---

## Info
由于Python变量的特殊性，Python函数参数的传递并不是传值或者是传引用，而是一种特殊的方式，我们可以称之为: Call by sharing。

### Nametag
在Python中，变量其实是内存中对象的一个“标签”，我们可以称之为nametag。它并不像C或者其他语言那样在变量之间赋值时创建一个新的对象给新变量，而是让新变量也指向之前的对象，这里有个很形象的[例子](http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html#other-languages-have-variables)。我们也可以通过以下代码来说明：

```python
a = 1
b = a
print a is b # True
```

其实在CPython中，Python变量的实现为`PyObject*`,结合指针类型也就更容易理解“标签”的含义（Python对一些操作进行了重载，所以不要完全以指针的操作来看待变量操作）。

### 可变/不可变对象
Python中的对象可以分为可变和不可变两类，可变对象包括`list`、`dict`和`object`等类型的对象；不可变对象包括`str`、`number`和`tuple`。可变对象内置改变自身的方法，比如`list.append(1)`，而不可变对象并没有。

### 函数参数传递
结合以上两点，我们来看下面几个例子：

#### 0x00

```python
# 函数参数为不可变对象
a = 1
def fun(b):
    b = b + 1
fun(b)
print a # 1
```

在此例中，传参时首先执行`b = a`，即让“标签”`b`也指向`1`这个对象，而在函数中首先去`b`所指向的对象的值，再`+1`创建一个新的对象`2`，最后让`b`再指向`2`这个对象。也就是说，函数的作用实际上是创建了一个新的对象并让函数内部变量`b`指向这个新对象，而并没有改变`a`所指向的对象，所以最后`a`的值仍为`1`。

#### 0x01

```python
# 函数参数为可变对象
a = [1]
def fun(b):
    b = b + [2]
fun(a)
print a # [1]
```

原理同上，虽然参数是可变对象，但是函数内部逻辑依然为让函数内部变量指向所创建新对象，所以并没有改变`a`的值。

#### 0x02

```python
# 函数参数为可变对象
a = [1]
def fun(b):
    b.append(2)
fun(a)
print a # [1, 2]
```

在这个例子中，函数内部使用`b.append(2)`改变了`b`所指向的对象，也就是`a`所指向的对象`[1]`，所以函数执行后`a`的值发生了改变。

#### 0x03

```python
# 函数参数为可变对象
a = [1]
def fun(b):
    b += [2]
fun(a)
print a # [1, 2]
```

这个例子看似很奇怪，其实究其原因是因为Python对`+=`这个操作符进行了重载，若操作的对象是可变对象，`+=`实际上等同于`extend()`这类改变对象自身的函数。

总结以上几个例子，Python中函数参数传递实际上传递的是指向对象的“标签”，这种传参的方法就是我们下面要提到的“Call by sharing”。至于对形参的改变是否影响到实参，实际上是看函数中有没有对实参所指的对象本身进行改变。

### Call by sharing
[维基百科如是说](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing)，核心的意思就是说此类传参方法与传引用的区别在于：函数内部对形参的赋值在函数外是不可见的。

在传统C++中，引用所使用的是栈空间的数据，所以函数内部操作会反映到函数外部，而Python传递的是“标签”。但是因为内外标签指向的是同一对象，如果对象可变并直接改变了对象，那么这种“非赋值”的变化是会反映到函数外部的。

#### 参考链接

- [英文版：https://eev.ee/blog/2012/05/23/python-faq-passing/](https://eev.ee/blog/2012/05/23/python-faq-passing/)
- [中文版：http://python-china.org/t/738](http://python-china.org/t/738)
- [http://stackoverflow.com/questions/986006/how-do-i-pass-a-variable-by-reference](http://stackoverflow.com/questions/986006/how-do-i-pass-a-variable-by-reference)
