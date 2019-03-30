---
title: JS call & bind & apply（I）
cover: https://image.littlechao.top/20180203092238000004.jpg
author: 
  nick: Lovae
tags: 
    - JavaScript 
date: 2017-11-21

# 首页每篇文章的子标题
subtitle: 模拟实现、深入理解 call bind apply
categories: JavaScript

---
## call
先给一个官方描述吧：
> call() 方法调用一个函数, 其具有一个指定的this值和分别地提供的参数(参数的列表)。

>可以让call()中的对象调用当前对象所拥有的function。你可以使用call()来实现继承：写一个方法，然后让另外一个新的对象来继承它（而不是在新对象中再写一次这个方法）。
 

#### 一般用法
##### 使用call方法调用匿名函数

在下例中的for循环体内，我们创建了一个匿名函数，然后通过调用该函数的call方法，将每个数组元素作为指定的this值执行了那个匿名函数。这个匿名函数的主要目的是给每个数组元素对象添加一个print方法，这个print方法可以打印出各元素在数组中的正确索引号
```js
var animals = [
  {species: 'Lion', name: 'King'},
  {species: 'Whale', name: 'Fail'}
];

for (var i = 0; i < animals.length; i++) {
  (function (i) { 
    this.print = function () { 
      console.log('#' + i  + ' ' + this.species + ': ' + this.name); 
    } 
    this.print();
  }).call(animals[i], i);
}

```

##### 使用call方法调用函数并且指定上下文的 `this`

举个栗子：
```js
var foo = {
    value: 1
};

function bar() {
    console.log(this.value);
}

bar.call(foo); // 1
```
#### 模拟实现
##### 分析
在上面的第二个例子里，注意两点：
* call 改变了 this 的指向，指向到 foo
* bar 函数执行了

试想当调用 call 的时候，把 foo 对象改造成如下：

```js
var foo = {
    value: 1,
    bar: function() {
        console.log(this.value)
    }
};

foo.bar(); // 1
```
这个时候 this 就指向了 foo，是不是很简单呢？

所以我们模拟的步骤可以分为：

* 将函数设为对象的属性
* 执行该函数
* 删除该函数
##### 模拟第一步
```js
Function.prototype.call1 = function(context) {
  context.fn = this;    //this 指向调用 call1 的函数
  context.fn();
  delete context.fn;
}

//测试一下
var foo = {
    value: 1
};

function bar() {
    console.log(this.value);
}

bar.call2(foo); // 1
```
注意：

##### 模拟第二步：指定参数
```js
Function.prototype.call1 = function(context) {
  context.fn = this;
  var args = Array.from(arguments).slice(1); 
  //获取arguments（传进来的所有参数），变成数组，并去掉第一个参数（context）
  context.fn(...args);   
  // ...是es6的语法，意思是展开后面的对象或数组
  // 例如： ...[1,2,3,4,5] => 1,2,3,4,5
  delete context.fn;
}
```

以下是没有 es6 时的替代方法
```js
Function.prototype.call1 = function(context) {
  context.fn = this;
  var args = [];
  for(var i = 1, len = arguments.length; i < len; i++) {
    args.push('arguments[' + i + ']');
  }
  eval('context.fn(' + args + ')');
  delete context.fn;
}
```

##### 模拟实现第三步

模拟代码已经完成 80%，还有两个小点要注意：

1.this 参数可以传 null，当为 null 的时候，视为指向 window

2.函数是可以有返回值的！

```js
Function.prototype.call1 = function(context) {
  var context = context || window;
  context.fn = this;
  var args = Array.from(arguments).slice(1); 
  //获取arguments（传进来的所有参数），变成数组，并去掉第一个参数（context）
  var result = context.fn(...args);   
  // ...是es6的语法，意思是展开后面的对象或数组
  // 例如： ...[1,2,3,4,5] => 1,2,3,4,5
  delete context.fn;
  
  return result;
}
```

大功告成！ｂ（￣▽￣）ｄ

## apply
> 和 call 一样，只是参数是以数组的形式给出
##### 模拟实现
```js
Function.prototype.apply = function (context, arr) {
    var context = Object(context) || window;
    context.fn = this;

    var result;
    if (!arr) {
        result = context.fn();
    }
    else {
        var args = [];
        for (var i = 0, len = arr.length; i < len; i++) {
            args.push('arr[' + i + ']');
        }
        result = eval('context.fn(' + args + ')')
    }

    delete context.fn;
    return result;
}
```

## bind
> MDN 的解释：bind() 函数会创建一个新函数（称为绑定函数），新函数与被调函数（绑定函数的目标函数）具有相同的函数体（在 ECMAScript 5 规范中内置的call属性）。当新函数被调用时 this 值绑定到 bind() 的第一个参数，该参数不能被重写。
```js
Function.prototype.bind = function (oThis) {
 if (typeof this !== "function") {
   // closest thing possible to the ECMAScript 5 internal IsCallable function
   throw new TypeError("Function.prototype.bind - what is trying to be bound is not callable");
 }

 var aArgs = Array.prototype.slice.call(arguments, 1), 
     fToBind = this, 
     fNOP = function () {},
     fBound = function () {
       return fToBind.apply(this instanceof fNOP && oThis
                              ? this
                              : oThis || window,
                            aArgs.concat(Array.prototype.slice.call(arguments)));
     };

 fNOP.prototype = this.prototype;
 fBound.prototype = new fNOP();

 return fBound;
};
```
## call apply bind 区别
call和apply，bind都是用来改变函数中this的指向
不同的是call和apply不仅改变了函数中this的指向并且立即调用了函数而bind仅仅是替换了this没有调用

apply和call的区别在于当Parent有参数的时候call只能一个一个的赋值 apply可以以数组的方式传递
bind体验了js的预处理，预先处理数据 稍后之行

