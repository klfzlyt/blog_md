---
title: JavaScript中存在纯函数吗？
date: 2017-06-12 21:20:52
tags: 自译
---

### JavaScript中存在纯函数吗?(自译)

 <https://hackernoon.com/do-pure-functions-exist-in-javascript-b128ed5f0ed2>

### 什么是纯函数？
满足下面两个条件： 
<!--more-->
1) The function returns exactly the same result every time it’s called with the same set of arguments.
2) Evaluation of the function does not modify some state outside its scope nor has an observable interaction with the outside world besides returning a value. (No side effects.)

Sometimes, a third condition is added: ‘Relies on no external mutable state.’ This, in fact, is redundant as such dependency on mutable variable would inevitably lead to breaking the first condition, too.


有下面四个函数
* doubleA
* doubleB
* doubleC
* doubleD
```javascript
// A: Simple multiplication
function doubleA(n) {
  return n * 2
}

// B: With a variable
var two = 2
function doubleB(n) {
  return n * two
}

// C: With a helper function
function getTwo() {
  return 2
}
function doubleC(n) {
  return n * getTwo()
}

// D: Mapping an array
function doubleD(arr) {
  return arr.map(n => n * 2)
}

```
大多数人认为doubleB不纯，doubleA, doubleC, doubleD 为纯函数.

```javascript
doubleB(1) // -> 2
two = 3
doubleB(1) // -> 3
```
可以知道doubleB不纯

### 然而事实上：上述4个函数都不纯？
为什么？

1. 对于doubleC

```javascript
doubleC(1) // -> 2
getTwo = function() { return 3 }
doubleC(1) // -> 3
```

2. 对于doubleD 

“Map, filter, reduce. Repeat.”在函数式编程中是核心的数据变换方法，在用到他们的时候应该谨慎

```js
doubleD([1]) // -> [2]
Array.prototype.map = function() {
  return [3]
}
doubleD([1]) // -> [3]
```
两次调用结果不一致，因而doubleD也不纯。

3. doubleA 

乘法操作符是纯的？？？

有个不常用的点：JavaScript also calls valueOf method of an object when it expects a number (or a boolean, or a function). What's left is to make this function return different value each time it is invoked. 

```js
var o = {
  valueOf: Math.random
}
doubleA(o) // -> 1.7709942335937932
doubleA(o) // -> 1.2600863386367704
```
因而 doubleA不纯

#### 到底什么是纯函数？

没有绝对的定义，一个函数可能在一个项目里纯，在其他项目里不纯


### 备注：
一些阻止上述小伎俩的手段：
对于doubleB： 
```js
var doubleB = (function () {
  var two = 2
  return function (n) {
    return n * two
  }
})()
```
或是

```js
const two = 2
const doubleB = (n) => n * two
```

对于doubleA

```js
function doubleA(n) {
  if (typeof n !== 'number') return NaN
  return n * 2
}
```

总之不要忘记保证一个函数真正的为纯函数