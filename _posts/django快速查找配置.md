date: 2019-12-21
categories:
- web
tags:
- ctf
- django
- 小工具
cover: https://i.loli.net/2019/12/21/w4ctC1qXgyDO6vR.png
title: django快速查找配置
---
# django快速查找配置



## 脚本



在看p牛的picklecode wp的时候卡在secret_key查找那边

看到p牛一句很容易,人就傻掉了

![image209](https://i.loli.net/2019/12/21/8mnp5l7ODuPkBQo.png)

花了一天时间(踩了个坑),写了个快速查找secret_key的脚本

代码写的不太好,自己改改用吧////



```python
from django.http.response import HttpResponse, HttpResponseRedirect
from django.template import engines
from django.contrib.auth import login as auth_login, get_user_model, authenticate
from django.contrib.auth.views import LoginView, logout_then_login
from django.contrib.auth.decorators import login_required
from django.views import generic
from django import template

import django
from django import template
register = template.Library()

@register.filter
def get_dict(obj,way="",depth=0):
    if depth>11:
        return 
    objdir=dir(obj)
    r={"dict":objdir,"way":way}
    result=""
    
    for i in objdir:            
        try :
            if '_' == i[0]:
                continue
            if getattr(obj, '__module__', None)!=None and getattr(obj, '__module__', None).split('.')[0] == django.__name__:
                result+=get_dict(getattr(obj,i,None),way+"."+i,depth+1) 
        except TypeError:
            pass

    if "SECRET_KEY" in objdir or "settings" in objdir:
        print(way)
        return result+way+"\n"
    return result
```





这个是把所有符合条件的都输出,你可以通过修改递归深度和返回条件来加快,不然要等个几分钟



## 顺便说下踩到的坑



因为不知道dir()和`__dict__`的区别,一直以为`__dict__`==dir,然后就boomm

dir()和`__dict__`的区别

最简单的一句发\话是`__dict__`是dir的子集合



 https://stackoverflow.com/questions/13302917/whats-the-difference-between-dirself-and-self-dict/13302981#13302981 



所以以后查看对象的所有属性一定要用dir()