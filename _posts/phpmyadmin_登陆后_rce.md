categories:
- cve复现
title: phpmyadmin_登陆后_rce
---
# phpmyadmin登陆后rce



## 漏洞信息

>在PHP5.4.7以前，`preg_replace`的第一个参数可以利用\0进行截断，并将正则模式修改为e。众所周知，e模式的正则支持执行代码，此时将可构造一个任意代码执行漏洞。
>
>以下版本受到影响：
>
>- 4.0.10.16之前4.0.x版本
>- 4.4.15.7之前4.4.x版本
>- 4.6.3之前4.6.x版本（实际上由于该版本要求PHP5.5+，所以无法复现本漏洞）



## 漏洞分析

全局搜索preg_replace,发现以下可能有可控变量的位置:

容易找到

`TableSearch.lib.php`

```php
function _getRegexReplaceRows($columnIndex, $find, $replaceWith, $charSet)
    {
        $column = $this->_columnNames[$columnIndex];
        $sql_query = "SELECT "
            . PMA_Util::backquote($column) . ","
            . " 1," // to add an extra column that will have replaced value
            . " COUNT(*)"
            . " FROM " . PMA_Util::backquote($this->_db)
            . "." . PMA_Util::backquote($this->_table)
            . " WHERE " . PMA_Util::backquote($column)
            . " RLIKE '" . PMA_Util::sqlAddSlashes($find) . "' COLLATE "
            . $charSet . "_bin"; // here we
            // change the collation of the 2nd operand to a case sensitive
            // binary collation to make sure that the comparison is case sensitive
        $sql_query .= " GROUP BY " . PMA_Util::backquote($column)
            . " ORDER BY " . PMA_Util::backquote($column) . " ASC";

        $result = $GLOBALS['dbi']->fetchResult($sql_query, 0);

        if (is_array($result)) {
            foreach ($result as $index=>$row) {
                $result[$index][1] = preg_replace(
                    "/" . $find . "/",
                    $replaceWith,
                    $row[0]
                );
            }
        }
        return $result;
    }
```



如果我们可以控制`$result和$replaceWith`那么我们就可以执行任意命令

![image1674](https://i.loli.net/2020/04/03/qo5ZbfuJHKA9d6m.png)

我们先看`getReplacePreview`

![image1764](https://i.loli.net/2020/04/03/vnk9IZUxJ5YoMbW.png)

再查看调用`getReplacePreview`的函数

![image1858](https://i.loli.net/2020/04/03/JxKS1TioFz4w7ql.png)

去看第二个和第三个明显不行,这里就不贴出来了

再跟进`tbl_find_replace.php`

```php
if (isset($_POST['find'])) {
    $preview = $table_search->getReplacePreview(
        $_POST['columnIndex'],
        $_POST['find'],
        $_POST['replaceWith'],
        $_POST['useRegex'],
        $connectionCharSet
    );
    $response->addJSON('preview', $preview);
    exit;
}
```

明显白给



我们再回到`_getRegexReplaceRows`去查看它的第二个调用函数

![image2318](https://i.loli.net/2020/04/03/OhCPcviVT8tm6ZK.png)

再查看调用replace的函数

![image2400](https://i.loli.net/2020/04/03/zT21ymqOYvUf5WE.png)

同样后面两个还是不行

在跟进`tbl_find_replace.php`

![image2504](https://i.loli.net/2020/04/03/j4JdNfKa8up2BCc.png)

还是白给,两个调用链都是可以rce的,我们以前面那个为例

先往表中添加一条数据`INSERT INTO `flags` (`username`,`flag`,`msgid`) VALUES (UNHEX('302F6500'),1,1);`

直接抓包,修改

![image2701](https://i.loli.net/2020/04/03/FwSzMuNI4nDadBh.png)



![image2768](https://i.loli.net/2020/04/03/QZE5fMTnUhPXdqV.png)

成功

## poc

这个poc直接抄个别人的,懒得写了

` https://www.exploit-db.com/exploits/40185 `

