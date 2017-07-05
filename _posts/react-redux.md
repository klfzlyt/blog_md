---
title: react-redux源码分析
date: 2017-05-28 21:20:52
tags: redux
---
## react-redux(4.4.5)
react-reudx 5.0+的版本已经进行了比较大的重构，这里主要介绍v4的版本
### provider
provider主要做了传递store的事情，通过在parent组件内，使用getChildContext方法，这个方法可以把相应的属性隐式往下传，子组件及子组件的子组件都可以获得此属性

```js
export default class Provider extends Component {
  getChildContext() {
    return { store: this.store }
  }

  constructor(props, context) {
    super(props, context)
    this.store = props.store
  }

  render() {
    return Children.only(this.props.children)
  }
}
```
从Provider的代码中可以看到在构造函数初始化的时候将store赋了值，之后通过getChildContext将值往下传递
还要注意`childContextTypes`需要声明出用到的属性
```js
/*
Index.childContextTypes={
	name: React.PropTypes.string.isRequired
}
*/
Provider.childContextTypes = {
  store: storeShape.isRequired
}

```
<!--more-->
如同在connect中也要声明用到的context,如下
```js
/*
ScrollBar.contextTypes={
    name:React.PropTypes.string
}
*/
 Connect.contextTypes = {
      store: storeShape
    }
```

https://react.bootcss.com/react/docs/higher-order-components.html

connect干了什么？
connect雏形 

```javascript
import React, { Component, PropTypes } from 'react';

function connect(mapStateToProps, mapDispatchToprops, ownerProps, option) {
	return function wrappComponent(wrappedComponent) {
		class Connect extends Component {
			constructor(props) {
				super(props);
			}

			componentWillMount() {}

			componentDidMount() {}

			componentWillReceiveProps(nextProps) {}

			shouldComponentUpdate(nextProps, nextState) {}

			componentWillUpdate(nextProps, nextState) {}

			componentDidUpdate(prevProps, prevState) {}

			componentWillUnmount() {}

			render() {
				return <wrappedComponent></wrappedComponent>;
			}
		}
		Connect.contextTypes = {};
        return Connect
	};
}

```


```js
connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {}
```
首先connect接受4个参数，第一个参数是要把对应的state映射到组件props的函数，第二个是把dipatch过程映射到props上的函数或者对象，第三个参数是把state映射到的props和dispatch映射到的props合成到一个props并最终注入到组件component中的方法，他们都有对应的默认实现，如下
注意在源码中Returning object literals的写法
```js
const defaultMapStateToProps = state => ({}) // eslint-disable-line no-unused-vars
const defaultMapDispatchToProps = dispatch => ({ dispatch })
const defaultMergeProps = (stateProps, dispatchProps, parentProps) => ({
  ...parentProps,
  ...stateProps,
  ...dispatchProps
})
```
mapStateToProps默认返回一个空对象，不做任何state转换，mapDispatchToProps默认注入dispatch方法，mergeProps默认合并state映射到的props，dispatch映射到的props以及自身的props。


```js
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
  const shouldSubscribe = Boolean(mapStateToProps)
  const mapState = mapStateToProps || defaultMapStateToProps
    //dispatch
    let mapDispatch
    if (typeof mapDispatchToProps === 'function') {
        //函数执行，得到一个object注入到组件的props中
        mapDispatch = mapDispatchToProps
    } else if (!mapDispatchToProps) {
        mapDispatch = defaultMapDispatchToProps
    } else {
        //对象传入，包装为一个function，执行完成后得到一个object，这个object就将注入到组件的props中
        mapDispatch = wrapActionCreators(mapDispatchToProps)
    }
    //[mergeProps(stateProps, dispatchProps, ownProps): props] (Function): 官方要求是一个function
  const finalMergeProps = mergeProps || defaultMergeProps
  const { pure = true, withRef = false } = options
  //如果不传第三个参数，checkMergedEquals为false,这个flag将会影响最后的mergedProps的计算，使得每次都要重新render一个新的component
  const checkMergedEquals = pure && finalMergeProps !== defaultMergeProps

  // Helps track hot reloading.
  const version = nextVersion++
```
对不同类型的`mapDispatchToProps`进行包装，包括对象，与函数形式，使得得到的`mapDispatch`是一个`function`


connnect组件作为一个Component的wrapper,或者称为HOC（high order component），其负责处理store状态的变化，并选择渲染组件,在构造时拿到全局的store
```js
  constructor(props, context) {
        super(props, context)
        this.version = version
        this.store = props.store || context.store
        const storeState = this.store.getState()
        this.state = { storeState }
        this.clearCache()
      }
```
先初始connnect的状态为全局状态





有几个问题
> 1.组件如何知道store的变化？
> 2.组件如何处理自身props的变化？
> 3.mapStateToProps是如何注入到props中的呢
> 4.mapDispatchToProps是如何注入到props中的呢

首先connect组件render的时候会去读一些状态
```js
      const  {
          haveOwnPropsChanged,
          hasStoreStateChanged,
          haveStatePropsBeenPrecalculated,
          statePropsPrecalculationError,
          renderedElement
        } = this
```
由于在construct的时候进行了clear
```js
 clearCache() {
        this.dispatchProps = null
        this.stateProps = null
        this.mergedProps = null
        this.haveOwnPropsChanged = true
        this.hasStoreStateChanged = true
        this.haveStatePropsBeenPrecalculated = false
        this.statePropsPrecalculationError = null
        this.renderedElement = null
        this.finalMapDispatchToProps = null
        this.finalMapStateToProps = null
      }
```
读到的值将为true,true,false,null,null


connect中提供了三个重要的对象
`this.stateProps`,`this.dispatchProps`,`this.mergedProps`
`this.stateProps`用来保存`mapStateToProps`的转换结果，这里类似于一个小reducer 
`this.dispatchProps`用来保存`mapDispatchToProps`的转换结果 
`this.mergedProps` 用于最后挂载到component上的props
这三个对象`this.stateProps`,`this.dispatchProps`,`this.mergedProps` 分别于
`updateStatePropsIfNeeded`
`updateDispatchPropsIfNeeded`
`updateMergedPropsIfNeeded`
得到更新
只要这3个函数中的某一个执行一次，对应的mapStateToProps, mapDispatchToProps, mergeProps都会相应执行一次

 

```js

       /*
          步骤： 1.判断是否应该更新this.stateProps this.dispatchProps
                  2.this.stateProps this.dispatchProps是否变更
       */
        //先声明变量默认值
        let shouldUpdateStateProps = true
        let shouldUpdateDispatchProps = true
        //如果之前render过，则计算一下是否需要更新this.stateProps this.dispatchProps
        if (pure && renderedElement) {
          shouldUpdateStateProps = hasStoreStateChanged || (
            haveOwnPropsChanged && this.doStatePropsDependOnOwnProps
          )
          shouldUpdateDispatchProps =
            haveOwnPropsChanged && this.doDispatchPropsDependOnOwnProps
        }

        let haveStatePropsChanged = false
        let haveDispatchPropsChanged = false

        //haveStatePropsBeenPrecalculated在这里是一个保存是否先前计算过的flag
        if (haveStatePropsBeenPrecalculated) {
          haveStatePropsChanged = true
        } else if (shouldUpdateStateProps) {
          haveStatePropsChanged = this.updateStatePropsIfNeeded()
        }
        if (shouldUpdateDispatchProps) {
          haveDispatchPropsChanged = this.updateDispatchPropsIfNeeded()
        }


```
selector 的作用就是为 React Components 构造适合自己需要的状态视图。selector 的引入，降低了 React Component 对 Store/State 数据结构的依赖，利于代码解耦；同时由于 selector 的实现完全是自定义函数，因此也有足够的灵活性（例如对原始状态数据进行过滤、汇总等）
![](http://p0.meituan.net/dpgroup/70611dd926bce97832856d737a421a3230554.png)
一个简化的render流程，判断this.stateProps，this.dispatchProps是否应该更新的主要逻辑是经过mapStateToProps select后的props有变更过或者是当componentWillReceiveProps判断自身props变更后（浅比较），如果mapDispatchToProps的入参有2个，也将判断this.stateProps应该更新。同理
this.dispatchProps的更新也是依赖于自身props变更后,且mapDispatchToProps的入参有2个。


从初始路径开始，看看当store变化后connent如何处理,首先看到组件在初始的时候就使用store注册了store的变更回调，这个回调会在dispatch任何type的情况下得到执行，所以初始时需要判断之前的store状态和当前的store是否一致，
```js
  if (pure && prevStoreState === storeState) {
          return
        }

```
1.因为有些情况下dispatch出来的一个action不会引起reducer结果的任何变化，
之后如果`mapStateToProps`的入参只有一个，会进行一次select操作，如果selector得到的结果没有任何变化（浅拷贝）,则直接返回，这个时候效率是最高的，
没有组件的rerender(表现在connnect组件的setState),相当于组件不知道store改变了，react组件保持原状

2.如果mapStateToProps得到的结果有变化（浅拷贝），handleChange会做几个动作，一个是置2个标志位为true,
```js
          this.haveStatePropsBeenPrecalculated = true
          this.hasStoreStateChanged = true
```


`this.haveStatePropsBeenPrecalculated`用来表达updateStatePropsIfNeeded已经执行过，即mapStateToProps进行过select，
`this.hasStoreStateChanged`用来表达store的state已经变化，需要重新计算`this.stateProps`
这个状态位在`shouldComponentUpdate`，以及判断`this.stateProps`是否更新时用的上 
另一个重要的动作是：
  `this.setState({ storeState })` 
  这个时候将会触发
  ```js
  shouldComponentUpdate() {
        return !pure || this.haveOwnPropsChanged || this.hasStoreStateChanged
      }
  ```
  由于this.hasStoreStateChanged已经被置位，所以会进行render的操作
每次进入render都会进行一系列的判断
值得注意的是，如果在之前检测到store变化后，mapStateToProps得到新的结果，`updateStatePropsIfNeeded`执行过，
 则`this.haveStatePropsBeenPrecalculated = true`
那么下面的代码就进入第一个分支，`updateStatePropsIfNeeded`不需要再重新计算了。
```js
        if (haveStatePropsBeenPrecalculated) {
          haveStatePropsChanged = true
        } else if (shouldUpdateStateProps) {
          haveStatePropsChanged = this.updateStatePropsIfNeeded()
        }

```

对于`mapStateToProps` `mapDispatchToProps`第二个ownprops参数的认识：
这个2个函数的第二个参数主要是为了处理自身props变动的情形，如果这2个函数带了第二个参数ownprops，那么每次props变动的时候，这2个函数都会得到执行，
即为重新select,如果不带这2个参数，那么自身props有变动的时候，这2个函数不会得到执行，对于这2个函数，带不带第二个参数ownprops，props变更的行为都跟
裸用react组件时的行为一致。


`updateMergedPropsIfNeeded`是最后的过滤器
```js
componentDidMount() {
        this.trySubscribe()
      }
  trySubscribe() {
        if (shouldSubscribe && !this.unsubscribe) {
          this.unsubscribe = this.store.subscribe(this.handleChange.bind(this))
          this.handleChange()
        }
      }
handleChange() {
        if (!this.unsubscribe) {
          return
        }

        const storeState = this.store.getState()
        const prevStoreState = this.state.storeState
        if (pure && prevStoreState === storeState) {
          return
        }

        if (pure && !this.doStatePropsDependOnOwnProps) {
          const haveStatePropsChanged = tryCatch(this.updateStatePropsIfNeeded, this)
          if (!haveStatePropsChanged) {
            return
          }
          if (haveStatePropsChanged === errorObject) {
            this.statePropsPrecalculationError = errorObject.value
          }
          this.haveStatePropsBeenPrecalculated = true
        }

        this.hasStoreStateChanged = true
        this.setState({ storeState })
      }

```
 

# react-redux(v4)原理解析(ppt)

## 在用react-redux的时候难免会有些问题？
1. 组件怎么让我们无感知的得到store的？
2. connect干了什么？
3. connect过的组件如何知道store的变化？
4. connect过的组件如何处理父级传下来的props？
5. mapStateToProps是如何注入到props中的呢?
6. mapDispatchToProps是如何注入到props中的呢?
7. mergeProps是做什么的？
8. 整个react-redux流程是如何的？
8. 高阶组件的弊端？


## 组件怎么让我们无感知的得到store的？

### 利用最外层的Provide

provider主要做了传递store的事情，通过在parent组件内，使用getChildContext方法，这个方法可以把相应的属性隐式往下传，子组件及子组件的子组件都可以获得此属性


```javascript
export default class Provider extends Component {
  getChildContext() {
    return { store: this.store }
  }

  constructor(props, context) {
    super(props, context)
    this.store = props.store
  }

  render() {
    return Children.only(this.props.children)
  }
}
```

要注意`childContextTypes`需要声明出用到的属性
```javascript
/*
Index.childContextTypes={
	name: React.PropTypes.string.isRequired
}
*/
Provider.childContextTypes = {
  store: storeShape.isRequired
}

```

如同在`connect`中也要声明用到的`context`,如下
```js
/*
ScrollBar.contextTypes={
    name:React.PropTypes.string
}
*/
 Connect.contextTypes = {
      store: storeShape
    }
```

### 那么connect是什么？
### connect是一个`返回高阶组件`的`高阶函数`
[什么是高阶组件以及使用高阶组件的正确姿势?](https://react.bootcss.com/react/docs/higher-order-components.html)



```javascript
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);			 
			}	
			render() {
				return <wrappedComponent {...this.props} />;
			}
		}		 
	};
}
```

### connect过的组件如何知道store的变化？
```javascript
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);			 
			}
			componentDidMount() {
				this.store.subscribe(this.handleChange.bind(this));
				this.handleChange();
			}
			handleChange() {
				let currentState = this.store.getState();
				this.mergedProps = mapStateToProps(currentState);	
				this.setState({
					storeState: currentState,
				});
			}
			render() {
				let finnalProps = this.mergerPropsFunc(this.mergedProps, this.mergedActions, this.props);
				return <wrappedComponent {...finnalProps} />;
			}
		}		 
	};
}
```

### connect过的组件如何处理父级传下来的props?
## `透传`
```javascript
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);			 
			}	
			render() {
				return <wrappedComponent {...this.props} />;
			}
		}		 
	};
}
```

### mapStateToProps是如何注入到props中的呢?
`mapStateToProps`也叫`selector`
```javascript
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);
				this.store = props.store || context.store;
				const storeState = this.store.getState();
				//初始化state
				this.state = { storeState };
                //selector
				this.mergedProps = mapStateToProps(this.store.getState());
								
			}		
			render() {			
				return <wrappedComponent {...this.props}  {...this.mergedProps} />;
			}
		}
		 
	};
}
```

## mapDispatchToProps是如何注入到props中的呢?
```javascript
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);
				this.store = props.store || context.store;
				const storeState = this.store.getState();
				//计算dispatch				 
				if (typeof mapDispatchToprops === 'function') {
					this.dispatchProps = mapDispatchToprops(this.store.dispatch);
				} else {
					//bindActionCreators(actionCreators, dispatch)
					//对象
					this.dispatchProps = bindActionCreators(mapDispatchToprops, this.store.dispatch);
				}
			}		
			render() {		
				return <wrappedComponent {...this.props} {...this.mergedProps} {...this.dispatchProps} />;
			}
		}
	};
}
```

## mergeProps是做什么的？
    `this.props`,`this.mergedProps`,`this.mergedActions`合成,等于`Object.assign`

```javascript
const defaultMergeProps = (stateProps, dispatchProps, ownerProps) => ({
	...stateProps,
	...dispatchProps,
	...ownerProps,
});
function connect(mapStateToProps, mapDispatchToprops, mergeProps, option) {
	return function wrappComponent(wrappedComponent) {
		return class extends Component {
			constructor(props) {
				super(props);
				this.store = props.store || context.store;
				const storeState = this.store.getState();
				//初始化state
				this.state = { storeState };			
				this.mergerPropsFunc = mergeProps || defaultMergeProps;
			 
			}		
			render() {
				let finnalProps = this.mergerPropsFunc(this.mergedProps, this.dispatchProps, this.props);
				return <wrappedComponent {...finnalProps} />;
			}
		}
	};
}
```

## 事实上`mapStateToProps` 和 `mapDispatchToProps`都有对应的默认实现
```javascript
const defaultMapStateToProps = state => ({}) // eslint-disable-line no-unused-vars
const defaultMapDispatchToProps = dispatch => ({ dispatch })
```
* mapStateToProps默认返回一个空对象，不做任何state转换
* mapDispatchToProps默认注入dispatch方法
* mergeProps合成三部分props,即stateProps,dispatchProps,this.props

## 三份重要的props
1.`this.stateProps` ------- mapStateToProps  

2.`this.dispatchProps` --- mapDispatchToProps  

3.`this.mergedProps` -------- mergedProps 

### connect的一切逻辑判断都围绕这三个props进行
 


# react-redux工作流程

![few](http://p0.meituan.net/dpgroup/f322c89db4114894a945c36f2bbf05ac87300.png)

### connect维护了自己的state





### 怎么处理三个props的
#### 默认`shallowEqual`
```javascript
export default function shallowEqual(objA, objB) {
  if (objA === objB) {
    return true
  }

  const keysA = Object.keys(objA)
  const keysB = Object.keys(objB)

  if (keysA.length !== keysB.length) {
    return false
  }

  // Test for A's keys different from B.
  const hasOwn = Object.prototype.hasOwnProperty
  for (let i = 0; i < keysA.length; i++) {
    if (!hasOwn.call(objB, keysA[i]) ||
        objA[keysA[i]] !== objB[keysA[i]]) {
      return false
    }
  }

  return true
}
```

对于`this.stateProps`的处理
```javascript
  function updateStatePropsIfNeeded() {
        const nextStateProps = this.computeStateProps(this.store, this.props)
        if (this.stateProps && shallowEqual(nextStateProps, this.stateProps)) {
          return false
        }
        this.stateProps = nextStateProps
        return true
      }
```
其执行一次相当于`mapStateToProps`执行一次

对于`this.dispatchProps`的处理
```javascript
 function updateDispatchPropsIfNeeded() {
        const nextDispatchProps = this.computeDispatchProps(this.store, this.props)
        if (this.dispatchProps && shallowEqual(nextDispatchProps, this.dispatchProps)) {
          return false
        }
        this.dispatchProps = nextDispatchProps
        return true
      }
```
其执行一次相当于`mapDispatchToProps`执行一次

对于`this.mergedProps`的处理
```javascript
    function updateMergedPropsIfNeeded() {
        const nextMergedProps = computeMergedProps(this.stateProps, this.dispatchProps, this.props)
        if (this.mergedProps && checkMergedEquals && shallowEqual(nextMergedProps, this.mergedProps)) {
          return false
        }
        this.mergedProps = nextMergedProps
        return true
      }
```
其执行一次相当于`mergeProps`执行一次

## 怎样性能最高？
![few](http://p0.meituan.net/dpgroup/f322c89db4114894a945c36f2bbf05ac87300.png)
## 不让组件`setState`


```javascript
     function handleChange() {
        if (!this.unsubscribe) {
          return
        }

        const storeState = this.store.getState()
        const prevStoreState = this.state.storeState
        //第一次判断
        if (pure && prevStoreState === storeState) {
          return
        }

        
        if (pure && !this.doStatePropsDependOnOwnProps) {
          const haveStatePropsChanged = tryCatch(this.updateStatePropsIfNeeded, this)
          //状态相等
          if (!haveStatePropsChanged) {
            return
          }
          if (haveStatePropsChanged === errorObject) {
            this.statePropsPrecalculationError = errorObject.value
          }
          this.haveStatePropsBeenPrecalculated = true
        }

        this.hasStoreStateChanged = true
        this.setState({ storeState })
      }
```

1. 有些情况下dispatch出来的一个action不会引起reducer结果的任何变化,直接返回
2. 之后如果`mapStateToProps`的入参只有一个，会进行一次select操作，如果selector得到的结果没有任何变化（浅拷贝）,则直接返回，这个时候效率是最高的，组件不清楚发生了什么，保持原状

## 怎么最大限度保证不setState?
## 1. connect的state的状态粒度最细
## 2. mapStateToProps函数签名不写第二个ownProps
 

 ## 谈谈 [ownProps]
`mapStateToProps`,`mapDispatchToProps`，函数签名中的第二个参数，其签名为
```javascript
(state/dispatch,props)=>{
    return …………
}
```
从其功能来看，调用`mapStateToProps`,`mapDispatchToProps`得到的`新props`跟`state`与`组件props`有关，映射出来的`props`将会与`state`,`组件props`强耦合，因此只要有新props到达，都会进行一次selector,虽然带来了灵活性，但是增加selector的次数，性能稍有损失


对于`mapStateToProps` `mapDispatchToProps`,如果不带这个`ownProps`参数，那么自身props有变动的时候，这2个函数不会得到执行，对于组件来说，带不带第二个参数ownprops，props变更的行为都跟裸用react组件时的行为一致。



## 对于connect过的高阶组件的弊端？
Refs属性不能传递，所以源码中会有
```javascript
            if (withRef) {
          this.renderedElement = createElement(WrappedComponent, {
            ...this.mergedProps,
            ref: 'wrappedInstance'
          })
        }
```
这样的写法，在外部需要2次调用refs才能得到真正的示例

 
