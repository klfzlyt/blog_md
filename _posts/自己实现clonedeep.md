---
title: 自己实现lodash/cloneDeep
date: 2016-12-14 21:20:52
tags: 前端基础
---

### lodash之cloneDeep的实现

#### 背景
先回顾一下浅拷贝
![](http://p0.meituan.net/dpgroup/9d5d2a98381895c92101521c779dd88b52421.png)
如图所示，浅拷贝是新建立了一个对象，但是对象每个key还是引用了之前的。
而我们的深拷贝需要开辟所有的内存区域
![](http:////p1.meituan.net/dpgroup/8fcb3603ce1676152630d2f27dffe2d542972.png)

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

![](http://p0.meituan.net/dpgroup/da9554618959e748328baf9f5534335d43523.png)
如图所示，要把上图中的对象深拷贝为下面的对象。

**其实对于这种对象，用用递归就能简单实现**。

**但是要注意**
对于下图的循环引用
![](http://p0.meituan.net/dpgroup/a204b16b78019139dcaad5c2c4e138f550010.png)
↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
![](http://p1.meituan.net/dpgroup/09a449fa74e7c5dd4826d96978ff6b7f50506.png)

如果要递归来拷贝的话，那么得要加一些逻辑，不然一个递归将无限进行下去，因为这里面的路径有一个环。

参考lodash,我们用一个hashMap保存每个**对象有没有被Copy过**
![](http://p1.meituan.net/dpgroup/6d41a3b5c06fc4973f57b8e0c6cf604055993.png)
由于对象作为key,可以用symbol，也可以用其他手段实现，这里我们用stack,stack每个元素为一个array pair来实现，当然这种实现不唯一。

<!--more-->

#### 具体实现
```javascript
module.exports=(function(){
    /*
        用来存储有没有copy过的栈
    */  
    function stackSet(key,value){
        //o(n)
       var data=this.__data__;
        var index=this.arryIndexOf(key);
        if(index<0){
              //var data=[];//这个里面保存的是数组arry,数组的第一个是key,数组的第二个value
            data.push([key,value]);
        }
        else{
            data[index][1]=value
        }
    }
    function arryIndexOf(key){
        var data=this.__data__;
        var length=data.length;
        for(var i=0;i<length;i++){
            var entry=data[i];
            if(entry[0]==key)
                return i;
        }
        return -1;
    }
    function stackHas(key){
        var data=this.__data__;
        var length=data.length;
        for(var i=0;i<length;i++){
            var entry=data[i];
            if(entry[0]==key)
            return i!=-1;
        }
        return false;
    }
    function stackGet(key){
        //o(n)
        var index=this.arryIndexOf(key);
        if(index<0)return;
        return this.__data__ && this.__data__[index] && this.__data__[index][1];
    }
    function Stack(){
        var dataarry=[];
        this.__data__=dataarry;//这个里面保存的是数组arry,数组的第一个是key,数组的第二个value
    }
    Stack.prototype.get=stackGet;
    Stack.prototype.set=stackSet;
    Stack.prototype.arryIndexOf=arryIndexOf;
    

    /*
        Util functions
    */
    var TAGS=[
        '[object Number]',
        '[object Date]'
    ];
    var toString=Object.prototype.toString;
    function isObject(object){
        return object!=null && ( typeof object == 'object')
    }
    function assignkeyvalue(object,key,value){
        object[key]=value;
    }
    function baseAssign(object,props){
        var index=-1;
        var length=props.length;
        if(!object){
            return;
        }
        var dest={};
        while(++index<length){
            var key=props[index];
            assignkeyvalue(dest,key,object[key])
        }
        return dest
    }
    function getTag(object){
        return toString.call(object);
    }
    

    /*
        拷贝对象
    */
    function copyObject(object,stack){
        if(!isObject(object)) {
            return
        }
        var tag=getTag(object);
        if(!!~TAGS.indexOf(tag)){
            //Number Data  还有很多，这里是个示意
            var ctor=object.constructor;
            return new ctor(+object);
        }
        var index=-1;
        var keys=Object.keys(object);
        var length=keys.length;
        var dest={};
        //keys not include symbol
        while(++index<length){
            var key=keys[index];
            if(isObject(object[key])){
                stack=stack||new Stack;
                //看对象有没有copy过
                var saved_value=stack.get(object);
                if(saved_value){
                    return saved_value
                }
                //设置为已拷贝
                stack.set(object,dest);
                //递归赋值
                assignkeyvalue(dest,key,copyObject(object[key],stack));
            }
            else{
                assignkeyvalue(dest,key,object[key]);
            }
        }
        return dest;
    }
    /*
        拷贝数组
    */
    function copyArry(arry){
        if(!Array.isArray(arry)){return []}
        var dest =[],
            index=-1,
            length=arry.length;
        while(++index<length){
            if(isObject(arry[index])){
                dest[index]=copyObject(arry[index])
            }
            else{
                dest[index]=arry[index]
            }
        }
        return dest;

    }
    function cloneDeep(object){
        if(!isObject(object))return object;
        return Array.isArray(object)?copyArry(object):copyObject(object)
    }

    return cloneDeep;
})();

```

接下来我们用mocha 写一些用例：

```javascript
var assert = require('assert');
var deepClone = require('./clonedeep.js');
var lodash = require('lodash');
/*
    这部分是自己的cloneDeep实现
*/
describe('cloneDeep', function() {
	describe('own', function() {
		it('plant object', function() {
			var origin = {
				a: 1,
			};
			var dest = deepClone(origin);
			assert.notEqual(origin, dest);
		});
		it('nest object', function() {
			var origin = {
				a: {
					b: new Date(),
				},
			};
			var dest = deepClone(origin);
			assert.notEqual(origin, dest);
			assert.deepEqual(origin, dest, '应该相等');
			assert.notEqual(origin.a, dest.a, '属性不应该相等');
		});
		it('function', function() {
			var origin = {
				fuc: function() {},
			};
			var dest = deepClone(origin);
			assert.ok(origin.fuc === dest.fuc);
			assert.equal(origin.fuc, dest.fuc);
		});
		it('arry', function() {
			var orign = [1, 3, 3, 5, 9];
			var dest = deepClone(orign);
			assert.notEqual(dest, orign);
			assert.deepEqual(dest, orign);
		});
		it('arry nest object', function() {
			var origin = [{ a: { b: 1 } }, { b: new Number(1) }];
			var dest = deepClone(origin);
			assert.deepEqual(origin, dest);
			assert.notEqual(origin[0], dest[0]);
			assert.notEqual(origin[0].a, dest[0].a);
		});
		it('primary', function() {
			var origin = 1;
			var dest = deepClone(origin);
			assert.equal(origin, dest);
		});
		it('primary object', function() {
			var orgin = Object(1);
			var dest = deepClone(orgin);
			assert.notEqual(orgin, dest);
			assert.deepEqual(orgin, dest);
			assert.deepEqual(orgin.toString(), dest.toString());
		});
		it('date', function() {
			var orgin = new Date();
			var dest = deepClone(orgin);
			assert.notEqual(orgin, dest);
			assert.deepEqual(orgin, dest);
			assert.deepEqual(orgin.toString(), dest.toString());
		});
		it('undefine', function() {
			var orgin = undefined;
			var dest = deepClone(orgin);
			// assert.notEqual(orgin,dest);
			assert.strictEqual(orgin, dest);
			//assert.deepEqual(orgin.toString(),dest.toString());
		});
		it('circular reference', function() {
			var orgin = {};
			//orgin.a=orgin;
			var bb = {};
			var cc = {};
			bb.c = cc;
			cc.d = orgin;
			orgin.b = bb;
			var dest = deepClone(orgin);
			// assert.notEqual(orgin,dest);
			//console.log(orgin===dest)
			assert.notStrictEqual(orgin, dest);
			assert.notStrictEqual(orgin.b, dest.b);
			//assert.deepEqual(orgin.toString(),dest.toString());
		});
	});

	/*
        这部分是lodash的cloneDeep实现
    */
	describe('lodash', function() {
		it('plant object', function() {
			var origin = {
				a: 1,
			};
			var dest = lodash.cloneDeep(origin);
			assert.notEqual(origin, dest);
		});
		it('nest object', function() {
			var origin = {
				a: {
					b: new Date(),
				},
			};
			var dest = lodash.cloneDeep(origin);
			assert.notEqual(origin, dest);
			assert.deepEqual(origin, dest, '应该相等');
			assert.notEqual(origin.a, dest.a, '属性不应该相等');
		});
		it('function', function() {
			var fuc = function() {};

			var dest = lodash.cloneDeep(fuc);
			//console.log('fds',typeof dest)
			//for(var inn in dest){
			//    console.log('ii',inn)
			//}
			assert.notEqual(fuc, dest);
		});
		it('arry', function() {
			var orign = [1, 3, 3, 5, 9];
			var dest = lodash.cloneDeep(orign);
			assert.notEqual(dest, orign);
			assert.deepEqual(dest, orign);
		});
		it('arry nest object', function() {
			var origin = [{ a: { b: 1 } }, { b: new Number(1) }];
			var dest = lodash.cloneDeep(origin);
			assert.deepEqual(origin, dest);
			assert.notEqual(origin[0], dest[0]);
			assert.notEqual(origin[0].a, dest[0].a);
		});
		it('primary', function() {
			var origin = 1;
			var dest = lodash.cloneDeep(origin);
			assert.equal(origin, dest);
		});

		it('primary object', function() {
			var orgin = Object(1);
			var dest = lodash.cloneDeep(orgin);
			assert.notEqual(orgin, dest);
			assert.deepEqual(orgin, dest);
			assert.deepEqual(orgin.toString(), dest.toString());
		});

		it('date', function() {
			var orgin = new Date();
			var dest = lodash.cloneDeep(orgin);
			assert.notEqual(orgin, dest);
			assert.deepEqual(orgin, dest);
			assert.deepEqual(orgin.toString(), dest.toString());
		});
		it('undefine', function() {
			var orgin = undefined;
			var dest = lodash.cloneDeep(orgin);
			// assert.notEqual(orgin,dest);
			assert.strictEqual(orgin, dest);
			//assert.deepEqual(orgin.toString(),dest.toString());
		});
		it('circular reference', function() {
			var orgin = {};
			//orgin.a=orgin;
			var bb = {};
			var cc = {};
			bb.c = cc;
			cc.d = orgin;
			orgin.b = bb;
			var dest = lodash.cloneDeep(orgin);
			// assert.notEqual(orgin,dest);

			assert.notStrictEqual(orgin, dest);
			assert.notStrictEqual(orgin.b, dest.b);
			//assert.deepEqual(orgin.toString(),dest.toString());
		});
	});
});
```
测试有2部分，上面的部分是自己的cloneDeep，下面的是lodash的，测试结果通过
其实是慢慢的迭代，对用例进行开发（Test-driven development）
![](http://p0.meituan.net/dpgroup/54fa173c8bc335785211c19371e2a73d164500.jpg)