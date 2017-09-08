---
title: 实现简易MVVM
date: 2017-09-07 21:20:52
tags: 模式 mvvm
---

### 前言
mvvm双向绑定要解决的二个问题
1. 数据到ui的映射问题
2. ui到数据的映射问题

对于ui到数据的映射，比较简单，先天的的dom事件及其相关的响应函数即可以把相关交互映射到数据上，
为了实现数据到ui的问题，要使得`ob.a = 1`能真实反映在UI上，在实现上可以使用劫持数据属性的做法，
即为`Object.defineProperty`接口，这样能劫持ob.a的赋值过程，类似于`C#`属性，我们代理一个对象的方式如下：
```js
function proxys(data) {
	if (!data || typeof data !== 'object') {
		return;
	}
	Object.keys(data).forEach((key, index) => {
		let val = data[key];
		proxys(val);
		Object.defineProperty(data, key, {
			configurable: true,
			enumerable: true,
			get: function() {
				console.log('get~~~!');				
				return val;
			},
			set: function(newValue) {
				console.log('set~~~!');		 
				val = newValue;			 
			},
		});
	});
}
```
我们代理一个对象的属性获取过程，`proxys`递归调用以处理嵌套对象的情形,**需要注意我们要通过闭包进行赋值**，不能直接进行，如:
```js
            get: function() {
				console.log('get~~~!');				
				return data[key];
			},
```
这样由于闭包内引入了data,会不断调用`get`，形成死循环。
在我们代理了一个对象的属性获取行为后，我们便可以在对象属性设置或者获取过程做一些事情，运用常用的观察者模式，我们可以代理设置过程，在赋值的时候利用通知，将新值通知到具体的被观察者。我们实现一个观察者和一个被观察者：
<script async src="//en.jsrun.net/NYiKp/embed/all/light/"></script>
 
我们在`proxys`中加入一个`observebal`被观察者，并利用get，在get中加入观察者，而观察者，挂载在类Observebal的静态属性上，并在我们实例化一个`Observer`时，进行调用get，这样，当我们执行以下代码时，一个`Observebal`的实例就会得到他们对应的观察者（例子中为2个）,因而我们执行`model.a = 22222;`时，`console.log('observer1', value);`便会通过`notify`得到触发.
```js
let model = { a: 1, b: 2 };
let observal = new Observebal(model);
//监听a属性变化
Observebal.observer = new Observer(model, 'a', value => {
	console.log('observer1', value);
});
//监听b属性变化
Observebal.observer = new Observer(model, 'b', value => {
	console.log('observer2', value);
});
model.a = 22222;
```
<!--more-->
这里的get写法很妙，不断的覆盖不断的取值，感觉`_.get`似乎应该也是类似实现
```js
	getter(vm, expr) {
		var exps = expr.split('.');
		//只要不进行ob.xxx的修改就可以了，指针重新覆盖
		let obj = vm;
		for (var i = 0, len = exps.length; i < len; i++) {
			if (!obj) return;
			obj = obj[exps[i]];
		}
		return obj;
	}
```
### 指令编辑器

接下来只需要实现一个指令编辑器就可以，我们的思路是
实现数据到ui的通知我们的思路是:
数据变化----->回调函数（引用有真实dom的闭包）----->利用dom api改变dom
指令编辑器的作用就是执行后面两步，我们可以利用这个思路，先拿到dom的引用，这里拿取
dom引用的办法有很多，要不是利用模板字符串转换成dom,从而可以进行遍历，要不是直接写在页面中
，让html初始化了dom,然后我们再进行更改。我们最关键的一步就是我们遍历dom树，拿到了每个dom结点的
引用，之后解析dom结点上的指令，之后做一个闭包函数，函数内引用这个dom节点，把这个函数作为**相应指令对应观察者**的回调函数，并在这个函数内部进行dom的更新（由于已经有dom的引用）,就可以完成最关键的一步：数据到ui的更新
对于`ob.a.b.c`的赋值，可以有以下过程，对于get，反复执行getter,且更新指针，对于setter,取到`最后一个.后`,再执行真正的赋值。
```js
	getVmVal(vm, exp) {
		var val = vm;
		exp = exp.split('.');
		exp.forEach(function(k) {
      //反复更新指针，反复执行getter，不执行setter
			val = val[k];
		});
		return val;
	}
	setVmVal(vm, exp, newValue) {
		var val = vm;
		exp = exp.split('.');
		exp.forEach(function(k, i) {
			// 非最后一个key，更新val的值
			if (i < exp.length - 1) {
        //反复执行getter
				val = val[k];
			} else {
        //最后再执行setter
				val[k] = newValue;
			}
		});
	}
```
还有一个是`mvvm`对象的设置，对于以下代码，我们mvvm中的model部分是data属性字段，vm作为承接v和m的部分，也起到一定逻辑处理的角色
```js
 var vm = new MVVM({
            ele: document.getElementById('mvvm-app'),
            data: {
                someStr: 'hello ',
                className: 'btn',
                htmlStr: '<span style="color: #f00;">red</span>',
                child: {
                    htmlStr: 'few',
                    someStr: 'World !'
                }
            },

            computed: {
                getHelloWord: function() {

                    return this.someStr + this.child.someStr;
                }
            },

            methods: {
                clickBtn: function(e) {
                    var randomStrArr = ['childOne', 'childTwo', 'childThree'];
                    this.child.someStr = randomStrArr[parseInt(Math.random() * 3)];
                    this.child.htmlStr = randomStrArr[parseInt(Math.random() * 3)];
                }
            }
        });
```
MVVM通过一定的代理手段，将data字段映射到vm实例之上，目的是在`methods,computed`函数中，通过this(vm实例)能访问`model`的值,对于`computed`，用defineProperty，将`computed`中的所有字段也映射到vm实例中，最后有个非常重要的点:
```js
	//命令数据为可以观察的
	new Observebal(data);
```
需要对model部分的数据进行观察，这样，通过代理`model`的`key`到实例的`key`后，每次`vm.xx`都会进行getter,每次`vm.xx = xxx`都会进行setter，要注意`defineProperty`的getter,和setter都需要用引用，以防止循环调用
最终代码如下：
<script async src="//en.jsrun.net/IYiKp/embed/all/light/"></script>



感谢文章，部分代码参考以下文章：
> https://github.com/DMQ/mvvm
> https://zhuanlan.zhihu.com/p/24475845