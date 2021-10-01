date: 2020-07-09
categories:
- web
title: bypass CORS
---
# bypass CORS

## 什么是CORS

CORS: Cross Origin Resource Share,设置在服务端，客户端执行的一种策略。

> 出于安全原因，浏览器限制从脚本内发起的跨源HTTP请求或者 跨站请求可以正常发起，但是返回结果被浏览器拦截了 。 例如，XMLHttpRequest和Fetch API遵循同源策略。 这意味着使用这些API的Web应用程序只能从加载应用程序的同一个域请求HTTP资源，除非响应报文包含了正确CORS响应头。 



## 相关的字段

响应：

```
Access-Control-Allow-Origin
Access-Control-Expose-Headers
Access-Control-Max-Age
Access-Control-Allow-Credentials
Access-Control-Allow-Methods
Access-Control-Allow-Headers

```

请求：

```
Origin
Access-Control-Request-Method
Access-Control-Request-Headers

```



## CORS配置错误

### 根据某些字段来设置ACAO头

```
GET /sensitive-victim-data HTTP/1.1
Host: vulnerable-website.com
Origin: https://malicious-website.com
Cookie: sessionid=...

根据Origin头来设置Access-Control-Allow-Origin

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://malicious-website.com
Access-Control-Allow-Credentials: true
```

### 对Origin头的错误解析

因为要对子域名的支持，所以有些网站会这样写:

```
if Origin.domain startwith "domain.com":
	return "Access-Control-Allow-Origin: "+Origin

```

或者

```
if Origin.domain endwith "domain.com":
	return "Access-Control-Allow-Origin: "+Origin

```

像这种都很好绕过：

domain.com.evildomain.com 或者 evildomain-domain.com



### 白名单中存在:null

 Origin支持值null，在某些情况下，浏览器可能会在Origin标头中发送值null：

- 跨站点重定向

- 来自序列化数据的请求

- 使用`file`协议的请求

- 沙盒中的跨域请求

## 利用受信任的域

在受信任的域上找到xss，就可以绕过CORS

## 中间人攻击

23333

## 参考

 https://portswigger.net/web-security/cors 

 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS 