---
title: class语法糖分析
date: 2017-06-14 21:20:52
tags: babel
---

## class语法糖分析

比较好奇class到es5 babel是怎么转换的

```javascript
 class Base {
    constructor (name) {        
        this.name =  name
    }
    
    state={
        isMan:false
    }

    keepQuiet(){

        console.log('base--->………………………………')
    }

    alert(){
        alert('base--->alert')
    }
}
<!--more-->
class child extends Base{
    constructor(name,age) {
        super(name);
        this.age= age;
        
    }
    buySomething=()=>{
        console.log('child buy')
    }
    alert(){
        alert('child----->alert')
    }
}
new child('titan',80)
```

生成的代码如下：重点看下`_createClass`，`_inherits`方法
```javascript
('use strict');

//获得一个对象的某个属性
//prototype,'constructor',this
var _get = function get(_x, _x2, _x3) {
	var _again = true;
	_function: while (_again) {
		var object = _x, property = _x2, receiver = _x3;
		_again = false;
		if (object === null) object = Function.prototype;
		var desc = Object.getOwnPropertyDescriptor(object, property);
		if (desc === undefined) {
			var parent = Object.getPrototypeOf(object);
			if (parent === null) {
				return undefined;
			} else {
				_x = parent;
				_x2 = property;
				_x3 = receiver;
				_again = true;
				desc = parent = undefined;
				continue _function;
			}
		} else if ('value' in desc) {
            //这个desc.value是Base的构造函数
			return desc.value;
		} else {
            //这个应该是只针对可读的
			var getter = desc.get;
			if (getter === undefined) {
				return undefined;
			}
			return getter.call(receiver);
		}
	}
};

var _createClass = (function() {
	function defineProperties(target, props) {
		for (var i = 0; i < props.length; i++) {
			var descriptor = props[i];
			descriptor.enumerable = descriptor.enumerable || false;
			descriptor.configurable = true;
			if ('value' in descriptor) descriptor.writable = true;
			Object.defineProperty(target, descriptor.key, descriptor);
		}
	}
    //闭包
	return function(Constructor, protoProps, staticProps) {
        //真正的构造执行的地方
        //定义原型属性
        //传入的protoProps是一个对象，babel都会生成对应的key,value属性
		if (protoProps) defineProperties(Constructor.prototype, protoProps);
        //定义静态属性，直接挂载在Constructor上
		if (staticProps) defineProperties(Constructor, staticProps);
		return Constructor;
	};
})();

function _inherits(subClass, superClass) {
	if (typeof superClass !== 'function' && superClass !== null) {
		throw new TypeError('Super expression must either be null or a function, not ' + typeof superClass);
	}
    //创建一个空对象，对象的__proto__指向superClass.prototype，传统继承模式
    /*
        create 的第二个参数：
                    propertiesObject
            可选。该参数对象是一组属性与值，该对象的属性名称将是新创建的对象的属性名称，值是属性描述符（这些属性描述符的结构与Object.defineProperties()的第二个参数一样）。注意：该参数对象不能是 undefined，另外只有该对象中自身拥有的可枚举的属性才有效，也就是说该对象的原型链上属性是无效的。
            prototype上有constructor属性
    */

	subClass.prototype = Object.create(superClass && superClass.prototype, {
		constructor: { value: subClass, enumerable: false, writable: true, configurable: true },
	});

    //getPrototypeOf  ------   setPrototypeOf
	if (superClass)
		Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : (subClass.__proto__ = superClass);
}

function _classCallCheck(instance, Constructor) {
	if (!(instance instanceof Constructor)) {
		throw new TypeError('Cannot call a class as a function');
	}
}

var Base = (function() {
	function Base(name) {
		_classCallCheck(this, Base);

		this.state = {
			isMan: false,
		};

		this.name = name;
	}

	_createClass(Base, [
		{
			key: 'keepQuiet',
			value: function keepQuiet() {
				console.log('base--->………………………………');
			},
		},
		{
			key: 'alert',
			value: (function(_alert) {
				function alert() {
                    //递归情况
					return _alert.apply(this, arguments);
				}

				alert.toString = function() {
					return _alert.toString();
				};

				return alert;
			})(function() {
				alert('base--->alert');
			}),
		},
	]);

	return Base;
})();

var child = (function(_Base) {
    //继承Base
	_inherits(child, _Base);

	function child(name, age) {
		_classCallCheck(this, child);
        
        /*
            _get(Object.getPrototypeOf(child.prototype), 'constructor', this)
            得到Base的构造函数，通过call注入this得到父类的属性，name
        */
		_get(Object.getPrototypeOf(child.prototype), 'constructor', this).call(this, name);

        //箭头函数在这声明
		this.buySomething = function() {
			console.log('child buy');
		};

		this.age = age;
	}

    //在自己的原型上，覆盖了父类的alert
	_createClass(child, [
		{
			key: 'alert',
			value: (function(_alert2) {
				function alert() {
					return _alert2.apply(this, arguments);
				}

				alert.toString = function() {
					return _alert2.toString();
				};

				return alert;
			})(function() {
				alert('child----->alert');
			}),
		},
	]);

	return child;
})(Base);

new child('titan', 80);

```