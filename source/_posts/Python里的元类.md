---
layout: post
title: "Python里的元类"
date: 2017-07-15 21:39
comments: true
tags:
	- Python
	- 技术
---
最近看设计模式的时候看到一种Python实现的单例模式有用到元类来进行实现。
因为工作上更多的是写业务逻辑，所以一直都没用过元类。
好奇之下仔细看了看Python的元类，在这里对元类做些介绍。
<!-- more -->
首先推荐大家去看看Stack Overflow上一篇很著名的[帖子](https://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python)，本篇博客的大部分内容都是我从这篇帖子里所收获到的。
嗯，下面是正文

## 什么是元类
元类就是用来创建类的类。
用代码表示就是：
```python
MyClass = MetaClass()
MyObject = MyClass()
```
用示意图表示就是：

![](/blogImg/metaclass1.png)

在这里我们先说说Python里的类与对象。在Python里，类是一组用来描述如何生成一个对象的代码段，而对象则是类的实例；类定义了集合中每个对象所共有的属性和方法。
示例如下：
```python
# 代码段1
>>> class ObjectCreator(object):
...       pass
...

# 代码段2
>>> my_object = ObjectCreator()
>>> print(my_object)
<__main__.ObjectCreator object at 0x8974f2c>
```
而在Python中，类也是对象。
执行代码段1时，Python会在内存中创建一个名为ObjectCreator的对象。
这个对象（类）自身拥有创建对象（类实例）的能力，所以它是一个类，但它本质上仍然是一个对象。
而对象是可以被动态创建的，所以类同样可以被动态创建。
所以，元类就是用来创建这些类的，即元类就是类的类。
事实上，在Python中，函数type实际上就是一个元类；type就是Python在背后用来创建所有类的元类。

## Python里类的创建
Python里有两种类的创建方式，第一种就是代码段1的那种，通过继承object来创建；第二种则是直接使用type来创建。
关于type与object的关系，如果要说就又是一篇文章了；不过这篇[博客](https://www.cnblogs.com/busui/p/7283137.html?utm_source=itdadao&utm_medium=referral)已经说得很棒了，感兴趣的同学可以看看。
简单地说，object是父子关系的顶端，所有的数据类型的父类都是它；type是类型实例关系的顶端，所有对象都是它的实例；
即object是type的实例，type是object的子类。
基于type的这种特性，type可以用来创建类，例如代码段1中的ObjectCreator()可以这么实现：
```python
MyClass = type('ObjectCreator', (), {})
```
其创建格式如下：
```python
type(类名, 父类的元组（针对继承的情况，可以为空），包含属性的字典（名称和值）)
```
这就是一种动态创建类的方式了

## 元类的使用
当想要控制类的创建，例如校验或修改类的属性的时候，就可以使用元类。
元类通过继承type实现
以一个很通用的例子来说明一下元类的使用：
```python
class ListMetaclass(type):
    """ 用元类实现给类添加属性 """
    def __new__(cls, name, bases, dct):
        dct['add'] = lambda self, value: self.append(value)
        return type.__new__(cls, name, bases, dct)

class MyList(list):
    __metaclass__ = ListMetaclass

>>> l = MyList()
>>> l.add(1)
>>> print(l)
[1]
```
可以在写一个类的时候为其添加**__metaclass__**属性，此时Python就会使用元类来创建类，例如上面的代码，
Python做了如下操作：A中有__metaclass__这个属性吗？如果有，Python会在内存中通过__metaclass__（此处为UpperAttrMetaclass）创建一个名字为A的类对象。
（注：如果A中没有找到__metaclass__而且A还有父类，那么Python会继续在父类中寻找__metaclass__属性，并尝试做和前面同样的操作。
如果Python在任何父类中都找不到__metaclass__，它就会在模块层次中去寻找__metaclass__，并尝试做同样的操作。
如果还是找不到__metaclass__,Python就会用内置的type来创建这个类对象。）
此时，新创建出来的MyList类就有了add方法

## 结语
以上就是Python元类的一个大概介绍了；事实上在平常的业务里也用不到这个，需要用到元类的地方使用[装饰器](http://yaoyaoblog.xyz/2017/06/30/Python%E8%A3%85%E9%A5%B0%E5%99%A8%E6%8E%A2%E8%AE%A8/)就可以解决。
不过好歹算是Python的一个黑科技，了解了解也是好的。



