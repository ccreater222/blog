categories:
- web

tags:

- web
- ctf
title: jwt攻击方式
---
# jwt

## 文章

 https://xz.aliyun.com/t/6776

  https://www.anquanke.com/post/id/145540 

## 关于jwt

JWT的全称是Json Web Token。它遵循JSON格式，将用户信息加密到token里，服务器不保存任何用户信息，只保存密钥信息，通过使用特定加密算法验证token，通过token验证用户身份。基于token的身份验证可以替代传统的cookie+session身份验证方法。

jwt由三个部分组成：`header`.`payload`.`signature`

### header部分

header部分最常用的两个字段是`alg`和`typ`，`alg`指定了token加密使用的算法（最常用的为*HMAC*和*RSA*算法），typ`声明类型为JWT

header通常会长这个样子：

```
{
        "alg" : "HS256",
        "typ" : "jwt"
}
```

### payload部分

payload则为用户数据以及一些元数据有关的声明，用以声明权限，举个例子，一次登录的过程可能会传递以下数据

```
{
        "user_role" : "finn",    //当前登录用户
    "iss": "admin",          //该JWT的签发者
    "iat": 1573440582,        //签发时间
    "exp": 1573940267,        //过期时间
    "nbf": 1573440582,         //该时间之前不接收处理该Token
    "domain": "example.com",   //面向的用户
    "jti": "dff4214121e83057655e10bd9751d657"   //Token唯一标识
}
```

### signature部分

signature的功能是保护token完整性。

生成方法为将header和payload两个部分联结起来，然后通过header部分指定的算法，计算出签名。

抽象成公式就是

```
signature = HMAC-SHA256(base64urlEncode(header) + '.' + base64urlEncode(payload), secret_key)
```

值得注意的是，编码header和payload时使用的编码方式为`base64urlencode`，`base64url`编码是`base64`的修改版，为了方便在网络中传输使用了不同的编码表，它不会在末尾填充"="号，并将标准Base64中的"+"和"/"分别改成了"-"和"-"。

### 完整token生成

一个完整的jwt格式为(`header`.`payload`.`signature`)，其中header、payload使用base64url编码，signature通过指定算法生成。

python的`Pyjwt`使用示例如下

```
import jwt

encoded_jwt = jwt.encode({'user_name': 'admin'}, 'key', algorithm='HS256')
print(encoded_jwt)
print(jwt.decode(encoded_jwt, 'key', algorithms=['HS256']))
```

生成的token为

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`.`eyJ1c2VyX25hbWUiOiJhZG1pbiJ9.oL5szC7mFoJ_7FI9UVMcKfmisqr6Qlo1dusps5wOUlo
```



## 攻击方式



### 加密算法

#### 空加密算法

JWT支持使用空加密算法，可以在header中指定alg为`None`

这样的话，只要把signature设置为空（即不添加signature字段），提交到服务器，任何token都可以通过服务器的验证。举个例子，使用以下的字段

```
{
    "alg" : "None",
    "typ" : "jwt"
}

{
    "user" : "Admin"
}
```

生成的完整token为`ew0KCSJhbGciIDogIk5vbmUiLA0KCSJ0eXAiIDogImp3dCINCn0.ew0KCSJ1c2VyIiA6ICJBZG1pbiINCn0`

(header+'.'+payload，去掉了'.'+signature字段)

空加密算法的设计初衷是用于调试的，但是如果某天开发人员脑阔瓦特了，在生产环境中开启了空加密算法，缺少签名算法，jwt保证信息不被篡改的功能就失效了。攻击者只需要把alg字段设置为None，就可以在payload中构造身份信息，伪造用户身份。





#### 修改RSA加密算法为HMAC



### 爆破

工具 [c-jwt-cracker](https://github.com/brendan-rius/c-jwt-cracker) 

