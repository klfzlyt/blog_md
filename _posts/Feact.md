---
title: React简单实现之Feact---探寻React内部机制
date: 2017-08-14 17:20:52
tags: React
---

下面的文章介绍了React的内部机制，本文的一部分参考了文章中的实现。

[嫌麻烦可以点开跳到最后实现部分](#jumpend)

> 原文地址：
 http://www.mattgreer.org/articles/react-internals-part-three-basic-updating/

在读过上述文章后，对react的机制有了个大致的了解，感谢作者，但是其实现有一些局限在于没有实现creatElement接受多个children的情形，只能接受单个字符串文本，因而少了diff,patch这一环节,且缺少了事件系统，在发邮件咨询了作者后，下面的代码基于其实现进行了扩展。
此外在实现简易react的过程中还参考了react 16的源码，以及下面的文章：
> * [react 官方实现介绍](https://facebook.github.io/react/contributing/implementation-notes.html)
> * https://purplebamboo.github.io/2015/09/15/reactjs_source_analyze_part_two//
> * https://zhuanlan.zhihu.com/p/20346379



在实现react过程中，有一些简单术语：
1. 内部实例---指不对用户暴露的virtual Dom实例
我们的虚拟dom对应的具体对象，有`FeactDOMTextComponent`,`FeactDOMComponent`,`FeactCompositeComponentWrapper`,`FeactEmptyComponent`
这些用户无感知，仅在内部使用
2. 外部实例：我们常用的React.Component的实例，有render方法等
3. element---指React.creatElemnt得到的vitual dom

本文基于上述文章思路分几个步骤实现react
一. 初始挂载
二. setState状态变更
三. 事件实现

 
## 一. 初始挂载
初始挂载有一系列问题：
1 React.creatElement创建了什么？
1 ReactDom.render做了什么？
1 初始生命周期是怎么实现的？
2 组件是怎么挂载的？
3 虚拟dom是怎么实现的？
6 怎么挂载一个列表或者是一个有多个children的元素？

先来看一个例子：
<div><script async src="//en.jsrun.net/KSYKp/embed/all/light/"></script></div>
 
首先jsx经过babel转换为React.creatElement

我们先看React.creatElement创建了什么？
```javascript
  createElement(type, props, ...children) {
        const element = {
            type,
            props: props || {},
        };

        if (props && props.key) {
            element.key = props.key;
        }
        if (children) {
            element.props.children = children;
        }
        //返回一个对象 
        //返回一个element，代表虚拟Object
        return element;
    },
```
我们这里都是接受了一个`type`，`props`,`children`,用es6的语法接受children数组,我们后续会根据`type`的不同实例化不同的虚拟dom
<!--more-->
再来看render，render了什么？
```js
  //React.render
    render(element, container) {
        //获得保存过的实例
        const prevComponent = getTopLevelComponentInContainer(container);

        if (prevComponent) {
            //container挂载过
            return updateRootComponent(prevComponent, element);
        } else {
            //render 返回真实dom
            return renderNewRootComponent(element, container);
        }
    },
```
先忽略`updateRootComponent`,在`renderNewRootComponent`这个方法中
```js
function renderNewRootComponent(element, container) {
    //新建dom 挂载在container
    //type是一个function
    const wrapperElement = Feact.createElement(TopLevelWrapper, element);
    //new了 一个 vCompositeNode
    const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);

    const markUp = FeactReconciler.mountComponent(componentInstance, null);
    // container.__feactComponentInstance 挂了 vCompositeNode 的render返回的vCompositeNode
    container.__feactComponentInstance = componentInstance._renderedComponent;
    container.appendChild(markUp);

    return markUp;
}
const TopLevelWrapper = function(props) {
    this.props = props;
};

TopLevelWrapper.prototype.render = function() {
    return this.props;
};
//根据不同的type进行初始化
function instantiateFeactComponent(element) {
    //virtuan node 工厂
    if (element == null || element == undefined) {
        return new FeactEmptyComponent(element);
    } else if (typeof element === 'string') {
        return new FeactDOMTextComponent(element);
    } else if (typeof element.type === 'string') {
        return new FeactDOMComponent(element);
    } else if (typeof element.type === 'function') {
        return new FeactCompositeComponentWrapper(element);
    }
}
const FeactReconciler = {
    mountComponent(internalInstance, parentNode, hostContainerInfo) {
        return internalInstance.mountComponent(parentNode, hostContainerInfo);
    },

    receiveComponent(internalInstance, nextElement) {
        internalInstance.receiveComponent(nextElement);
    },

    performUpdateIfNecessary(internalInstance) {
        internalInstance.performUpdateIfNecessary();
    },
    instantiateChildren(internalInstance, propschildren) {
        internalInstance.instantiateChildren(propschildren);
    },
};
```
我们创建了一个顶层Wrapper,这个顶层Wrapper是个内部自定义组件实例，用这个顶层实例包装了一层我们的render的element，其render就返回自己的props,然后new了一个`FeactCompositeComponentWrapper`内部虚拟dom组件实例（自定义组件对应的虚拟dom的内部实例），再调用`mountComponent`获得对应的dom节点，并挂载在`container`中，可以看到，有个`Reconciler`，在react官方文档中，
`Reconcilers`指不与平台相关联的那部分，由于react的render不止在浏览器中，还可能在服务端，React Native中，参考[react codebase](https://discountry.github.io/react/contributing/codebase-overview.html)，
由Reconcilers执行渲染，生命周期，自定义组件等公共部分。我们简单提供`FeactReconciler`,在虚拟dom方面，我们简单提供4种实例
* `FeactEmptyComponent`(对render里，返回null，undefined进行处理)
* `FeactDOMTextComponent`(对纯文本的虚拟dom)
* `FeactCompositeComponentWrapper`(对自定义组件的虚拟dom)
* `FeactDOMComponent`(对html tag的虚拟dom)

我们根据传入的不同type进行不同的初始化,接下来我们看看`mountComponent`方法，
首先这个方法的意义是：**生成该虚拟dom对应的真实dom**
这个方法中，我们还设置当前虚拟节点的父虚拟节点
> 对于`FeactEmptyComponent`内部组件
我们可以返回一个注释节点，以便让虚拟dom有个真实节点可以对应
```js
 mountComponent(hostParent, hostContainerInfo) {
        this._hostParent = hostParent;
        this._hostContainerInfo = hostContainerInfo;
        this._domNode = document.createComment('feact --- empty --- component');
        return this._domNode;
    }
```
***
> 对于`FeactDOMTextComponent`文本节点，返回一个文本dom对象
```js
mountComponent(hostParent, hostContainerInfo) {
        this._hostParent = hostParent;
        this._hostContainerInfo = hostContainerInfo;
        this._domNode = document.createTextNode(this._currentElement);
        return this._domNode;
    }
```
***
> `FeactCompositeComponentWrapper`是自定义组件的内部实例类，我们在写的时候用**render方法提供单个Element**,注意我们在render不能提供一个list,也不能提供多个平级的html tag。这样的设计使得`FeactCompositeComponentWrapper`有一个好处，就是只用render出来的element只有一个，不会有children的存在，即便有children，也是render的第一个html tag对应的element的children,再如果是自定义组件，那么children即为this.props.children将全权交给组件来处理，这个设计使得我们的自定义组件不需要关心children的问题，在挂载和更新时可以方便的将要挂载的组件或者要更新的组件传递下去。在挂载过程中，我们可以实现相应的生命周期方法：
```js
  mountComponent(hostParent, hostContainerInfo) {
        //保存父虚拟dom实例
        this._hostParent = hostParent;
        this._hostContainerInfo = hostContainerInfo;
        //type是个函数的时候，我们认为是个自定义组件，因而要实例化这个组件（外部组件
            //有render,componentDidMount等生命周期钩子方法
        //）
        const Component = this._currentElement.type;
        //这个type拿到一个函数 这个函数有那些对应的 react 方法
        const componentInstance = new Component(this._currentElement.props);
        //componentInstance 这个实例不是react vnode 的实例 是React.component的实例 因而可以调用render方法
        this._instance = componentInstance;

        //我们还可以保存内部实例到外部实例的映射
        FeactInstanceMap.set(componentInstance, this);

        if (componentInstance.componentWillMount) {
            componentInstance.componentWillMount();
        }

        const markup = this.performInitialMount();

        if (componentInstance.componentDidMount) {
            componentInstance.componentDidMount();
        }

        return markup;
    }
```
`performInitialMount`这个方法获得真正的dom,在这个方法前既可以执行生命周期方法
我们在`performInitialMount`中调用组件的生命周期方法`render`，之后将render返回的element实例化成虚拟dom内部实例对象，在返回这个对象的`mountComponent`方法即可，这个过程就可以递归的下去进行了。
```js
    performInitialMount() {
        //调用自定义组件的render方法，其实就是调用creactElement 出来的
        const renderedElement = this._instance.render();

        //virtual node 工厂得到child 可能会多个不同的实例 公用一组接口
        const child = instantiateFeactComponent(renderedElement);
        this._renderedComponent = child;

        //在这里抹平了mountComponent的过程
        //child.mountComponent 可能是FeactDOMComponent.mountComponent
        //也可能是FeactCompositeComponentWrapper.mountComponent
        return FeactReconciler.mountComponent(child, this);
    }
```
> `FeactDOMComponent`：重要是的`FeactDOMComponent`的挂载，由于一个html tag对应的虚拟dom，可以带多个children，类似
```js
 Feact.createElement('div', {}, 'child 1',null,'child 3'……………………)
```
所以`FeactDOMComponent`的挂载需要考虑children问题，之所以自定义组件不需要考虑，是因为，this.props.children全权交给了组件来处理
`FeactDOMComponent`的虚拟dom的挂载分几个步骤来处理：
1 创建对应的真实dom(document.createElement)
2 设置style等属性
3 创建所有一级children真实dom
4 将第三步得到的所有真实dom挂载到第一步得到的dom中

```js
function flatten(list, depth) {
    depth = typeof depth == 'number' ? depth : Infinity;

    if (!depth) {
        if (Array.isArray(list)) {
            return list.map(function(i) {
                return i;
            });
        }
        return list;
    }

    return _flatten(list, 1);

    function _flatten(list, d) {
        return list.reduce(function(acc, item) {
            //if(d>1)item
            if (Array.isArray(item) && d < depth) {
                return acc.concat(_flatten(item, d + 1));
            } else {
                return acc.concat(item);
            }
        }, []);
    }
}

function flattenChildren(children, init_value) {
    let childrenMap = {};
    init_value = init_value || '';
    for (let i = 0; i < children.length; i++) {
        let child = children[i];
        let name = `.${i}`;
        if (Array.isArray(child)) {
            let flattendObject = flattenChildren(child, `.${i}`);
            Object.assign(childrenMap, flattendObject);
            continue;
        }

        if (child && child._currentElement && child._currentElement.key) {
            //实例
            name = `.$${child._currentElement.key}`;
        }

        if (child && child.key) {
            name = `.$${child.key}`;
        }
        name = init_value + name;
        childrenMap[name] = child;
    }
    return childrenMap;
}
function instantiateFeactComponentArrayItem(ele) {
    if (Array.isArray(ele)) {
        return ele.map((item, index) => {
            return instantiateFeactComponent(item);
        });
    } else return instantiateFeactComponent(ele);
}
 mountChildren(propschildren, el) {
        //得到propschildren 对应的实例 一级children todo
        //   var children = FeactReconciler.instantiateChildren(this, propschildren);
        var mountImages = [];
        var index = 0;
        var children = [];
        let flattenpropschildren = propschildren;
        for (let i = 0; i < flattenpropschildren.length; i++) {
            let child = flattenpropschildren[i];
            let vInstance = instantiateFeactComponentArrayItem(child);
            children.push(vInstance);
        }
        let flattenddChildrenInst = flatten(children);
        for (let j = 0; j < flattenddChildrenInst.length; j++) {
            let vInstance = flattenddChildrenInst[j];
            vInstance._mountIndex = j;
            //对每个children进行挂载，递归下去
            mountImages.push(FeactReconciler.mountComponent(vInstance, this));
        }

        this._renderedChildren = flattenChildren(children);
        return mountImages;
    }
    _createInitialChildren(props, el) {
        //获得所有children 真实 dom
        var mountImages = this.mountChildren(props.children, el);

        for (var i = 0; i < mountImages.length; i++) {
            el.appendChild(mountImages[i]);
        }
    }
    mountComponent(hostParent, hostContainerInfo) {
        this._hostParent = hostParent;
        this._rootNodeID = globalIdCounter++;
        this._domID = hostContainerInfo && hostContainerInfo._idCounter++;

        //宿主Element
        const domElement = document.createElement(this._currentElement.type);

        this._updateDOMProperties(null, this._currentElement.props);
        //挂载
        this._createInitialChildren(this._currentElement.props, domElement);

        FeactInstanceMap.set(domElement, this);
        this._hostNode = domElement;
        return domElement;
    }
```
按照上述情况，我们调用`mountChildren`得到一个children真实dom的数组，然后挂在宿主dom上，获得chilren真实dom的过程是对每一个一级children执行`mountComponent`获得其真实dom,关键是有些chilren是数组的形式:
```js
 Feact.createElement('div', {}, ['child 1',null,'child 3'],'child 4')
```
<div id='jumperdot'></div>
按照react的思路 我们在挂载时候需要扁平化，即为把children数组转变为对象（可以理解为转换为hash），这样方便快速索引，这里有个技巧,我们在key前加`.`把数字变为非数字，**这样`for in`遍历对象的顺序将和声明的顺序一样，也和我们for循环添加的顺序一致**，对于有key的节点，我们以key值作为实例的索引，对于是[]数组的情况，我们递归调用`flattenChildren`，并把调用顺序传递下去，可以得到
之后我们再利用类似lodash的`flatten`方法，把所有children扁平到一个数组中，再依次`mountComponent`得到真实dom,
注意：在`FeactDOMComponent`挂载的时候有个全局自增的id，这个id将作为这个`FeactDOMComponent`的唯一索引。在后续事件中用的上。
```js
  this._rootNodeID = globalIdCounter++;
```
下面的图中有一个数组children，然后有4个不带key的children,对于不带key的，我们用`.`加index的形式使其变为非数字，对于有key字段的内部实例，我们用`$`加key的形式作为key,数组作为第一个元素，其扁平化后的key为2级的形式。
![](http://p0.meituan.net/dpgroup/26d1168b5b44d272ccf6c64be1d42402393679.jpg)

处理完最复杂的`FeactDOMComponent`的mount情形后，我们的mount过程基本结束。

最后查看dom可以得到，也可以打开调试工具查看：
![](https://p1.meituan.net/dpgroup/d0848188b728ee30d378b7dbbdb19feb175331.jpg)
<div></div>

## 二. setState状态变更

在 http://www.mattgreer.org/articles/react-internals-part-three-basic-updating/ 文章中,实现了setState的类似缓存变更功能，但没有对多children的情况做处理，只简单处理字符串，diff的过程也没有体现，模仿react，我们需要定义`receiveComponent`这个方法，这个方法的具体意义是：**更新本组件虚拟dom树下的的真实dom**，也就是说，执行了这个方法之后，这个虚拟dom包括这个虚拟dom之下的所有叶子都得到了更新，更新虚拟dom的主要是更新属性，对于`FeactDOMComponent`,需要更新这个虚拟dom的自身的属性，以及更新其所有的一级child,通过递归依次向下，比如有个&lt;div&gt;&lt;span&gt;&lt;/span&gt;&lt;/div&gt;那么更新`div`的话，其div下的所有children也将得到更新。

根据react官方介绍，我们还需要保存每个虚拟dom的hostNode，即为其对应的真实虚拟dom,`FeactDOMComponent`这样的虚拟dom，很好保存，在mount组件的时候即可以保存，对于合成组件即自定义组件，我们可以获得其render出来元素的hostNode.
```js
getHost() {
        return this._renderedComponent.getHost();
    }
```

我们再原封不动使用React的`shouldUpdateFeactComponent`来判断是否需要更新组件，可以从这个函数中看出，我们更新组件的有几种情况，对于文本，数字，我们都要更新，对于虚拟DOM，我们只要type相等，且key相等就需要更新，这里key相等也有`undefined == undefined`的情形

假如有个父节点，其setState了，那么子节点需要render,那么子节点是怎么知道需要render的呢？先不考虑`setState`，我们知道，在render的时候，会调用creactElement创建新元素，如果有多次render,那么每次都会creactElement创建新元素。事实上，我们在第一次实例化element为内部实例的时候，会把该内部实例保存下来，之后会用每次creactElement得到的新element会与之前的实例中的element进行对比，如果满足`shouldUpdateFeactComponent`更新要求，那么只需执行`receiveComponent`方法，更新组件即可。`FeactCompositeComponentWrapper`的更新如下所示：`FeactCompositeComponentWrapper`需要判断其render方法出来的element需不需要更新，如果需要更新，用之前的实例进行更新（receiveComponent）,如果不满足更新条件，那么按照react的思路，我们要销毁之前的保存的节点（销毁生命周期暂时无体现），并重新挂载新的节点，即先实例化新节点，再挂载到hostNode的父节点上完成转换。
```js
  _updateRenderedComponent() {
        const prevComponentInstance = this._renderedComponent;
        const inst = this._instance;
        const prevRenderedElement = prevComponentInstance._currentElement;
        //接受新props 和 state的时候再次render
        const nextRenderedElement = inst.render();
        //渲染过的_renderedComponent接受新的render得到的element
        //这个地方也会出现分叉，可能是FeactDOMComponent.receiveComponent
        //也可能是FeactCompositeComponentWrapper.receiveComponent
        if (shouldUpdateFeactComponent(prevRenderedElement, nextRenderedElement)) {
            FeactReconciler.receiveComponent(prevComponentInstance, nextRenderedElement);
        } else {
            var oldHostNode = prevComponentInstance.getHost();                      
            var child = instantiateFeactComponent(nextRenderedElement);
            this._renderedComponent = child;
            var nextMarkup = FeactReconciler.mountComponent(child, this);
            let parent = oldHostNode.parentElement;
            parent.innerHTML = '';
            parent.append(nextMarkup);
        }
    }
```
有个问题是setState怎么缓存的？
这个实现上比较简单，先用个数组把所有变动存起来，再在需要更新的时候，依次赋值
```js
  _processPendingState() {
        const inst = this._instance;
        if (!this._pendingPartialState) {
            return inst.state;
        }
        let nextState = inst.state;
        for (let i = 0; i < this._pendingPartialState.length; ++i) {
            nextState = Object.assign({}, nextState, this._pendingPartialState[i]);
        }
        this._pendingPartialState = null;
        return nextState;
    }
```

`FeactDOMComponent`的更新稍微麻烦一些，需要考虑diff,以做出最小改动,在理解diff上，有一个很重要的点，就是`FeactDOMComponent`的更新，只需要考虑一级child节点，diff的比较对象是内部实例，之所以可以`for in`进行遍历还有顺序保证，即新children的从左到右的顺序，并且使用从0自增的`nextIndex`,是由于之前介绍的在key前[加`.`的操作](#jumperdot)，可以保证是for循环的顺序，可用用nextIndex自增，得到的index和for的index一致，更新一个`FeactDOMComponent`的大致流程是：
1. 依次更新该`FeactDOMComponent`下的所有children，该更新的更新，该新挂载的重新挂载，得到新的children内部虚拟dom实例（这个过程会产生递归）
2. 收集新增加的节点（包括替换旧的也算新增），和要删除的节点
3. 做O(n)的diff,这里的这个n,指的是所有一级children的个数，关于[diff文章](https://zhuanlan.zhihu.com/p/20346379)说的很清楚了，有几个问题可以解答下：
`_mountIndex`怎么来的？在初始挂载的时候，就会按照children的`index`次序，为每个child增加`_mountIndex`属性，lastIndex的意义是什么？访问过的旧有节点的最大位置，这个index只会增加。lastPlaceNode是什么？是按顺序得到的新节点的真实dom，所有的新增节点，节点移动，都是在这个`lastPlaceNode`之后，所以也叫`afterNode`，`mountImages`用来保存所有新增节点的真实dom,且所有的新增dom都在第二步进行完成放入数组中了。
4. 根据得到的更新数组，进行patch操作，更新真实dom
> 利用`lastPlaceNode`的好处是，我们可以对于`MOVE_EXISTING`的操作，我们也可以调用原生的`insertBefore`方法，这个方法把原有节点移动到`lastPlaceNode`这个后面，刚好符合要求（按照新的children顺序 我们的新节点，不管是移动的新节点，还是真正新增的节点，都要放在lastPlaceNode之后）


## 三. 事件机制
事件没有实现合成事件和批处理队列,这里只是简单实现一下思路
在挂载阶段和更新阶段，我们都会调用`_updateDOMProperties`函数来更新节点的属性，这里就把`eventHanlder`响应函数加入到全局map中保存了起来
```js
    _updateDOMProperties(lastProps, nextProps) {    
        let eventReg = /^on[\w]*$/;
        for (let propskey in nextProps) {
            if (propskey.match(eventReg)) {
                let eventHanlder = nextProps[propskey];
                ReactEventEmitter.putListener(this._rootNodeID, propskey, eventHanlder);
            }
        }
    }
```
[react的事件机制](https://zhuanlan.zhihu.com/p/27132447)
react的事件冒泡回调数组不是存于全局事件对象中，而是存在合成事件SyntheticEvent对象中，在合成事件产生过程中，有一段开发模块中的Proxy对象，代码如下：
```javascript
//SyntheticEvent ----->   合成事件的构造函数
/** Proxying after everything set on SyntheticEvent
  * to resolve Proxy issue on some WebKit browsers
  * in which some Event properties are set to undefined (GH#10010)
  */
if (__DEV__) {
  if (isProxySupported) {
    /*eslint-disable no-func-assign */
    SyntheticEvent = new Proxy(SyntheticEvent, {
      construct: function(target, args) {
        return this.apply(target, Object.create(target.prototype), args);
      },
      apply: function(constructor, that, args) {
        return new Proxy(constructor.apply(that, args), {
          set: function(target, prop, value) {
            if (
              prop !== 'isPersistent' &&
              !target.constructor.Interface.hasOwnProperty(prop) &&
              shouldBeReleasedProperties.indexOf(prop) === -1
            ) {
              warning(
                didWarnForAddedNewProperty || target.isPersistent(),
                "This synthetic event is reused for performance reasons. If you're " +
                  "seeing this, you're adding a new property in the synthetic event object. " +
                  'The property is never released. See ' +
                  'https://fb.me/react-event-pooling for more information.',
              );
              didWarnForAddedNewProperty = true;
            }
            target[prop] = value;
            return true;
          },
        });
      },
    });
    /*eslint-enable no-func-assign */
  }
}
```

***

要实现一个合成事件系统，我们得需要3样东西，SyntheticEvent，存储映射关系的内部对象的key,以及为了事件冒泡系统的内部实例的_hostParent父亲实例引用，对于_hostParent，
只要在mount的时候把this传入即可，由于我们的事件只能挂在`ReactDomComponent`上，
在合成组件即继承自`React.component`的组件，并不能挂载事件，只能定义事件响应在原型上，所以不用为其增加标识，我们设置一个全局id,在`FeactDOMComponent`mount时这个id自增，保证每一个`FeactDOMComponent`有一个唯一的id，这样我们就能利用id作为key，存储对应的事件响应函数,如图,在一个全局二维数组里保存有所有id对应的相应函数  

***
![img](http://p0.meituan.net/dpgroup/833c2178f2528bd42dfeecff1b36612a73003.jpg)
我们用_hostParent找到所有需要冒泡的虚拟dom,在dispatchEvent的时候简单调用，
> 事实上，react的事件分发是在一个事务中的，所以在这个事务中做出的任何setState操作都会缓存，在事务结束的时候进行统一更新.
```js
var ReactEventEmitter = {
    listenerBank: {},

    putListener: function putListener(id, registrationName, listener) {
        var bankForRegistrationName = this.listenerBank[registrationName] || (this.listenerBank[registrationName] = {});

        bankForRegistrationName[id] = listener;
    },

    getListener: function getListener(id, registrationName) {
        return this.listenerBank[registrationName][id];
    },

    trapBubbledEvent: function trapBubbledEvent(topLevelEventType, element) {
        var eventMap = {
            onClick: 'click',
        };
        var baseEventType = eventMap[topLevelEventType];
        element.addEventListener(baseEventType, this.dispatchEvent.bind(this, topLevelEventType));
    },

    dispatchEvent: function dispatchEvent(eventType, event) {
        //这个是原生事件
        event.preventDefault();
        let internalInst = FeactInstanceMap.get(event.target);
        //    var id = internalInst._rootNodeID;
        //先默认实现一个bubbule的事件系统
        var parentArray = [];
        var listenArray = [];
        var parent = internalInst;
        while (parent) {
            parentArray.push(parent);
            parent = parent._hostParent;
        }
        for (let i = 0; i < parentArray.length; i++) {
            let instID = parentArray[i]._rootNodeID;
            let handler = this.getListener(instID, eventType);
            handler && listenArray.push(handler);
        }
        for (let i = 0; i < listenArray.length; i++) {
            listenArray[i](event);
        }    
    },
};

```
### 最终代码：
<div id="jumpend">
<script async src="//en.jsrun.net/iuYKp/embed/all/light/"></script></div>    
