date: 2020-01-25
categories:
- 杂
tags:
- 编程
title: coursera-杜克大学 程序设计与 Web 入门
---
#   coursera-杜克大学 程序设计与 Web 入门

## 在coursera上学习的感受

我觉得最大的优点是:教学和练习同时进行,极大的帮助了我快速掌握知识,这是我最喜欢的一点.但是对于中国境内的视频 加载有点难受,不过可以通过修改hosts来改善.

虽然这种教学方式我很喜欢,但是由于我的课程选择不太适合我自己(太基础了),导致我其实没学到啥,但是我还是硬着头皮看完了(闲得无聊?),选择适合自己的课程真的特别重要

## 提问问题的注意事项

**详细**,**清楚**的描述自己遇到的问题,及自己尝试解决问题的结果



## css语法

```css
.classname{
    preperty:value;
}

#ID{
    preperty:value;
}
tagname {
    preperty:value;
}
;or
tag1 tag2 {
    preperty:value;
}
;这个会修改tag1下的tag2
```



## css资料

### 文档

 https://www.w3schools.com/css/css_border.asp 

### css colors

 https://www.w3schools.com/cssref/css_colors.asp 

 https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Colors/Color_picker_tool

 http://www.w3schools.com/colors/colors_picker.asp



## 用编程解决问题的七步骤

1. Work example by hand. 
   清晰的认识要解决的问题,并手动尝试解决
2.  Write down what you did. 
   写下自己解决问题的过程(只针对这个具体的例子),不要忽略我们理所当然的东西,计算机不会这么想,我们要尽可能的详细,精确的记录我们解决问题的步骤
3.  Find patterns. 
   将过程抽象成解决这个问题的一般性步骤,注意重复的步骤(变为循环语句),注意判断(变为条件语句)
4. Check by hand. 
   用具体的例子来检验抽象的步骤是否能解决问题
5. Translate to code. 
6.  Run test cases. 
7.  Debug failed test cases. 
   一般可以把问题分为:步骤问题(如没考虑某些特殊情况)和将步骤转为程序时(如函数用错)的问题



## js

### 手册

 https://developer.mozilla.org/ 

 https://www.w3schools.com/ 



### 循环

```javascript
var img = new SimpleImage(200, 200);
for(var pixel of img.values())
{//pixel是img.values()的引用
    pixel.setRed(255);
    pixel.setGreen(255);
    pixel.setBlue(0);
}
print(img);
```





### canvas操作

```javascript
var canvas=document.getElementById("show2");
var context = canvas.getContext('2d');
#init
context.clearRect(0, 0, canvas.width, canvas.height);
#clear canvas
context.font = "30px Arial";
context.fillText("Hello World", 10, 50);
#draw a text


context.fillStyle = "red";
context.fillRect(10, 10, 150, 80);
#draw rectangle
```



### 变量赋值

当变量是数字和字符串的时候,会生成一个拷贝,再赋值给目标

但如果对象是数组和对象时,这会将引用赋值给目标,此时修改新变量也会影响到旧变量



## html标签

### input

#### color

`<input type="color" id="color" value="pink" onchange="colorchange()">`

#### botton

`<input type="button" value="make pink" onclick="makepink()">`

#### range

`<input type="range" min="10" max="100" value="10" id="slr" oninput=doSquare()>`

#### upload



### 事件

onclick,onchage,oninput

## 给自己的建议

>合格的程序员不应该浪费很多时间用于程序调试,他们应该一开始就不要把故障引入.
>
>----迪杰斯特拉



## debug

![image2319](https://i.loli.net/2020/01/20/bmQ5AvDenOHlqP7.png)



发现bug->明确目的(了解bug发生的原因及解决方法)->收集信息->(多次)提出猜想(where,why)(要具有可测试性)->验证->解决



## final work

 https://codepen.io/explorersss/full/wvBOwmR 