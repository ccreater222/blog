date: 2020-04-20
categories:
- 比赛
tags:
- wp
- ctf
title: 百越杯总结
---
# 百越杯总结



## include,require

如果**文件被包含两次**，PHP 5   **发出致命错误**因为函数已经被定义，但是 PHP 4 不会对在   [return](mk:@MSITStore:D:\ctf\手册\php_manual_zh.chm::/res/function.return.html) 之后定义的函数报错。推荐使用   [include_once](mk:@MSITStore:D:\ctf\手册\php_manual_zh.chm::/res/function.include-once.html) 而不是检查文件是否已包含并在包含文件中有条件返回。 

## include_once,require_once



*include_once* 语句在脚本执行期间包含并运行指定文件。此行为和   [include](mk:@MSITStore:D:\ctf\手册\php_manual_zh.chm::/res/function.include.html)   语句类似，唯一区别是如果该文件中已经被包含过，则不会再次包含。如同此语句名字暗示的那样，只会包含一次。

*require_once* 语句和 [require](mk:@MSITStore:D:\ctf\手册\php_manual_zh.chm::/res/function.require.html)   语句完全相同，唯一区别是 PHP 会检查该文件是否已经被包含过，如果是则不会再次包含。 

**如果所包含文件不存在,include_once不会退出,而require_once会直接退出**

## 比赛失误原因

### log没挂好







### 没有找到有效的攻击流量



grep这个命令不够熟悉,



#### system

我的过滤条件变迁:

`"flag" => "/flag"`,到这里后我就没有继续过滤,"/flag"其实还是有很多干扰信息的,我就差一步



`grep "/flag " */* `

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g91fl7ptrij30k20arac7.jpg)



`grep "/flag " */* -C300 | grep "end pageheader" -C 10`

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g91fk6h3jxj30oa05cdfx.jpg)





### 没找到的洞

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g91c2qcxxyj30pl01vdgk.jpg)

php的web主要也就这两的洞

#### grayscale

`include $_GET['img'];`配合有后门的图片m.jpg

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g91c4zci7cj30f001g3yg.jpg)

开始我直接搜索`<?`和`<%` 没搜索到就以为是误报,??也能执行?????

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g91c93p00wj307100zjr6.jpg)

用strings查看////,编码问题导致??,以后用ascii码来查看



#### ssystem

虽然找到了exec那个洞

但是在index.php中：

```php
<?php
if (isset($_GET['page'])){
    $page = $_GET['page'];

}else{
        $page = 'chart.php';
}
?>
<?php
	include_once "$page";
?>
```

可以配合文件上传拿到shell或者是直接include /flag也能拿到flag





### 洞没修好

#### SSystem

```php
    <?php
    if (isset($_POST['name'])){
        $name = $_POST['name'];
        exec("tar -cf backup/$name images/*.jpg");
        echo "<div class=\"alert alert-success\" role=\"alert\">
                导出成功,<a href='backup/$name'>点击下载</a></div>";
    }
    ?>
```



原本以为`$name`再参数位置,用escapeshellarg就可以了

但是!!!!!!!虽然escapeshellarg会把$name放在引号里面

```
escapeshellarg() 将给字符串增加一个单引号并且能引用或者转码任何已经存在的单引号，这样以确保能够直接将一个字符串传入 shell 函数，并且还是确保安全的。对于用户输入的部分参数就应该使用这个函数。shell 函数包含 exec(), system() 执行运算符 。 
```

php手册里解释是单引号,实际却是` tar -cf backup/"123" images/*.jpg `

?????这是在我本机php5.6的环境下,php7环境也是双引号???

然后在自己的服务器上之前是双引号后面是单引号????

由于不知道靶场上会怎样,我也不确定我用escapeshellarg是不是修好了,然后这题就修补失败了





比较好的修复方案(在实现原功能的情况下去掉命令执行):

```
exec("tar -cf backup/aa.tar images/*.jpg");
rename("backup/aa.tar","backup/$name.tar");
```





### 没找到check没过的原因

check的id是172.16.4.7

但是直接搜索 HTTP 没找到check没过的原因,我猜因为拖下来的log只有前半小时,而开局的是否我们的check好像是过的



## 总结

我看到洞的时候没有修好

 