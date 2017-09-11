---
layout: post
title: "Python装饰器探讨"
date: 2017-06-30 22:30
comments: true
tags:
	- Python
	- 技术
---

因为一直以来对Python的装饰器理解的都不是很透彻，所以最近详细看了看Python的装饰器部分。
结合装饰器设计模式以及Python本身实现的装饰器来说明一下装饰器这个概念。
<!-- more -->

## 设计模式里的装饰器模式

假设有这么一个对象，他实现了存钱的功能，现在要给他往里面加一个存钱时显示存钱数目的功能；
通常的做法是直接在该对象里添加相应的功能，或者可以从该对象里派生出一个子对象来实现这个功能；
还有一种做法是将该对象与另一个实现显示钱数的对象进行组合，称之为”对象组合”；

直接往对象里添加功能这种方式肯定是不可取的；使用继承则是会破坏对象的封装性，因此最好的办法还是用对象组合；
而装饰器模式就是一种对象组合，其本质就是动态组合，通过把复杂的功能简单化，分散化，
然后根据需要来进行动态组合，这就是装饰器模式。

具体地说，装饰器模式就是通过增加一个修饰对象包裹原来的对象，
包裹的方式一般是通过在将原来的对象作为修饰对象的构造函数的参数。
装饰对象实现新的功能，但是，在不需要用到新功能的地方，它可以直接调用原来的对象的方法（修饰对象必须和原来的对象有相同的接口）。

## Python中装饰器的使用
### 简单的装饰器
首先来看一个在讲装饰器时经常会用到的例子：
```python
def connect_db():
    print 'connect db'
def close_db():
    print 'close db'
def query_user():
    connect_db()
    print 'query the user'
    close_db()
query_user()            # connect db
                        # query the user
                        # close db

```
上面代码实现的是一个数据库的操作，首先需要连接数据库，然后执行查找用户的操作，最后关闭数据库。
因为连接数据库与关闭数据库是必需的，不同的只是中间对于数据库的处理；所以可以将中间的处理做成一个函数，将数据库的操作传到这个函数里面，代码如下：

```python
def connect_db():
    print 'connect db'

def close_db():
    print 'close db'

def query_user():
    print 'query the user'

def query_data(query):
    connect_db()
    query
    close_db()
query_data(query_user)

```
这就是一个简单的装饰器了，query_data是query_user的装饰，同样可以往query_data里传其他的数据库操作函数。
### 实际的装饰器
上面这么做的缺点是需要修改原有的query_user函数，换成query_data(query_user)；因此我们可以更进一步，把query_data的返回变成一个函数，代码如下：

```python
def connect_db():
    print 'connect db'

def close_db():
    print 'close db'
def query_user():
    print 'query some user'
def query_data(query):
    """ 定义装饰器，返回一个函数，对query进行wrapper包装 """
    def wrapper():
        connect_db()
        query()
        close_db()
    return wrapper
# 调用query_data来实际装饰query_user
query_user = query_data(query_user)
# 调用被装饰后的函数query_user
query_user()
```
此时就算是一个真正的装饰器了，可以直接调用query_user了
### 装饰器的语法糖
Python提供了对装饰器的语法糖，可以使用@调用装饰函数来对目标函数进行装饰，代码如下：
```python
def connect_db():
    print 'connect db'

def close_db():
    print 'close db'
def query_data(query):
    def wrapper():
        connect_db()
        query()
        close_db()
    return wrapper
# 使用 @ 调用装饰器进行装饰
@query_data
def query_user():
    print 'query some user'
query_user()
```
其中，后面的装饰代码等价于query_user = query_data(query_user)

## 带参数的装饰器
装饰器函数针对被装饰的函数进行装饰时，返回一个wrapper函数。这个函数可以等同于被装饰的函数，因此被装饰的函数参数可以通过wrapper来进行传递。代码如下：

```python
def connect_db():
    print 'connect db'

def close_db():
    print 'close db'
def query_data(query):
    def wrapper(count):
        connect_db()
        query(count)
        close_db()
    return wrapper
@query_data
def query_user(count):
    print 'query some user limit  {count}'.format(count=count)
query_user(count=100)       # connect db
                            # query some user limit  100
                            # close db

```

## 装饰器在flask里的应用
Flask的路由模块router使用的就是装饰器，和一般的装饰器不同的是，在router里装饰器是带参数的。下面是一个示例：
```python
@app.router('/user')
def user_page():
    return 'user page'
```
它的实现实际上是在原先的装饰器外面再包裹一层，也就是针对装饰器进行装饰。以上面的数据库操作装饰器为例：
```python
def connect_db():
    print 'connect db'

def close_db():
    print 'close db'

def router(url):
    print 'router invoke url', url
    def query_data(query):
        print 'query_data invoke url', url
        def wrapper(count):
            connect_db()
            query(count)
            close_db()
        return wrapper
    return query_data
@router('/user')          # 首先调用了router函数， 输出 router invoke url /user， 进行@装饰，输出 'query_data invoke url', url
def query_user(count):
    print 'query some user limit  {count}'.format(count=count)
query_user(count=100)   # connect db
                        # query some user limit  100
                        # close db
```
