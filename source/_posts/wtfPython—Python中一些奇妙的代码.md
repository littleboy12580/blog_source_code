---
layout: post
title: "wtfPython—Python中一些奇妙的代码"
date: 2017-09-04 21:30
comments: true
tags:
	- Python
	- 技术
---
[wtfPython](https://github.com/satwikkansal/wtfpython#skipping-lines)是github上的一个项目，作者收集了一些奇妙的Python代码片段，这些代码的输出结果会和我们想象中的不太一样；
通过探寻产生这种结果的内部原因，可以让我们对Python里的一些细节有更广泛的认知。
<!-- more -->

### 1.字典键的隐式转换
```python
some_dict = {}
some_dict[5.5] = "Ruby"
some_dict[5.0] = "JavaScript"
some_dict[5] = "Python"
```
输出如下：
```
>>> some_dict
{5.0: "Python", 5.5: "Ruby"}
>>> some_dict[5.5]
"Ruby"
>>> some_dict[5.0]
"Python"
>>> some_dict[5]
"Python"
```
**原因**：
* Python的字典键的比较是通过哈希值来比较的
* 在Python里如果两个不可变对象的值相等，那他们的哈希也是一样的

因此此处hash(5) == hash(5.0)是True的，所以键被隐式的转换了


### 2.生成器执行时间的差异
```python
array = [1, 8, 15]
g = (x for x in array if array.count(x) > 0)
array = [2, 8, 22]
```
输出：
```
>>> print(list(g))
[8]
```
**原因**
* 在一个生成器表达式里，in的操作是在声明时求值的，而if是在运行期求值的
* 所以在运行期之前，array已经被重新分配成了[2,8,22],x的值也是2，8，22


### 3.在列表迭代式删除item
```python
list_1 = [1, 2, 3, 4]
list_2 = [1, 2, 3, 4]
list_3 = [1, 2, 3, 4]
list_4 = [1, 2, 3, 4]

for idx, item in enumerate(list_1):
    del item

for idx, item in enumerate(list_2):
    list_2.remove(item)

for idx, item in enumerate(list_3[:]):
    list_3.remove(item)

for idx, item in enumerate(list_4):
    list_4.pop(idx)
```
输出：
```
>>> list_1
[1, 2, 3, 4]
>>> list_2
[2, 4]
>>> list_3
[]
>>> list_4
[2, 4]
```
**原因**
* 其实只有list3才算是合格的写法，对一个正在迭代的对象进行修改并不是一个很好的选择，正确的做法应该是建立一份该对象的拷贝来进行迭代
* 对于list1，del item删除的只是item变量而不是变量指向的数据，对列表本身没有影响
* 对于list2和list4，因为列表的迭代是根据索引来的，第一次删掉了索引为0的1，剩下[2, 3, 4]，然后移除索引 1（此时为3），剩下了[2, 4]，此时只有2个元素，循环结束

### 4.else的不同处理
对于循环的else
```python
def does_exists_num(l, to_find):
      for num in l:
          if num == to_find:
              print("Exists!")
              break
      else:
          print("Does not exist")
```
输出：
```
>>> some_list = [1, 2, 3, 4, 5]
>>> does_exists_num(some_list, 4)
Exists!
>>> does_exists_num(some_list, -1)
Does not exist
```
对于try的else
```python
try:
    pass
except:
    print("Exception occurred!!!")
else:
    print("Try block executed successfully...")
```
输出：
```
Try block executed successfully...
```
**原因**
* 循环后的else只会在经过了所有迭代且没有出现break的时候才会执行
* 一个try模块后的else会在try里的代码成功执行完后去执行

### 5.python里的is
```python
>>> a = 256
>>> b = 256
>>> a is b
True

>>> a = 257
>>> b = 257
>>> a is b
False
```
**原因**
* is和==是不一样的；is判断的是两个对象是否是同一个对象，而==判断的是两个对象的值是否相等；即is是既要值相等又要引用一致
* 在Python中-5~256因为被经常使用所以被设计成固定存在的对象

### 6.循环里的局部变量泄露
#### 代码段1
```python
for x in range(7):
    if x == 6:
        print(x, ': for x inside loop')
print(x, ': x in global')
```
输出：
```
6 : for x inside loop
6 : x in global
```
#### 代码段2
```python
# This time let's initialize x first
x = -1
for x in range(7):
    if x == 6:
        print(x, ': for x inside loop')
print(x, ': x in global')
```
输出：
```
6 : for x inside loop
6 : x in global
```
#### 代码段3
```python
x = 1
print([x for x in range(5)])
print(x, ': x in global')
```
在Python2.x里的输出：
```
[0, 1, 2, 3, 4]
(4, ': x in global')
```
在Python3.x里的输出：
```
[0, 1, 2, 3, 4]
1 : x in global
```
**原因**
* 对于代码段1，在Python中，for循环可以使用包含他们的命名空间的变量，并将他们自己定义的循环变量保存下来；* 对于代码段2，如果我们在全局命名空间里显示定义for循环变量，则循环变量会重新绑定到现有变量上。
* 对于代码段3，在Python3.x中改变了对列表解析的语法形式；Python2.x中，列表解析的语法形式为：[... for var in item1, item2, ...]；而Python3.x的列表解析式为：[... for var in (item1, item2, ...)]，这种情况下不会发生循环变量的泄露


### 7.+和+=的区别
#### 代码段1
```python
a = [1, 2, 3, 4]
b = a
a = a + [5, 6, 7, 8]
```
输出：
```
>>> a
[1, 2, 3, 4, 5, 6, 7, 8]
>>> b
[1, 2, 3, 4]
```
#### 代码段2
```python
a = [1, 2, 3, 4]
b = a
a += [5, 6, 7, 8]
```
输出：
```
>>> a
[1, 2, 3, 4, 5, 6, 7, 8]
>>> b
[1, 2, 3, 4, 5, 6, 7, 8]
```
**原因**
* a = a + b的操作生成了一个新的对象并建立了一个新的引用
* a += b是在a这个列表上做extend操作

### 8.关于try---finally里的return
```python
def some_func():
    try:
        return 'from_try'
    finally:
        return 'from_finally'
```
输出：
```
>>> some_func()
'from_finally'
```
**原因**
* 在try…finally这种写法里面，finally中的return语句永远是最后一个执行
* 一个函数的return的值是由最后一个return语句来决定的

### 9.True=False
```python
True = False
if True == False:
    print("I've lost faith in truth!")
```
输出：
```
I've lost faith in truth!
```
**原因**
* 最开始的时候，Python是没有bool类型的（使用0表示false，使用非0值表示真），后来加上了True，False和bool类型；但是为了向后兼容性，True和False并没有被设置成常量，而只是一个内建变量，所以可以被赋值修改
* 在Python3当中，因为并没有向后兼容，所以不会有这种情况发生

### 10.一步操作，从有到无
```python
some_list = [1, 2, 3]
some_dict = {
  "key_1": 1,
  "key_2": 2,
  "key_3": 3
}

some_list = some_list.append(4)
some_dict = some_dict.update({"key_4": 4})
```
输出：
```
>>> print(some_list)
None
>>> print(some_dict)
None
```
**原因**
许多修改序列/映射对象的方法（例如list.append, dict.update, list.sort等等）都是直接修改对象并返回一个None；所以平常碰到这种直接修改的操作，应该避免直接赋值。

### 11.Python的for
```python
for i in range(4):
    print(i)
    i = 10
```
输出：
```
0
1
2
3
```
**原因**
* Python的for循环机制是每次迭代到下一项的时候都会解包并分配一次；即range(4)里的四个值在每次迭代的时候都会解包一次并赋值；所以i = 10对迭代没有影响