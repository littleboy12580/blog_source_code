---
layout: post
title: "通过request获取用户信息"
date: 2017-04-19 16:36
comments: true
tags:
	- Python
	- 技术
---

当我们上网访问某个网站时，有哪些信息会在这一行为中被暴露出来呢？仅以后端而言，因为我用flask比较多，所以就以flask做服务器的为例，来看看通过用户的一个请求我们后端可以收集到用户的哪些信息。

<!-- more -->

## 用户的终端设备信息

flask自带了request，当用户发送一个请求到后端服务器后，通过request可以获取到该请求的头部里的User Agent信息；通过该UA我们就可以获取到用户的终端设备类型、操作系统类型、以及访问该网站的浏览器类型；具体代码实现如下：

```python
# 根据User-Agent获取设备类型，系统类型与浏览器信号
def get_device_by_ua(ua_string):
    # 此处使用了第三方库ua-parser，直接可以对ua进行解析
    # ua_string通过request.headers['User-Agent']获取
    from ua_parser import user_agent_parser
    ua_dict = user_agent_parser.Parse(ua_string)
    user_agent_dict = ua_dict['user_agent']
    teminal_type = ua_dict['device']['family']
    if teminal_type.upper() == 'OTHER':
        teminal_type = 'PC'
    user_agent_version = (user_agent_dict['family'] + '/' +
                          user_agent_dict['major'] + '.' +
                          user_agent_dict['minor'] + '.' +
                          user_agent_dict['patch'])
    return dict(zip(('TERMINAL_TYPE', 'OS_TYPE', 'LOGIN_DEV_SOFTWARE'),
                    (teminal_type,ua_dict['os']['family'],
                    user_agent_version)))
```
ua-parser的基本思路就是对user-agent的那一串字符串做正则匹配，感兴趣的可以去看看github上的[开源库](https://github.com/ua-parser/uap-python)
对于user-agent里的那一串字符具体的含义可以参考这篇博客[浏览器野史 UserAgent列传](http://litten.me/2014/09/26/history-of-browser-useragent/)，写得十分有趣

## 用户的IP与地域信息
通过request我们还可以获取到用户的ip地址与端口号；而获取到ip后我们就可以根据ip来判断用户当时所处的地理位置；用户的ip地址与端口号获取代码实现如下：
```python
from flask import request
user_ip = request.remote_addr
user_port = request.environ.get('REMOTE_PORT')
```
在获取到ip后，可以通过ip库来找到该ip对应的地域信息；ip库现在网络上比较权威的有淘宝的ip库，qq的纯甄库，以及百度的ip库；
不过这几个库都要钱，因此我在github上找了一个开源ip库[ip region库](https://github.com/lionsoul2014/ip2region)；
这个库是一个范围式的ip库，取了几个例子测试了一下，发现准确率还是很不错的；下面是通过这个ip库来做ip与地址转换的代码实现：

```python
# 字符串式的ip地址不好比较大小，因此做个正相关转换
def get_transfer_ip(ip_addr):
    ip_set = [int(i) for i in ip_addr.split('.')]
    ip_number = (ip_set[0] << 24) + (ip_set[1] << 16) + (ip_set[2] << 8) + ip_set[3]
    return ip_number
# 使用二分查找法来查找IP对应地域信息
def binary_scope_search(target_ip, ip_dict):
    ip_scope_list = sorted(ip_dict.keys())
    start = 0
    end = len(ip_scope_list)
    # 由于这个ip库的IP地址是范围式的，所以对应二分查找算法也需要做些修改
    while start < end - 1:
        mid = (start + end) / 2
        if ip_scope_list[mid] <= target_ip:
            start = mid
        else:
            end = mid
    return ip_dict[ip_scope_list[start]]
# 通过获取的ip找到相应地域信息
def get_location_by_ip(ip):
    ip_scope_list = []
    ip_info_dict = {}
    with open('_ip.txt','r') as f:
        for line in f.readlines():
            # 该ip库使用的utf-8编码
            line = line.decode('utf-8')
            info_list = line.split('|')
            start_ip = get_transfer_ip(info_list[0])
            ip_info_dict[start_ip] = info_list
            ip_scope_list.append(start_ip)
    ip_scope_list = sorted(list(set(ip_scope_list)))
    ip_start = binary_search_scope(ip, ip_scope_list)
    return {'loaction': '|'.join(ip_info_dict[ip_start])}
```