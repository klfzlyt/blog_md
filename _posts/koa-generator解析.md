---
title: koa-generator解析
date: 2016-08-28 21:20:52
tags: node
---
# 目录
* [引言](#intro)
* [解析](#analyse)
* [讨论](#end)



<div id="intro"></div>
### 1.引言
* * * 
在koa中，存在着大量的中间件，koa并不绑定任何中间件，许多功能都要靠中间件来进行实现， @dp/node-server基于koa，koa-router作为koa的一个基础中间件，在很多web应用都有用到。在node-server中用到的==action==本质也是用到了koa-router，因此koa-router与我们的关系还是很紧密的，理解其原理对于我们写action有一定帮助。
#### 1.1 回顾
##### co+generator
* [koa](http://koa.bootcss.com/)通过co来组合不同的generator,使得我们的中间件可以串行化执行，并提供downstrem，upstream回溯的能力。
* co这个库，将我们的异步操作都串行化，koa中间件执行核心，即是先把所有generator fuction反复包装，然后调用co.wrap同步化
* 在**图1**中，橙色的箭头表示`yield`一个generator function或者是`yield`一个generator的点，通过co，我们得以递归的执行generator或者generator function。虚线的箭头表示控制权的交出与交回，绿色表示控制权在不在本函数内部，一个请求的经过所有中间件的时间等于图1中最下面的长条的时间。
在最后一个中间件之后不存在我们注册的中间件，yield会将控制权交给一个`noop`空的generator function。如果之前还有yield非generator的点，co会将相关array,object,thunkfunction转为promise,统一对promise进行处理，并等待promise的resolve，期间控制权在node中，直到resolve之后控制权才交回我们的函数。
![第一部分](http://7xs6k5.com1.z0.glb.clouddn.com/image/4/f6/18f563a1ceb27e642901777339de6.png)
<center>图1·co+generator 过程</center>

###### 相关代码片段
每app.use一次加入一个中间件generator function到`this.middleware`这个数组中
```js
app.use = function(fn){
  if (!this.experimental) {
    // es7 async functions are not allowed,
    // so we have to make sure that `fn` is a generator function
    assert(fn && 'GeneratorFunction' == fn.constructor.name, 'app.use() requires a generator function');
  }
  debug('use %s', fn._name || fn.name || '-');
  this.middleware.push(fn);///////////加入一个中间件
  return this;
};
```


反复包装的过程，不断的将得到的generator作为参赛传递到==下一个 generator==中:
```javascript
function compose(middleware){
  return function *(next){
    if (!next) next = noop();

    var i = middleware.length;

    while (i--) {
      next = middleware[i].call(this, next);//反复包装
    }

    return yield *next;
  }
}
```
<!--more-->
```js
app.callback = function(){
  var fn = this.experimental
    ? compose_es7(this.middleware)
    : co.wrap(compose(this.middleware));
  var self = this;
  if (!this.listeners('error').length) this.on('error', this.onerror);
  return function(req, res){
    res.statusCode = 404;
    var ctx = self.createContext(req, res);
    onFinished(res, ctx.onerror);
    fn.call(ctx).then(function () {
      respond.call(ctx);//通过co执行中间件,执行完成在respond中处理this.body
    }).catch(ctx.onerror);
  }
};
```
<div id="analyse"></div>
### 2.koa-router解析
* * *
中间件一般都分为注册和执行2个阶段，一般我们只用关心注册的阶段，在执行阶段，执行的控制流完全交给了co，我们只提供了执行的实现，执行的时机由co把控。

##### 2.2 koa-router用法

1. `router.get('/', function *(next) {…})` HTTP动词注册
1. `router.use('/', function *(next) {…})` 路径匹配注册
1. `router.register('/',['GET','POST'],function *(next){…}) ` 多HTTP方法注册
1. `router.param('param',function *(next){…}) ` 命名参数注册
对于1，2两种注册方法，其本质是调用了3这个注册方法，对于4注册方法，是与其它注册方法不同的另外一种注册方法。

##### 2.3 koa-router数据结构
图2展示了koa-router 的内部数据结构，`Router`作为我们直接使用的对象，暴露register等方法，图2中`Router`对象的stack属性作为一个集合保存了多个`Layer`,每个`Layer`保存了本层的注册路径，并引用了多个中间件generator function，在我们的应用中，即为`Action`。在图3中，展示了一个`Router`拥有4个`Layer`的情况。
![数据结构](http://7xs6k5.com1.z0.glb.clouddn.com/image/4/69/5573520e19c93ca1cfc3e263cbcc0.png)
<center>图2·koa-router中间件类图</center>
![数据结构](http://7xs6k5.com1.z0.glb.clouddn.com/image/3/62/e0998228ea90be503b8fa4bdb6d79.png)
<center>图3·Router进行了4次注册的情况</center>

##### 2.4 koa-router注册中间件流程

一开始的GET,POST,PUT等HTTP动词都是调用了register方法，包括@dp/node-server里面的注册，也是直接调用了register进行路由注册,以register为例，图4中用户一开始调用register,会在Router中创建一个`Layer`对象，之后再使用`layer`的param进行命名参数注册，最后将得到的`layer`加入队列。
![](http://7xs6k5.com1.z0.glb.clouddn.com/image/d/d5/1b84aacb28ba199f03c3dbe0593dc.png)
<center>图4·koa-router注册顺序图</center>

###### koa-router注册中间件相关片段
```js
Router.prototype.register = function (path, methods, middleware, opts) {
  opts = opts || {};
  var stack = this.stack;
  //-------------> create route
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    name: opts.name,
    sensitive: opts.sensitive || this.opts.sensitive || false,
    strict: opts.strict || this.opts.strict || false,
    prefix: opts.prefix || this.opts.prefix || "",
  });
  if (this.opts.prefix) {
    route.setPrefix(this.opts.prefix);
  }
  //-------------> add parameter middleware
  Object.keys(this.params).forEach(function (param) {
    route.param(param, this.params[param]);
  }, this);
     /*
     ……………………
    */
  //-------------> 加入stack中，增加了一个Layer
    stack.some(function (m, i) {
      if (!m.methods.length && i === stack.length - 1) {
        return stack.push(route);
      } else if (m.methods.length) {
        if (stack[i - 1]) {
          return stack.splice(i, 0, route);
        } else {
          return stack.unshift(route);
        }   
    });
  }
  return route;
};
```
###### 2.4.1 koa-router注册过程分解
1.`router.register('/dp/ac4',['GET','POST'],function *(next){…})`
在这一步，我们注册'/dp/ac4'这个路径，注册get,post方法，router的数据结构会如图5红色部分所示变化。
![register1](http://7xs6k5.com1.z0.glb.clouddn.com/image/e/0a/8aba59b6f76d6fe15a9ed11b056b9.png)
<center>图5·注册一个非命名参数情形</center>
* * *

2.`router.param('user',function *(next){…}) `
在这一步注册了一个命名参数中间件，这时koa-router会在所有layer中检查是否有对应命名参数的`layer`已经注册，如图6，在第三个layer中，注册了`user`命名参数，其会新增一个action的引用,指向新增加的命名参数中间件，即为图中的红色部分。在执行阶段，如果路径匹配，这个action就会得到执行。
![register2](http://7xs6k5.com1.z0.glb.clouddn.com/image/9/ce/b4e986b19bafc6c7eb83f223d6d4e.png)
<center>图6·注册一个命名参数的情形</center>
* * *

3.`router.register('/dp/:user',['GET','POST'],function *(next){…}) `
在这一步注册了一个带命名参数和路径的`Layer`，这时koa-router会在所有命名参数中检查是否有对应命名参数中间件已经注册，如图7中的==user==命名参数，如果已经有过注册，即会执行红色部分的注册过程，将引用指向已经注册过的命名参数中间件。
![register3](http://7xs6k5.com1.z0.glb.clouddn.com/image/d/08/b458da8100312b1695fcaf4ad0265.png)
<center>图7·注册一个带命名参数和路径的情形</center>
* * *

##### 2.5 koa-router中间件执行流程
在注册完成以后，每来一个请求，就会进行中间件的执行，在请求到达我们的中间件之前，koa-router会依赖Path-to-RegExp进行路由路径的正则匹配，如图8所示，我们之前注册的路径通过红色部分的Path-to-RegExp组件将会转换为相应的正则表达式保存在各个`Layer`中，之后对于请求的路径，即会进行如图8所示的正则匹配，并获得相应的匹配结果。
![action2](http://7xs6k5.com1.z0.glb.clouddn.com/image/f/0d/a10f4e50c379831af9ddf6263b7dd.png)
<center>图8·中间件执行前的处理</center>
在请求实际得到处理之前，会将所有符合请求路径与方法都匹配的路径筛选出，并进行中间件包装，包装过程与app.callback中的包装过程类似，每次传递generator到下一个中间件中，当所有`Layer`包装完成后，koa-router会交出控制权，由co来执行中间件函数。
![action1](http://7xs6k5.com1.z0.glb.clouddn.com/image/f/39/71873f28d3fd8e1fd7a0668fe8388.png)
<center>图9·中间件执行前的处理</center>
图10中，如果HTTP的请求路径为'dp/ac1'，koa-router会将所有layer的路径进行正则匹配，如图，第一，二个匹配成功，在匹配完成之后，进行中间的包装过程，这个过程还会产生命名参数，包装过程从最后一个layer的最后一个action开始包装，依次传递generator，最后在包装完第一个layer的第一个action后得到一个generator，之后便可以将这个generator交于co来控制执行。
![action](http://7xs6k5.com1.z0.glb.clouddn.com/image/6/91/08eec4eb320a7c877246f51563d21.png)
<center>图10·中间件执行前的包装过程</center>
图11中，展示了有2个action（图中红色）匹配路径与方法的情况，在最下面的执行过程的白色部分，koa-router进行了中间件的包装，在包装完成之后，即橙色部分yield交出控制权，控制权交到了第一个action中，第一个action在执行完成或者执行中途，同样yield next在橙色部分将控制权交出到下一个action中，下一个action同样yield next将控制权交于相对于koa-router下一个中间件，之后控制权返回可以执行action剩余的动作。
![action3](http://7xs6k5.com1.z0.glb.clouddn.com/image/6/63/48509a66927aa7094fe02657a9425.png)
<center>图11·注册中间件执行过程</center>
###### 相关代码片段
每次执行koa-router中间件的过程即是执行dispatch这个generator function的过程
```js
  var dispatch = function *dispatch(next) {
    var path = router.opts.routerPath || this.routerPath || this.path;//获得请求路径
    var matched = router.match(path, this.method);
//对本router中的layer进行匹配，得到匹配结果
    var layer, i, ii;
//加入上下文matched属性
    if (this.matched) {
      this.matched.push.apply(this.matched, matched.path);
    } else {
      this.matched = matched.path;
    }
//pathAndMethod为路径和HTTP方法都匹配的所有layer
    if (matched.pathAndMethod.length) {
      i = matched.pathAndMethod.length;
      var mostSpecificPath = matched.pathAndMethod[matched.pathAndMethod.length - 1].path
      this._matchedRoute = mostSpecificPath

      while (matched.route && i--) {
        layer = matched.pathAndMethod[i];//从最后一个layer开始压缩
        ii = layer.stack.length;
        this.captures = layer.captures(path, this.captures);//构造上下文captures属性
        this.params = layer.params(path, this.captures, this.params);//构造上下文命名参数
        while (ii--) {
//对当前layer中所有的中间件进行压缩，从最后一个开始，压缩过程类似app.callback中compose的压缩过程，即不断的将generator传递
          if (layer.stack[ii].constructor.name === 'GeneratorFunction') {
            next = layer.stack[ii].call(this, next);
          } else {
            next = Promise.resolve(layer.stack[ii].call(this, next));
          }
        }
      }
    }
//交出控制权
    if (typeof next.next === 'function') {
      yield *next;
    } else {
      yield next;
    }
  };
```

<div id="end"></div>
### 3.讨论
我们的项目中的action，通常是一个generator function，==有一些接口没有写yield next==，由于我们的koa-router在@dp/node-server中是最后一个注册的中间件，所以不进行yield next影响也不大，但个人建议还是写yield next，如果今后在路由之后有进行this.body的处理，即koa-router不是最后一个中间件的情况，如果不写yield next，之后的中间件将会得不到执行；对于现在koa-router是最后一个中间件，即便写了yield next，控制流也会进入到一个内容为空的==noop==函数中，之后立即返回，对于执行逻辑不造成影响，所以建议还是在每一个action中写yield next~~。

参考资料：<br/>
co: [https://github.com/tj/co](https://github.com/tj/co)
koa: [http://koa.bootcss.com/](http://koa.bootcss.com/)
koa-router: [https://github.com/alexmingoia/koa-router](https://github.com/alexmingoia/koa-router)
path-to-regexp: [https://github.com/pillarjs/path-to-regexp](https://github.com/pillarjs/path-to-regexp)


* * *
 
![end](http://7xs6k5.com1.z0.glb.clouddn.com/image/2/17/8ce11349638426ad32372f506bbc3.png)