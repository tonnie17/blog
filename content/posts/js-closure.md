+++
title = "JS中的闭包概念"
date = 2017-03-14T00:00:00+08:00
tags = ["javascript"]
categories = [""]
draft = false
+++

像其他大多数动态编程语言一样，js也采用词法作用域，换而言之，函数的执行依赖于变量作用域，这个作用域是在函数定义时决定的，而不是函数调用时决定的。

```javascript
var scope = "global scope";
function checkscope() {
    var scope = "local scope";
    function f() { return scope; }
    return f;
}

var local_func = checkscope()
local_func()    //  = "local scope"
```

观察上面这段代码，首先在外层定义了一个名为scope的全局变量，接着定义了一个函数checkscope()，在函数中也定义了一个名为scope的局部变量以及一个返回该变量的嵌套函数f()，最后在外层函数的最后一行把该嵌套函数返回。代码的最后把嵌套函数拿到再调用，输出local scope。

下面来窥探一下代码的原理，我们都知道在C语言这种底层编程语言中，函数中的非静态变量是储存在CPU栈中的，当函数调用结束后堆栈中的空间也由编译器自动释放。在JS中，有一个叫作用域链的对象，简单的说，它就是一个对象列表，而不是一个绑定的栈，函数调用的时候会创建一个新的对象保存局部变量，随之把该对象添加到作用域链中的对象列表中，当函数返回时，便把该对象删除，也即是说，当该保存变量的对象的引用数为0时，它就会被垃圾回收器清除掉，然而，在闭包中，外层函数返回的嵌套函数依然保留着作用域链中对象的外部引用，因此，该对象并不会被垃圾回收删除，而返回的嵌套函数此时便能通过作用域链访问和修改到外部变量了，这就是闭包的大概思想。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-538dd6e7e1df07fb712b7a44ae7468d0_1440w.jpg)
