---
title: redux源码分析及自己实现
date: 2017-06-19 21:20:52
tags: 前端
---

 
在介绍redux源码前，先来提几个问题：
1. dispatch做了什么 
1. 订阅状态改变是怎么实现的？
1. 状态树是怎样产生的？
1. dispatch出去的action是怎么过中间件的？
1. 中间是怎么串起来的，执行顺序是如何的？
    * 为什么源码中不用for来设计中间件?
    * 中间件的设计是否对应了哪种设计模式？        
1. 中间件compose为什么要从后往前包装，得到的新dispatch为什么又是从前到后的？
1. 假如2个store,哪些部分是独立的？哪些部分是可复用的？


### 源码赏析
 
先回答第一第二个问题
1. dispatch做了什么 
1. 订阅状态改变是怎么实现的？

**dispatch只做了两件很微小的工作:**
> 1. **根据之前state生成新state（执行reducer/一般为rootreducer）**
> 1. **触发订阅函数**

<!--more-->
如下：
```javascript
	function dispatch(action) {
		currentState = currentReducer(currentState, action);
		for (let i = 0; i < currentListeners.length; i++) {
			currentListeners[i]();
		}
		return action;
	}

	function subscribe(listener) {
		let index = 0;
		for (let i = 0; i < currentListeners.length; i++) {
			//如果有不做操作
			if (listener == currentListeners[i]) {
				index = i;
				return;
			}
		}
		index = currentListeners.push(listener) - 1;
		return function unsubscribe() {
			currentListeners.splice(index, 1);
		};
	}
```

再来看一个问题：
1. 状态树是怎样产生的？
![状态树](http://p1.meituan.net/dpgroup/f682793c7a074d11e1e211f9d7abb241477890.jpg)
我们在使用redux的时候，要产生状态树，需要用到redux中的combineReducer函数，那么可以先想一下，这个函数接受一个对象，返回一个新函数，作为一个新reducer存在，这个新reducer的状态应该是啥？从库的角度来说，它不可能有状态，所以状态只跟我们传入的参数有关，我们传入了多个reducer,怎么合成这些reducer?所以用个对象hashMap就好,每个key对应一个子reducer，这个combineReducer只负责承接状态的转手合成，状态的生成都是reducer得到。

```javascript
const reducer = combineReducers({
  a: doSomethingWithA,
  b: processB,
  c: c
})

// 等同于
function reducer(state = {}, action) {
  return {
    a: doSomethingWithA(state.a, action),
    b: processB(state.b, action),
    c: c(state.c, action)
  }
}
```
总之，combineReducers()做的就是产生一个整体的 Reducer 函数。该函数根据 State 的 key 去执行相应的子 Reducer，并将返回结果合并成一个大的 State 对象。
简单实现如下：
```javascript
export default function combineReducers(reducers){
        let keysOfreducers = Object.keys(reducers);
        return function newReducer(state={},action){
            let newState= {}
            for(let i=0;i<keysOfreducers.length;i++){
                let key = keysOfreducers[i];
                //可以看到我们传入进去的是父state的各个key，从而子reducer不需要关心父reducer的存在
                let singleReducerState = state[key];
                newState[key] = reducers[key](singleReducerState,action);
            }
            return newState;
        }
}
```


### **中间件部分（重点）**


中间作为是作为源码中比较难读的一部分，涉及到一些函数式的编程都在里面，中间件也是redux相当强大的部分，下面详细说下：
从中间件的签名说起：
第一眼看到这种中间件结构，其实是奔溃的。。
![](//p1.meituan.net/dpgroup/6256c64a2ba5f820c58970f426ab8b678590.jpg)
```javascript
const xxxxMiddleware = ({getState,dispatch})=>next=>action=> {
}

function xxxxMiddleware({getState,dispatch}) {
    return function (next) {
        return function (action) {...}
    }
}

```
那么能不能改成这样的呢?

```javascript
function xxxxMiddleware({getState,dispatch},next,action){
  
}
```
或者这样？
```javascript
function xxxxMiddleware({getState,dispatch})=>(next,action)=>{
  
}
```
答案是当然可以,那么我们来改造一下源码:
(加入了curry化 和 compose的实现)
```javascript
import R from 'ramda';
/*
    ---------------------util---------------------
*/
var curryMy = function(fn) {
	var args = [];
	for (var i = 1; i < arguments.length; i++) {
		args.push(arguments[i]);
	}
	return function() {
		for (var i = 0; i < arguments.length; i++) {
			args.push(arguments[i]);
		}
		if (args.length >= fn.length) {
			return fn.apply(window, args);
		} else {
			return curryMy.apply(window, [fn].concat(args));
		}
	};
};

function compose(curryedMiddwares, func) {
	let pre = func;
	for (let i = 0; i < curryedMiddwares.length; i++) {
		let curryedMiddware = curryedMiddwares[curryedMiddwares.length - 1 - i];
		//不断把后一个的结果保存到前一个中
		pre = curryedMiddware(pre);
	}
	return pre;
}

/*
-----------------applyMiddleware----------------------------------
*/
export default function applyMiddleware(...middlewares) {
	return creatStore => (reducer, preloadedState, enhancer) => {
		let store = creatStore(reducer, preloadedState, enhancer);
		if (middlewares[0].length == 3) {
			/*	
				对应以下情况：
				function ({getState,dispatch},next,action){
				
				}
			*/
			middlewares = middlewares.map(middleware => {
				return R.curry(middleware);
			});
		}

		let injectedMiddlewares = middlewares.map(middleware => {
			return middleware({
				getState,
				dispatch: action => store.dispatch(action),
			});
		});
		let curryedMiddwares = injectedMiddlewares;
		if (injectedMiddlewares[0].length > 1) {
			/*	
				对应以下情况：
				function ({getState,dispatch})=>(next,action)=>{
				
				}
			*/
			curryedMiddwares = injectedMiddlewares.map(middware => {
				//这里也可以用R.curry
				return curryMy(middware);
			});
		}
		//包装dispatch,使得dispatch一个action的时候，能过中间件
		//R.compose || compose 任选一个
		//R.compose(...curryedMiddwares)(dispatch)
		let dispatch = R.curry(compose)(curryedMiddwares)(store.dispatch);
		return {
			...store,
			dispatch,
		};
	};
}
```

我们自己实现了一遍`applyMiddleware`
可以看到我们用到了2个函数式编程中非常重要的概念：
1. **curry化**
1. **compose组合**

#### 简单介绍下这两者
#### 先说柯里化：
![](http://p1.meituan.net/dpgroup/e580afeb9df87c899854f6f10bc50e38606707.png)


![](http://p0.meituan.net/dpgroup/f3c8184e59a4ebefec5fe923f77297d3119612.jpg)

有`n个参数的单个函数`转换成有`单个参数的n个函数`



简单例子：
```js
	add(x,y){
		return x+y
	}
	let curry_add = R.curry(add)

	/*
		调用
	*/
	let add20 =  curry_add(20);
	add20(30) //return 50

	curry_add(10)(10) //return 20

```


先转换一下思路，function作为Function的实例，可以向变量一样到处传递的，且function作为变量有一个非常重要的功能：`存储`
```javascript
	let add20 =  curry_add(20);
	add20(30) //return 50
```
可以看到我们把20存储了下来，之后的函数保存的有20这个值了


curry化的意义:只传递给函数一部分参数来调用他，让他返回一个函数去处理剩下的参数，一句话： **`保存参数，延迟执行，局部调用`**
> ps:_自己实现`R.curry`可以用递归去做_

#### 再说组合
![](http://p1.meituan.net/dpgroup/541ee8ea83621400e5839c133d6cbf2b98559.jpg)
```javascript
let g= (str) => parseInt(str)
let f= (num) => !!num

f(g('0')) //false

let composed_f_g = R.compose(f,g)
composed_f_g('0') //false
```
在大致了解了这两个概念后：来看我们的`applyMiddleware`
由于柯里化函数具有延迟执行的特性，通过不断柯里化形成的 middleware 可以累积参数，
所以不管我们的中间件函数签名如何，都要进行科里化，目的是存储变量（即为存储next函数，下一个中间件的执行函数），延迟执行。

看一个例子(已curry化)：
```javascript
//A
function A({getState,dispatch}){
	return function A(next) {
		return function A(action) {
			console.log('A before next')
			next(action);
			console.log('A after next')
			return 'A';
		}
	}
}

//B
function B({getState,dispatch}){
	return function B(next) {
		return function B(action) {
			console.log('B before next')
			next(action);
			console.log('B after next')
			return 'B';
		}
	}
}
//C
function C({getState,dispatch}){
	return function C(next) {
		return function C(action) {
			console.log('C before next')
			next(action);
			console.log('C after next')
			return 'C';
		}
	}
}

//串联中间件
applyMiddleware(A,B,C)

```
看看源码
```javascript
   var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
	//注入api
    chain = middlewares.map(middleware => middleware(middlewareAPI))

	//合成
    dispatch = compose(...chain)(store.dispatch)
```
一开始把`middlewareAPI`注入，得到一个部分执行的函数，保存了api(体现了`存储`的功能)
```javascript
function B(next) {
    return function B(action) {
        console.log('B before next')
        next(action);
       	console.log('B after next')
        return 'B';
    }
}
```
再来看关键的一步
```javascript
dispatch = compose(...chain)(store.dispatch)
```


```javascript
dispatch = compose(A,B,C)(store.dispatch) 
等价于
dispatch = A(B(C(store.dispatch)))
```
下面我们来暴力展开下
```javascript
		C(store.dispatch)
			//得到
		function C(action) {
			console.log('C before next')
			/*
					替换next
			*/	
			store.dispatch(action);
			/*
					替换next
			*/	
			console.log('C after next')
			return 'C';
		}
```
```javascript
		B(C(store.dispatch))
		//得到
 		function B(action) {
			console.log('B before next')
				/*
					替换next
				*/			
				console.log('C before next')
				store.dispatch(action);
				console.log('C after next')
				//C的返回 这不是真的返回
				return 'C';
				/*
					替换next
				*/
			console.log('B after next')
			return 'B';
		 }
```
```javascript
		//再执行
		dipatch = A(B(C(store.dispatch)))	
		//得到新的dispatch
		dipatch = function A(action) {
			console.log('A before next')
				/*
					再次替换next
				*/	
				console.log('B before next')
					/*
						替换next
					*/			
						console.log('C before next')
						//原来的dispatch在这里
						store.dispatch(action);
						console.log('C after next')
						//C的返回 这不是真的返回
						return 'C';
					/*
						替换next
					*/
				console.log('B after next')
				//B的返回 这不是真的返回
				return 'B';
				/*
					再次替换next
				*/	
			console.log('A after next')
			return 'A';
		}

```
所以我们可以抽象出来中间件是怎么执行的：如下：
![](http://p1.meituan.net/dpgroup/dd3aef4a5486560cdd00b577211beb44342595.jpg)
看到这里好像发现了什么，这不是跟koa的co实现类似吗？
![](http://7xs6k5.com1.z0.glb.clouddn.com/image/4/f6/18f563a1ceb27e642901777339de6.png)
可以回答：
**一模一样**。
所以网上有一种说法，叫做洋葱模型（onion model）
> compose的实现有很多，不止一种
 
看看之前的koa的compose: 

```javascript
function compose(middleware){
  return function *(next){
    if (!next) next = noop();

    var i = middleware.length;

    while (i--) {
      next = middleware[i].call(this, next);
    }

    return yield *next;
  }
}
```
以及我们的compose:

```javascript
function compose(curryedMiddwares, func) {
	let pre = func;
	for (let i = 0; i < curryedMiddwares.length; i++) {
		let curryedMiddware = curryedMiddwares[curryedMiddwares.length - 1 - i];
		//不断把后一个的结果保存到前一个中
		pre = curryedMiddware(pre);
	}
	return pre;
}
```
我们的compose好像接受了额外参数？没关系，我们科里化它就好
`R.curry(compose)`

最后是redux自身的compose:
```javascript
	export default function compose(...funcs) {
		if (funcs.length === 0) {
			return arg => arg
		}

		if (funcs.length === 1) {
			return funcs[0]
		}

		const last = funcs[funcs.length - 1]
		const rest = funcs.slice(0, -1)
		//把从最后一个开始计算的结果依次往前传递
		return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
	}
```
那么源码
```javascript
 dispatch = compose(...chain)(store.dispatch)
```
就很清楚了，如下图，我们包装了老的`dispatch`,得到了一个新的`dispatch`,调用这个新的`dispatch`，就顺次执行了中间件中的函数，对开发来说这个新的`dispatch`是透明的
![fe](http://p0.meituan.net/dpgroup/4b5454e3619e5ace557c889a13b1cb7324199.png)



> 值得一提的是redux的applyMiddleware叫做 
([higher-order store](https://github.com/reactjs/redux/blob/cdaa3e81ffdf49e25ce39eeed37affc8f0c590f7/docs/higher-order-stores.md)) 
其他的 higher-order store还有devTools等，看下面一个例子：
```javascript
const newCreateStore = compose(
  applyMiddleware(m1, m2, m3),
  devTools,
  createStore
);
```
我们通过`applyMiddleware`,`devTools`把原有createStore的能力提升了




  ### 注意点：

小心在中间件中`dispatch(action)`
如下图，可能有死循环
![fds](http://p1.meituan.net/dpgroup/7211a167e08e9a4652024ca5657cca2652797.png) 

设计不好一不小心就爆栈，假如在中间件中dispatch(action)，这个action又重头开始依次经过中间件，遇到这个中间件，又重头开始，形成死循环。
![](http://p0.meituan.net/dpgroup/6bb2b292fc3ecf18d8dfc53c0ee35751312809.jpg)
所以一般的建议是：
* 最好清楚dispatch action的路径
* 像thunk-middleware一样注入dispatch给业务，中间件不关心dispatch

  
 
### 几个问题： 
源码中
```javascript
var middlewareAPI = {
     	 getState: store.getState,
       	dispatch: (action) => dispatch(action)
    }
```	
为什么不写成：
```javascript
var middlewareAPI = {
     	 getState: store.getState,
      	dispatch: dispatch
    }
```	
为什么要用函数包一下？
只有包装过，才会永远等于store.dispatch,来看一张图，如果不包装的话,我们在进行compose之前，都是得到的旧dispatch,传入middware后，middware中的dispatch将一直指向那边老的内存区域，而新的dispatch得执行了compose之后才能得到，如果用闭包，传入的dipatch将一直引用最新的store.dispatch
![](//p1.meituan.net/dpgroup/bb911adcd680f23c66de07f503175839129845.jpg)
* 中间是怎么串起来的，执行顺序是如何的？
利用组合，从左到右执行
* 为什么源码中不用for来设计中间件?
1.不能回溯，在中间交出控制权
2.中间件没有控制权（但是控制能力可以用类似于事件冒泡那样去实现），不能进行向下走的控制,因而没有异步能力
        
* 中间件的设计是否对应了哪种设计模式？ 
1.职责链

* 中间件compose为什么要从后往前包装？

从后往前包装是为了保存next函数，不从后往前的话，当前中间件的next函数无法确立，因为next函数就是后一个中间件的执行代码

* 得到的新dispatch为什么又是从前到后的？ 
因为最后一个包装的在最外层，因而可以先得到执行，从而得到从前到后的效果。

* 假如2个store,哪些部分是独立的？哪些部分是可复用的？
除了中间件可以复用，其他都是独立的。

 http://cn.redux.js.org/docs/faq/Performance.html

 http://redux.js.org/docs/recipes/ImplementingUndoHistory.html