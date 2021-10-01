date: 2019-12-15
tags:

- 编程

categories:

- 编程

title: 由cout和printf来对函数传参过程的探讨

---

# 由cout和printf来对函数传参过程的探讨



## 令人疑惑的结果



来看一个代码吧
```c++
#include<stdio.h>
int i=0;
int update(){i++;return i;}
int main()
{
	printf("update():%d i:%d\n",update(),i);
	return 0;
}
```
你觉得它的输出结果是什么
~~update():1 i:1~~ ?
**错**，正确输出是`update():1 i:0`



## 测试



为何会如此,先来做个实验康康吧:
```c++
#include<iostream>
#include <queue>
#include<stdio.h> 
using namespace std;
int i=0;
int update(){i++;return i;};
int main()
{
	printf("----------------------------test printf----------------------------\n");
	printf("before:%d\n",i);
	printf("update():%d i:%d\n",update(),i);
	printf("later:%d\n",i);
	
	printf("before:%d\n",i);
	printf("i:%d update():%d \n",i,update());
	printf("later:%d\n",i);
	
	printf("update1:%d update2:%d update3:%d",update(),update(),update());
}
```

它的输出结果是
```
----------------------------test printf----------------------------
before:0
update():1 i:0
later:1
before:1
i:2 update():2
later:2
update1:5 update2:4 update3:3
```

只有printf是这样的?不，可变参数的打印函数都具有这样的特性
我们来测试一下cout来验证一下

```c++
#include<iostream>
#include <queue>
#include<stdio.h> 
using namespace std;
int i=0;
int update(){i++;return i;};
int main()
{

	cout<<"---------------------------- test cout ----------------------------\n";
	i=0;
	cout<<"before:"<<i<<endl;
	cout<<"update():"<<update()<<" i:"<<i<<endl;
	cout<<"later:"<<i<<endl;
	
	cout<<"before:"<<i<<endl;
	cout<<"i:"<<i<<" update():"<<update()<<endl;
	cout<<"later:"<<i<<endl;
	
	cout<<"update1:"<<update()<<" update2:"<<update()<<" update3:"<<update()<<endl;

}
```

它的输出结果是:
```
---------------------------- test cout ----------------------------
before:0
update():1 i:0
later:1
before:1
i:2 update():2
later:2
update1:5 update2:4 update3:3

```



## 汇编层面的解释



为什么会这样,这就必须说起c和c++中关于函数传参的过程了:

在汇编中函数传参要从最后一个参数到第一个参数分别入栈，这样函数取参数的时候，pop出来的顺序才是1-n。

因此,在`printf("update():%d i:%d\n",update(),i);`中,函数先压入i,再执行update(),将其返回值压入栈。

以上只是理论，真实的环境中由x86,x64,32位,16位机,它们的具体的传参方式略有不同:)  


我们来看一下x64下这一条语句的汇编代码


```
mov     ebx, cs:i
call    _Z6updatev      ; update(void)
mov     r8d, ebx
mov     edx, eax
lea     rcx, aUpdateDID ; "update():%d i:%d\n"
call    _ZL6printfPKcz  ; printf(char const*,...)
```


在X64下,是寄存器传参. 前4个参数分别是 rcx rdx r8 r9进行传参.多余的通过栈传参.从右向左入栈。

上述代码中 格式化字符串在rcx中(最后一个赋值),update()返回值在edx,中第二个赋值。i在r8中第一个赋值 。

但是我们可以看到`mov     ebx, cs:i`程序先将i的值移动到ebx中，再调用update函数,将返回值所在寄存器eax移动到edx中。

虽然过程有点变化,但是最后的结果还是和在汇编中函数传参要从最后一个参数到第一个参数分别入栈。



## 来练习一下吧



最后上一个题目吧//

```c++
#include <stdio.h>
int main()
{
    long long a = 1, b = 2, c = 3;
    printf("%d %d %d\n", a,b,c);
    return 0;
}
```
求输出

[答案和解析](https://blog.csdn.net/u014713819/article/details/29355455)