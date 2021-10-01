categories:
- cve复现
tags:
- cve复现
title: DedeCMS 5.7 plusguestbook.php注入漏洞
---
# DedeCMS 5.7 plus/guestbook.php 注入漏洞

## 利用条件

漏洞成功需要条件：

1. php magic_quotes_gpc=off
2. 漏洞文件存在 plus/guestbook.php dede_guestbook 表当然也要存在。

## 漏洞复现

poc:` www.xxx.com/plus/guestbook.php?action=admin&job=editok&msg=sebug'&id=存在的留言ID `

`/plus/guestbook.php`中为对身份进行验证:

```php
if($action=='admin')
{
    include_once(dirname(__FILE__).'/guestbook/edit.inc.php');
    exit();
}
```

`/plus/guestbook/edit.inc.php`

```php
if($job=='editok')
{
    $remsg = trim($remsg);
    if($remsg!='')
    {
        //管理员回复不过滤HTML
        if($g_isadmin)
        {
            $msg = "<div class=\\'rebox\\'>".$msg."</div>\n".$remsg; 
            //$remsg <br><font color=red>管理员回复：</font>
        }
        else
        {
            $row = $dsql->GetOne("SELECT msg From `#@__guestbook` WHERE id='$id' ");
            $oldmsg = "<div class=\\'rebox\\'>".addslashes($row['msg'])."</div>\n";
            $remsg = trimMsg(cn_substrR($remsg, 1024), 1);
            $msg = $oldmsg.$remsg;
        }
    } 
	$msg = HtmlReplace($msg, -1);
    $dsql->ExecuteNoneQuery("UPDATE `#@__guestbook` SET `msg`='$msg', `posttime`='".time()."' WHERE id='$id' ");
    ShowMsg("成功更改或回复一条留言！", $GUEST_BOOK_POS);
    exit();
}
```

其中:

```php
$msg = HtmlReplace($msg, -1);
    $dsql->ExecuteNoneQuery("UPDATE `#@__guestbook` SET `msg`='$msg', `posttime`='".time()."' WHERE id='$id' ");
```

dedecms中的`common.inc.php`,有这样的一段代码

```php
foreach(Array('_GET','_POST','_COOKIE') as $_request)
    {
        foreach($$_request as $_k => $_v) 
		{
			if($_k == 'nvarname') ${$_k} = $_v;
			else ${$_k} = _RunMagicQuotes($_v);
		}
    }
```

它将`_GET`,`_POST`,`_COOKIE`中的变量放出来,而`_RunMagicQuotes`

```php
function _RunMagicQuotes(&$svar)
{
    if(!get_magic_quotes_gpc())
    {
        ...
    }
    return $svar;
}
```



所以`magic_quotes_gpc=off`的情况下便会引起sql注入





## 参考

 https://www.seebug.org/vuldb/ssvid-89599 