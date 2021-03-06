---
layout: post
title: "Python引用机制"
date: 2017-06-09 19:32
comments: true
tags:
	- Python
	- 技术
---

最近逛V2看到有人提到了一个关于Python赋值的问题
看了一下问题相关的讨论，自己去测试了一下，结果也是很有趣的
然后系统的看了看Python的引用机制，整理了一下自己对Python引用的理解

<!-- more -->

## 问题

首先有这么一道题：

```python
a = [[0]*3]*5
a[0][0] = 1
```
请问此时的a的值是什么？
一开始我以为答案是[[1, 0, 0], [0, 0, 0], [0, 0, 0], [0, 0, 0], [0, 0, 0]]
然而事实上答案是[[1, 0, 0], [1, 0, 0], [1, 0, 0], [1, 0, 0], [1, 0, 0]]

再看下面这道题：


```python
a = [1, 2]
a = [a, a, None]
b = [1, 2]
b[:] = [b, b, None]
c = [1, 2]
c[0] = [c, c, None]
```
请问此时的a，b，c各是多少？
正确答案是a的值为[[1, 2], [1, 2], None]
b的值为[[…], […], None]
c的值为[[[…], […], None], 2]
为了理解这两道题的答案，就得先来了解一下Python的引用机制了

## Python引用机制
在Python中变量名和对象是分离的
因为Python中的传递都是“对象引用”的方式来传递
所以也可以说在Python中，引用和对象是分离的
### 示例一
```
In[171]: a = 1
In[172]: id(a)
Out[171]: 140526342944360
In[173]: a = 2
In[174]: id(a)
Out[173]: 140526342944336
```
第一个赋值语句中，1是存储在内存中的一个对象，通过赋值引用a指向了对象1
第二个赋值语句中，内存中尽力了另一个数字型对象2，通过赋值将引用a指向了2
此时，对象1不再有引用指向他，他就会被python的内存处理机制当做垃圾回收，释放内存
### 示例二
```
In[175]: a = 3
In[176]: id(a)
Out[175]: 140526342944312
In[177]: b = 3
In[178]: id(b)
Out[177]: 140526342944312
```
引用a与引用b指向了同一个对象
由于在Python中整数和短小字符都会被缓存（为了性能考虑，不然不停创建销毁太耗性能）
因此两个引用指向的对象是同一个
不过这种也只限于整数和短字符了，如果是列表这种就不是同一个id了
### 示例三
```
In[179]: a = 4
In[180]: b = a
In[181]: id(a)
Out[180]: 140526342944288
In[182]: id(b)
Out[181]: 140526342944288
In[183]: a = a + 2
In[184]: id(a)
Out[183]: 140526342944240
In[185]: id(b)
Out[184]: 140526342944288
In[186]: a
Out[185]: 6
In[187]: b
Out[186]: 4
```
一开始的时候创建一个整形对象4，让引用a指向这个对象
再让引用b指向引用a指向的那个对象，此时a与b都指向同一个对象
接下来的赋值语句使a的引用发生了改变，指向了新的整形对象6，而b依然指向4
因此可以看出即是多个引用指向同一个对象，如果一个引用值发生了变化，那么这个引用实际上指向了一个新引用，并不影响其他的引用
即各个引用互相独立，互不影响
### 示例四
```
In[188]: L1 = [1, 2 ,3]
In[189]: L2 = L1
In[190]: id(L1)
Out[189]: 4459145264
In[191]: id(L2)
Out[190]: 4459145264
In[192]: L1[0] = 5
In[193]: id(L1)
Out[192]: 4459145264
In[194]: id(L2)
Out[193]: 4459145264
In[195]: L1
Out[194]: [5, 2, 3]
In[196]: L2
Out[195]: [5, 2, 3]
```
与示例三类似，不同的是此时引用指向的对象是一个列表
而在python中列表是一个可变类型（通过引用其元素，改变对象自身的这种对象类型称为可变数据对象）
在该示例中，L1，L2的指向没有发生变化，依然指向那个列表
但列表本身包含了多个引用的对象（每个引用是一个元素，比如L1[0]，L1[1]…, 每个引用指向一个对象，比如1,2,3)
而L1[0] = 5这一赋值操作，并不是改变L1的指向，而是对L1[0], 也就是列表对象的一部份进行操作，所以所有指向该对象的引用都受到影响
之前的数字和字符串是不可变数据对象，不能改变对象本身，只能改变引用的指向

## 解答
### 问题一
看了上面的Python引用机制后，对于问题一的答案就很容易理解了。
a = [[0]3]5最后生成的是一个包含五个列表的列表[[0, 0, 0], [0, 0, 0], [0, 0, 0], [0, 0, 0], [0, 0, 0]]
这五个列表指向的都是同一个列表对象[0, 0, 0]，这个对象本身包含了对整数0这个对象的引用
而a[0][0] = 1这一操作改变了这个对象的第一个引用的对象，将其改成了1
但是该操作对另外两个引用不会造成影响（因为0是整数，不可变，各个引用此时相互独立）
所以最后的结果是[[1, 0, 0], [1, 0, 0], [1, 0, 0], [1, 0, 0], [1, 0, 0]]
这个地方还有一个坑就是[0]3的结果是[0, 0, 0]而不是[[0],[0],[0]]
因为这个操作其实是相加，[0]*3即是[0]+[0]+[0]结果为[0, 0, 0]
### 问题二
第一种情况下的引用a一开始指向了对象[1，2]
下面的赋值语句则是先新建了一个对象[a, a, None]
其中包含两个引用指向[1, 2]，所以改对象为[[1, 2], [1, 2], None]
然后将引用a指向了这个新对象
第二种情况下，a[:]事实上是新建了一个对象
a[:]是一个slice操作，其底层实现如下代码所示:
```python
static PyObject *
list_slice(PyListObject *a, Py_ssize_t ilow, Py_ssize_t ihigh)
{
    PyListObject *np;
    PyObject **src, **dest;
    Py_ssize_t i, len;
    if (ilow < 0)
        ilow = 0;
    else if (ilow > Py_SIZE(a))
        ilow = Py_SIZE(a);
    if (ihigh < ilow)
        ihigh = ilow;
    else if (ihigh > Py_SIZE(a))
        ihigh = Py_SIZE(a);
    len = ihigh - ilow;
    np = (PyListObject *) PyList_New(len);
    if (np == NULL)
        return NULL;
    src = a->ob_item + ilow;
    dest = np->ob_item;
    for (i = 0; i < len; i++) {
        PyObject *v = src[i];
        Py_INCREF(v);
        dest[i] = v;
    }
    return (PyObject *)np;
}
```
上述代码中的np = (PyListObject *) PyList_New(len)
这一句说明了slice操作会新分配内存出来，是一个新的对象了
此时对b[:]执行赋值操作，改变的不是b的引用，而是b列表里的每个部分指向的引用
于是此时b引用的对象列表里的引用指向的是[b, b, None]，造成循环
所以最后结果为[[…], […], None]（此处之所以是[…]是Python底层实现的一种默认输出，底层源码如下）
```python
1
2
3
4
5
6
7
8
9
10
11
static PyObject *
list_repr(PyListObject *v)
{
    Py_ssize_t i;
    PyObject *s, *temp;
    PyObject *pieces = NULL, *result = NULL;
    i = Py_ReprEnter((PyObject*)v);
    if (i != 0) {
        return i > 0 ? PyString_FromString("[...]") : NULL;
    }
...
```
具体实现可以自己去看源码，此处就不详解了
