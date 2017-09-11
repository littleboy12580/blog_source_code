---
layout: post
title: "Python源码分析之OrderedDict实现"
date: 2017-04-11 10:36
comments: true
tags:
	- Python
	- 技术
---

在写代码时用了一下Python里的OrderedDict类，因为好奇它的实现就去看了一下2.7里的源码，然后被它这种精简的思维给震撼到了。下面介绍一下OrderedDict的作用和具体实现思路

<!-- more -->

## OrderedDict作用

顾名思义，OrderedDict是有序的字典，它继承自dict，不过会记住每个key的插入顺序；注意OrderedDict的排序并不是通过key的值来排序的，而是通过key的插入顺序来排序的。具体事例如下：

```python
from collections import OrderedDict
ordered_dict = OrderedDict()
ordered_dict['A'] = 'a'
ordered_dict['B'] = 'b'
ordered_dict['1'] = '1'
for k, v in ordered_dict.items():
    print k, v
'''
最后的输出为
A a
B b
1 1
'''
```

## OrderedDict具体实现
### 思路
主要思路是用构建一个双端循环链表，该链表初始化时有一个哨兵节点（该节点永远不会被删除，这样可以简化算法）
链表起于该哨兵节点，终于该哨兵节点；链表中每个节点格式为 [PREV, NEXT, KEY]
使用 self.__map 来保存 Key 值和对应的双端循环链表中的节点的映射关系
使用继承的 dict 功能来保存 Key 和对应 Value 的映射关系

### 源码解析
```python
class OrderedDict(dict):
    def __init__(self, *args, **kwds):
        """
        此处虽然提供了kwds，但是并不建议在初始化的时候就传入多个命名参数
        因为这些参数的顺序是随意的，不是按照你传入的顺序来的
        """
        if not args:
            raise TypeError("descriptor '__init__' of 'OrderedDict' object "
                            "needs an argument")
        self = args[0]
        args = args[1:]
        if len(args) > 1:
            raise TypeError('expected at most 1 arguments, got %d' % len(args))
        try:
            self.__root
        except AttributeError:
            # 此处构建了哨兵节点
            self.__root = root = []
            # 构建双端循环链表
            root[:] = [root, root, None]
            self.__map = {}
        self.__update(*args, **kwds)
    def __setitem__(self, key, value, dict_setitem=dict.__setitem__):
        'od.__setitem__(i, y) <==> od[i]=y'
        if key not in self:
            # 向链表中插入新的 Key 节点
            # 每次都是在哨兵节点的前面插入一个新的节点
            # 每次插入的节点逻辑上都是最后一个节点
            # self.__map 中添加 Key 和 Key 节点的映射关系
            root = self.__root
            last = root[0]
            last[1] = root[0] = self.__map[key] = [last, root, key]
        # 使用继承的 dict 功能来保存 Key 和对应 Value 的映射关系
        return dict_setitem(self, key, value)
    def __delitem__(self, key, dict_delitem=dict.__delitem__):
        'od.__delitem__(y) <==> del od[y]'
        dict_delitem(self, key)
        # 先从映射关系中根据 Key 找到 Key 节点 , 然后再删除双端循环列表对应的 Key 节点
        link_prev, link_next, _ = self.__map.pop(key)
        link_prev[1] = link_next                        # update link_prev[NEXT]
        link_next[0] = link_prev                        # update link_next[PREV]
    def __iter__(self):
        'od.__iter__() <==> iter(od)'
        # 正向遍历，哨兵节点不参与遍历
        root = self.__root
        curr = root[1]                                  # start at the first node
        while curr is not root:
            yield curr[2]                               # yield the curr[KEY]
            curr = curr[1]                              # move to next node
    def __reversed__(self):
        'od.__reversed__() <==> reversed(od)'
        # 反向遍历，哨兵节点不参与遍历
        root = self.__root
        curr = root[0]                                  # start at the last node
        while curr is not root:
            yield curr[2]                               # yield the curr[KEY]
            curr = curr[0]                              # move to previous node
    def clear(self):
        'od.clear() -> None.  Remove all items from od.'
        root = self.__root
        # 由于引用本身不保存任何值，直接清除了root中原本的值
        root[:] = [root, root, None]
        self.__map.clear()
        dict.clear(self)
```
上面的就是OrderedDict的核心实现了，剩下的都是些高层API，感兴趣的话可以自行看看源码实现