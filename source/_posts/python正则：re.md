---
title: Python正则使用
date: 2018-07-27 
tags: 
- 正则表达式
categories:
- Python
---

<!-- toc -->
#### 导入正则：re
```
import re
```

#### match
```
#match匹配的过程是从左第一个字符向右逐一匹配的，当匹配的过程出现不符合字符时，即停止
匹配过程，相当于匹配表达式中设置了"^"

re.match(pattern,str)

In [34]: re.match(r"aa(\d+?)ddd","aa1234ddd").group(1)
Out[34]: '1234'
```
<!--more--> 
#### search
```
#search搜索匹配整个字符串中，符合表达式的字符串，并不要求第一个字符必须匹配，除非表达
式中设置了"^"；
#search只能匹配到第一个字符串

re.search(pattern,str)

In [66]: s = """<img data-original="https://rpic.douyucdn.cn/appCovers/2016/11/13/1213973_201611131917_small.jpg" src="https://rpic.douyucdn.cn/appCovers/2016/11/13/1213973_201611131917_small.jpg" style="display: inline;">"""

In [71]: r = re.search(r"https://.+?jpg",s)

In [72]: r.group()
Out[72]: 'https://rpic.douyucdn.cn/appCovers/2016/11/13/1213973_201611131917_small.jpg'
```

#### findall
```
#findall类似于search，但是能够匹配出多个符合要求的结果的列表
re.findall(rex,str)

In [75]: re.findall(r"https://.+?jpg",s)
Out[75]: 
['https://rpic.douyucdn.cn/appCovers/2016/11/13/1213973_201611131917_small.jpg',
 'https://rpic.douyucdn.cn/appCovers/2016/11/13/1213973_201611131917_small.jpg']
```

#### sub
```
#sub能够将查找到的匹配的字符串进行替换为指定的字符串
#参数replace可以是字符串，用来指定替换成的字符串，也可以是一个回调函数，该函数在每次匹
#配到结果符合要求的字符串时，都会被调用，并且将该匹配结果作为参数，我们需要提供一个
#返回值作为替换的值。
re.sub(rex,replace,str)

比如：
    //对匹配到的结果进行加1操作
    def replace(result):
    	r = int(result.group()) + 1
    	return str(r)
    
    re.sub(r"\d+", replace, "python=1000, php=0")
    结果：python=1001, php=1

    提取网站的host
    In [35]: s = "http://www.baidu.com/a.jpg"
    In [40]: re.sub("(http://.+?/).*",lambda x:x.group(1),s)
    
    原理：匹配整个网站，然后使用分组host替换整个字符串；

```

#### split
```
#split按照给定规则将字符串分割为列表
re.split(r":|,|-","java:php,python-c")
out:['java', 'php', 'python', 'c']
```
