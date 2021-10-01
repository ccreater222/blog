categories:
- 编程
title: maven学习笔记
---
# maven学习笔记



## 学习网站

 https://www.imooc.com/learn/443 (这个挺好的)

 https://www.runoob.com/maven/maven-tutorial.html 

## maven功能

- 构建
- 文档生成
- 报告
- 依赖
- SCMs
- 发布
- 分发
- 邮件列表

## 约定配置

Maven 提倡使用一个共同的标准目录结构，Maven 使用约定优于配置的原则，大家尽可能的遵守这样的目录结构。如下所示：

| 目录                               | 目的                                                         |
| :--------------------------------- | :----------------------------------------------------------- |
| ${basedir}                         | 存放pom.xml和所有的子目录                                    |
| ${basedir}/src/main/java           | 项目的java源代码                                             |
| ${basedir}/src/main/resources      | 项目的资源，比如说property文件，springmvc.xml                |
| ${basedir}/src/test/java           | 项目的测试类，比如说Junit代码                                |
| ${basedir}/src/test/resources      | 测试用的资源                                                 |
| ${basedir}/src/main/webapp/WEB-INF | web应用文件目录，web项目的信息，比如存放web.xml、本地图片、jsp视图页面 |
| ${basedir}/target                  | 打包输出目录                                                 |
| ${basedir}/target/classes          | 编译输出目录                                                 |
| ${basedir}/target/test-classes     | 测试编译输出目录                                             |
| Test.java                          | Maven只会自动运行符合该命名规则的测试类                      |
| ~/.m2/repository                   | Maven默认的本地仓库目录位置                                  |



## Maven POM

 POM( Project Object Model，项目对象模型 ) 是 Maven 工程的基本工作单元，是一个XML文件 ,用于描述如何构建项目

在创建 POM 之前，我们首先需要描述项目组 (groupId), 项目的唯一ID。 

### 模板

```xml
<project xmlns = "http://maven.apache.org/POM/4.0.0"
    xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation = "http://maven.apache.org/POM/4.0.0
    http://maven.apache.org/xsd/maven-4.0.0.xsd">
 
    <!-- 模型版本 -->
    <modelVersion>4.0.0</modelVersion>
    <!-- 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.companyname.project-group，maven会将该项目打成的jar包放本地路径：/com/companyname/project-group -->
    <groupId>com.companyname.project-group(公司网站反写+项目名)</groupId>
 
    <!-- 项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的 -->
    <artifactId>project+模块名</artifactId>
 
    <!-- 版本号 -->
    <version>1.0</version>
</project>
```

### 常见标签解释

```xml
<project xmlns = "http://maven.apache.org/POM/4.0.0"
    xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation = "http://maven.apache.org/POM/4.0.0
    http://maven.apache.org/xsd/maven-4.0.0.xsd">
 
    <!-- 模型版本 -->
    <modelVersion>4.0.0</modelVersion>
    <!-- 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.companyname.project-group，maven会将该项目打成的jar包放本地路径：/com/companyname/project-group -->
    <groupId>com.companyname.project-group(公司网站反写+项目名)</groupId>
 
    <!-- 项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的 -->
    <artifactId>project+模块名</artifactId>
 
    <!-- 版本号
 	第一个数字表示大版本号
	第二个数字表示分支版本号
	第三个数字表示小版本号
	snapshot 快照
	alpha 内部测试
	beta 公测
	Release 稳定版本
	GA 正式发布
	-->
    <version>1.0.0-xxx</version>
    <!-- 打包方式，默认是jar,还有:war zip pom-->
    <packaging></packaging>
    <!--项目描述名-->
    <name></name>
    <!--项目地址 -->
    <url></url>
    <!--项目描述 -->
    <descriptions></descriptions>
    <!--开发人员信息 -->
    <developers></developers>
    <dependeies>
        <dependency>
            <groupId>com.companyname.project-group(公司网站反写+项目名)</groupId>

    		<artifactId>project+模块名</artifactId>
    		<version>1.0.0-xxx</version>
            <type></type>
            <!-- 在test中有用 -->
            <scope>test</scope>
            <!--设置依赖是否可选，默认False，设为False则子项目默认继承，否则得显示声明-->
            <optional></optional>
            <!-- 排除依赖传递列表
				如A依赖B，B依赖C，而A不依赖C，此时就可以在此处添加C，来排除依赖
 			-->
            
            <exclusions>
                <exclusion>
                </exclusion>
            </exclusions>
        </dependency>
    </dependeies>
    <!-- 依赖的管理，子pom自动继承，但是本身并不依赖 -->
    <dependencyManagement>
       <dependeies>
        	<dependency>
           </dependency>
        </dependeies> 
    </dependencyManagement>
    <build>
        <plugins>
            <!-- 详情见在某个阶段使用插件 -->
        </plugins>
    </build>
    <!-- 聚合运行多个maven项目,一口气全部编译 -->
    <modules>
        <module>
        </module>
    </modules>
</project>
```



### 执行main函数

在pom下的project添加：

```xml
<build>
 <plugins>
  <plugin>
   <groupId>org.codehaus.mojo</groupId>
   <artifactId>exec-maven-plugin</artifactId>
   <version>1.1.1</version>
   <executions>
    <execution>
     <phase>test</phase>
     <goals>
      <goal>java</goal>
     </goals>
     <configuration>
      <mainClass>com.vineetmanohar.module.CodeGenerator</mainClass>
      <arguments>
       <argument>arg0</argument>
       <argument>arg1</argument>
      </arguments>
     </configuration>
    </execution>
   </executions>
  </plugin>
 </plugins>
</build>
```





### 依赖范围

- compile
  默认的范围，编译测试运行都有效
- provided
  在测试和编译时有效
- runtime
  在测试和运行时有效
- test
  只有在测试中有效



### 排除依赖

在pom中的project下添加

```xml
<!-- 排除依赖传递列表
				如A依赖B，B依赖C，而A不依赖C，此时就可以在此处添加C，来排除依赖
 			-->
            
            <exclusions>
                <exclusion>
                </exclusion>
            </exclusions>
```



### 依赖冲突解决原则

#### 短路优先

```
A -> B -> C -> X1
A -> D -> X2
#A -> D代表 A依赖于D
此时A依赖于X2
```





#### 先声明先优先

如果路径长度相同(依赖链)，则谁先声明，先解析谁



### 聚合



在容器pom中修改packaging为pom，添加modules

#### 示例

现有maven项目:maven01和maven02

新建maven项目：maven03,三个项目均在同一个文件夹下

在maven03中修改packaging为pom，添加如下modules

```xml
<modules>
    <module>
        ../maven01
    </module>
    <module>
        ../maven02
    </module>
</modules>
```



### 继承

当多个maven模块都依赖一个相同的库时，多次声明显得太麻烦此时可以用继承来解决

#### 示例

现有两个maven项目(maven01,maven02)都使用junit:3.8.1

新建父maven项目:maven03，其对应的pom文件为：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>top.ccreater</groupId>
  <artifactId>maven03</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>pom</packaging>

  <name>maven03</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>


	<dependencyManagement>
	  <dependencies>
	    <dependency>
	      <groupId>junit</groupId>
	      <artifactId>junit</artifactId>
	      <version>3.8.1</version>
	      <scope>test</scope>
	    </dependency>
  		</dependencies>
	</dependencyManagement>
</project>

```

将相同的依赖都添加到dependencyManagement下，并修改packaging为pom

maven01和maven02的pom文件均为：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>top.ccreater</groupId>
  <artifactId>maven02</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>maven02</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
	<parent>
	  <groupId>top.ccreater</groupId>
	  <artifactId>maven03(/maven02)</artifactId>
	  <version>0.0.1-SNAPSHOT</version>
	</parent>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
    </dependency>
  </dependencies>
</project>

```







### Super POM

所有的 POM 都继承自一个父 POM（无论是否显式定义了这个父 POM）。父 POM 也被称作 **Super POM**，它包含了一些可以被继承的默认设置。

Maven 使用 effective pom（Super pom 加上工程自己的配置）来执行相关的目标，它帮助开发者在 pom.xml 中做尽可能少的配置，当然这些配置可以被方便的重写。

查看 Super POM 默认配置的一个简单方法是执行以下命令：**mvn help:effective-pom** （需要在当前目录下新建一个pom.xml）

### POM 标签详解
https://www.runoob.com/maven/maven-pom.html

## Maven 构建生命周期

 Maven 构建生命周期定义了一个项目构建跟发布的过程。 

Maven 有以下三个标准的生命周期：

- **clean**：项目清理的处理
- **default(或 build)**：项目部署的处理
- **site**：项目站点文档创建的处理

### 典型的生命构建周期

| 阶段          | 处理     | 描述                                                     |
| :------------ | :------- | :------------------------------------------------------- |
| 验证 validate | 验证项目 | 验证项目是否正确且所有必须信息是可用的                   |
| 编译 compile  | 执行编译 | 源代码编译在此阶段完成                                   |
| 测试 Test     | 测试     | 使用适当的单元测试框架（例如JUnit）运行测试。            |
| 包装 package  | 打包     | 创建JAR/WAR包如在 pom.xml 中定义提及的包                 |
| 检查 verify   | 检查     | 对集成测试的结果进行检查，以保证质量达标                 |
| 安装 install  | 安装     | 安装打包的项目到本地仓库，以供其他项目使用               |
| 部署 deploy   | 部署     | 拷贝最终的工程包到远程仓库中，以共享给其他开发人员和工程 |

如果执行package，那么clean,test等都会被执行

### 构建阶段由插件目标构成

 一个插件目标代表一个特定的任务（比构建阶段更为精细），这有助于项目的构建和管理。这些目标可能被绑定到多个阶段或者无绑定。不绑定到任何构建阶段的目标可以在构建生命周期之外通过直接调用执行。这些目标的执行顺序取决于调用目标和构建阶段的顺序。 

插件目标解释： 一个插件有多个目标，每个功能就是一个*插件目标*，所以一个插件有多个功能。 



## maven 常用命令

```shell
mvn -v
mvn compile 编译
mvn test 测试
mvn package 打包项目生成jar文件
mvn clearn 删除target 目录
mvn install 安装jar包到本地仓库
创建目录的两种方式
1. 交互式
mvn archetype:generate 
2. 非交互式
mvn archetype:generate -DgroupId=com.companyname.bank -DartifactId=consumerBanking -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```



## 在某个阶段使用插件

 http://maven.apache.org/plugins 

各个插件下有个example config如： http://maven.apache.org/plugins/maven-source-plugin/ 

在pom.xml文件中project下添加

```xml
<build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>3.2.0</version>
        <!--  <configuration>
          <outputDirectory>/absolute/path/to/the/output/directory</outputDirectory>
          <finalName>filename-of-generated-jar-file</finalName>
          <attach>false</attach>
        </configuration>-->
        <executions>
        	<execution>
        		<phase>package</phase>
        		<goals>
        		<goal>jar-no-fork</goal>
                <!-- 在package阶段执行source:jar-no-fork -->
        	</goals>
        	</execution>
        	
        </executions>
      </plugin>
    </plugins>
  </build>
```



## maven配置文件

### 添加镜像仓库

在mirrors下添加

```xml
<mirror>
		<id>alimaven</id>
		<mirrorOf>central</mirrorOf><!-- 哪个仓库的镜像，利用*来匹配所有仓库 -->
		<name>aliyun maven</name>
		<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
	</mirror>
	<mirror>
		<id>jcenter</id>
		<mirrorOf>central</mirrorOf>
		<name>jcenter.bintray.com</name>
		<url>http://jcenter.bintray.com/</url>
	</mirror>

	<mirror>
		<id>repo1</id>
		<mirrorOf>central</mirrorOf>
		<name>Human Readable Name for this Mirror.</name>
		<url>http://repo1.maven.org/maven2/</url>
	</mirror>
	<mirror>
		<id>repo2</id>
		<mirrorOf>central</mirrorOf>
		<name>Human Readable Name for this Mirror.</name>
		<url>http://repo2.maven.org/maven2/</url>
	</mirror>
```







### 修改本地仓库地址

修改`<localRepository>E:/apache-maven-3.6.3/repository</localRepository>`



## 常用插件介绍

 jetty : servlet 本地测试插件

 [tomcat]( http://tomcat.apache.org/maven-plugin.html )  :  tomcat  本地测试插件