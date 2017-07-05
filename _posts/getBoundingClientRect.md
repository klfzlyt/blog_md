---
title: domAPI/css
date: 2016-06-12 21:20:52
tags: 前端
---

Reselect 解决selector中计算量过大的问题

multireducer 解决state中多个list导致state过大的问题（拆分list到state）

reducer enhance ---- 高阶reducer

acdlite/redux-actions 是actionCreactor的工厂方法

javascript 可以说都是传值的，对于引用类型来说是应该叫传共享调用 call by share
ref:https://github.com/nodejh/nodejh.github.io/issues/32

getBoundingClientRect的top的是盒模型上边缘与视口顶端的距离
其bottom为盒模型下边缘与视口顶端的距离

text-shadow: h-shadow v-shadow blur color;
水平方向，垂直方向


Object.getOwnPropertyNames()方法返回一个由指定对象的所有自身属性的属性名（包括不可枚举属性）组成的数组。

Object.getOwnPropertyNames 返回一个数组，该数组对元素是 obj 自身拥有的枚举或不可枚举属性名称字符串。 数组中枚举属性的顺序与通过 for...in 循环（或 Object.keys）迭代该对象属性时一致。 数组中不可枚举属性的顺序未定义。


下面的例子演示了该方法不会获取到原型链上的属性：

function ParentClass() {}
ParentClass.prototype.inheritedMethod = function() {};

function ChildClass() {
  this.prop = 5;
  this.method = function() {};
}

ChildClass.prototype = new ParentClass;
ChildClass.prototype.prototypeMethod = function() {};

console.log(
  Object.getOwnPropertyNames(
    new ChildClass()  // ["prop", "method"]
  )
);


1.jquery.dotdotdot

2：
.header{
  overflow : hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

单行
overflow: hidden;
white-space: nowrap;
text-overflow: ellipsis;