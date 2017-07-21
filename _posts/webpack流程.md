---
title: webpack内部流程
date: 2017-04-28 21:20:52
tags: 基础工具
---

### webpack的一切都建立在其插件机制之上
#### webpack是什么？

![](http://p1.meituan.net/dpgroup/c048c5bc511979147aa5cb086bfb8d2239047.png)

webpack 类似C++,java中的编译器，有编译，链接的能力 

那么为什么是编译器？ 

 webpack也需要做词法分析，语法分析，也需要把代码合起来
注意webapck的语法分析重点分析**require即为依赖的分析**，要区分babel是对整个业务代码的分析
webpack是基于node的能力的
---

<!--more-->

下图是webpack的整个框架流程，可以看到有complation,compiler

 
![f](http://p0.meituan.net/dpgroup/fe6a3a12e407e13aa74e57612b29c6f780336.png)


更详细的这张阿里博客的图很能说明问题
![流程图](http://img.alicdn.com/tps/TB1GVGFNXXXXXaTapXXXXXXXXXX-4436-4244.jpg)
地址：http://taobaofed.org/blog/2016/08/24/react-key/
已经说的非常详细了


下图是webpack的内部类结构图
![交互](http://p1.meituan.net/dpgroup/302e7d10adb2cf1f81f8660337465690307284.png)


每次编译的complation对象:
![](https://img.alicdn.com/tps/TB1UgS4NXXXXXXZXVXXXXXXXXXX-693-940.png)


webpack是基于tapable
![](http://ww4.sinaimg.cn/large/006tNbRwgw1fa8w58kxspj30bs0m9wgi.jpg)

关于tapable:
https://segmentfault.com/a/1190000008060440

tapable是一个事件系统，有关于事件一系列同/异步，串并行事件处理机制，提供事件注册等能力，如图:
![](https://sfault-image.b0.upaiyun.com/257/666/2576661044-58735466843de_articlex)
下面是tapable的一些简单用法

```js
var Tapable = require("tapable");

function MyClass() {
    Tapable.call(this);
}

MyClass.prototype = Object.create(Tapable.prototype);

MyClass.prototype.method = function() {};

//plugin

function plugin(){

}
plugin.prototype.apply = function(){
    console.log('plugin plugin tapable')
};


let tap =new MyClass();
//注册事件
tap.plugin('first',function(args,cb){
    console.log('first',args)
    cb()

});
tap.plugin('first',function(args,cb){
    console.log('first 2',args)
    cb()

});
tap.plugin('first',function(args,cb){
    console.log('first 3',args)
    cb()

});
tap.apply(new plugin());
//触发事件
//tap.applyPlugins('first');
tap.applyPluginsAsync('first',{a:1}, function () {
    console.log('async')
})
```
源码中，webpack在build模块时 (调用doBuild方法)，要先调用相应的loader对resource进行加工，生成一段js代码后交给acorn解析生成AST.所以不管是css文件，还是jpg文件，还是html模版， 最终经过loader处理会变成一个module：一段js代码。

webpack使用acorn解析每一个经loader处理过的source，并且成AST，然后遍历所有节点，当遇到require调用时，会分析是AMD的还是CMD的调用，或者是require.ensure . 我们不再分析AST的遍历过程了。

下图是webpack的时间线流程
![](http://p0.meituan.net/dpgroup/c527baf97e04d3e9c09aa1135398259271949.png)
还是那2个关键的compiler,compilation

分析完语法树之后，进行addchunk等操作，之后就调用模板生成器生成代码（assets）
每次执行都有对应的事件发出（emit等）

---

看看babel插件机制：

以babel为例，当webpack发现模块名称匹配test中的正则/js[x]?的时候。

它会将当前模块作为参数传入babel函数处理，babel([当前模块资源的引用])。

函数执行的结果将会缓存在webpack的compilation对象上，并分配唯一的id 。

以上的这一步，非常非常关键。唯一的id值决定了webpack在最后的编译结果中，是否会存在重复代码。 而缓存在compilation对象上，则决定了webpack可以在plugin阶段直接拿取模块资源进行二度加工。

相关参考资料：<http://taobaofed.org/blog/2016/08/24/react-key/>

 
