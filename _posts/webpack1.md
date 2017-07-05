---
title: webpack v1 概览
date: 2016-11-28 21:20:52
tags: 基础工具
---
  
# 目录
1. Webpack简介
2. Webpack常用概念
3. Webpack运行时
4. Webpack使用
5. Webpack进阶
6. 通灵塔webpack编译优化（Dll/DllReference）
 
-----
### 1. Webpack 简介

Webpack 是德国开发者 Tobias Koppers 开发的模块加载器。核心的理念是，视各种资源，例如JS（含JSX）、coffee、样式（含less/sass）、图片等都作为模块，来使用和处理。
![webpack](http://p0.meituan.net/dpgroup/f78661bef717cf2cc2c2e5158f196384368389.png)
### webpack.config.js 配置
1.**entry**  
2.**output**

```js
        var path = require("path");
        var webpack = require("../../");
        module.exports = {
          entry: {
            main: "./example",
            common: ["./vendor"] // optional
          },
          output: {
            path: path.join(__dirname, "js"),
            filename: "[name].chunkhash.js",
            chunkFilename: "[chunkhash].js"
          },
          plugins: [
            new webpack.optimize.CommonsChunkPlugin({
              names: ["common"]
            })
          ]
         };
```
-----  
### 2. Webpack 常用概念
1. 模块(module)
2. chunk
3. 资源(assets)
4. loader(资源加载)
5. plugin
6. 代码分割
-----
#### 2.1 模块

模块间依赖关系网(js,css,png,node_module)

![modules](http://p1.meituan.net/dpgroup/b41b6c0d667f6dc1235ed93c43b3a9f9649848.png)

 

-----

模块使用CommonJs管理，每个模块有一个模块id,一般情况下id为数字，但可以使用插件换成文件具体路径，以下
为id为184的模块

        /* 184 */
        /***/ function(module, exports, __webpack_require__) {

          
          var Core = __webpack_require__(185);
          var NativeCore = __webpack_require__(194);
          var Network = __webpack_require__(196);
          var Promise = __webpack_require__(188);
          
          Core.prototype._mixin(Core.prototype, NativeCore);
          Core.prototype._mixin(Core.prototype, Network);
          Core.prototype.all = function(list){
            return Promise.all(list);
          };
          module.exports = Core;


        /***/ },


-----
#### 2.2 chunk
* 上图打包成chunk之后得到3个chunk
* chunk是一个或多个源文件的集合，可以是js,css,png等，表现为一个数组，装载这些文件.
chunk有父子概念， 多个chunk也可以包含同一个module模块.
![chunk](http://p0.meituan.net/dpgroup/bdec532507526e4b101e78ff3ef64cc8235544.jpeg)
* 一般来说每个entry产生一个chunk,但可以借助插件来改变chunk的依赖情况（后续commonplugin会涉及），父子chunk的表现为chunk执行前置后置的关系
* 网状的modules最后经过webpack整理最后会表现为一个chunk有一个modules数组，这个数组全是module,且由webpack理清了网状依赖关系
一个chunk有多个module,平行的module
![](http://p1.meituan.net/dpgroup/71e52a5970a606c6f5afb63f7444c801862437.jpeg)
-----
每个chunk也有一个唯一的id,并且有对应的flag

1. entry:runtime
2. initial:优化chunk
3. render:normal chunk 没有runtime
![chunk](http://p0.meituan.net/dpgroup/bdec532507526e4b101e78ff3ef64cc8235544.jpeg)
<!--more-->
-----
#### 2.3 assets

即为最终项目中引用的文件，一个资源文件中可能含有一个或多个chunk，并且有一个或多个module，表现如下

    window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules) {}

![](http://p0.meituan.net/dpgroup/e9bbc663942e8332470a9982aa3c7c6b488021.jpeg)

    
-----
#### 2.4 loader
webpack中所有文件即模块，用loader将所有文件转换为模块，loader为资源加载器，对于每一类文件都需要设置loader。
loader是一个函数，可以传递参数进行loader设置
常用loader有：

1. babel-loader
2. css-loader
3. style-loader
4. url-loader
5. file-loader
6. …………
-----
#### 2.5 plugin
plugin用处广泛，是webpack的重要组成部分，利用plugin可以侵入编译过程，改变chunk的依赖关系
常用plugin:

1. CortexRecombinerPlugin
2. CommonsChunkPlugin
3. ExtractTextPlugin
4. WebpackShellPlugin
5. …………





-----
#### 2.6 代码分割

代码分割得到的结果类似于RequireJS，可以异步加载模块
分割点：

* CommonJs：require.ensure(dependencies, callback)
* AMD：require(dependencies, callback)
![code spliting](http://p1.meituan.net/dpgroup/d922acba4cd2284f0db4415b9ff935dd382244.jpeg)
* 对于被分割后的chunk,其依赖也在被分割后的chunk里面

### 3. Webpack运行时


常用的有有3个方法：

1. ``__webpack_require__``
2. ``__webpack_require__.e= function requireEnsure(chunkId, callback) {}``
3. ``webpackJsonp= function webpackJsonpCallback(chunkIds, moreModules)``

-----

#### ``1. __webpack_require__``

``__webpack_require__``方法用来同步加载已经安装过的模块，即执行一个模块中的代码，得到module.exports的值， 要运行的模块在打包的时候就已经打包进assets中，属于同步加载，影响文件体积




-----
#### ``2. requireEnsure``

``__webpack_require__.e=requireEnsure``  用以支持code spliting，通过挂载script标签来异步加载模块，不影响文件体积。

-----
#### ``3. webpackJsonp= function webpackJsonpCallback(chunkIds, moreModules)``

  ``webpackJsonp``方法用来安装模块，方法中chunkIds表示本assets中包含的chunk，为一个数组，可以有1个或多个id，列出的chunkid标志即将安装的chunkid，安装过的chunk在异步加载时可以直接调用，不再需要加载script标签，第二个参数为包含的模块，可以为对象或者数组，其中对象标志有模块对应的模块id，在执行后这些模块将被加载进内存，可以通过``__webpack_require__(模块id)``的方式调用

-----

### 4. Webpack使用

#### 4.1 webpack.config.js配置
1. entry
2. output
3. module
4. resolve
5. externals
6. target
7. devServer
8. plugin
-----
#### 4.2 [loader](https://webpack.github.io/docs/using-loaders.html)

* test：一个匹配loaders所处理的文件的拓展名的正则表达式（必须）
* loader：loader的名称（必须）
* include/exclude:手动添加必须处理的文件（文件夹）或屏蔽不需要处理的文件（文件夹）（可选）；
* query：为loaders提供额外的设置选项（可选）
* 链式调用:css!postcss!less
* 传参数：css?modules!postcss!less
* require指定：require("style-loader!css-loader!less-loader!./my-styles.less");
* [编写loader](https://webpack.github.io/docs/how-to-write-a-loader.html)


-----
#### 4.3 [plugin](https://webpack.github.io/docs/using-plugins.html)

插件（Plugins）是用来拓展Webpack功能的，它们会在整个构建过程中生效，执行相关的任务。

        plugins: [
          new webpack.optimize.CommonsChunkPlugin({
              name: "common",
              filename: "common.js",
              minChunks: Infinity//当项目中引用次数超过2次的包自动打入commons.js中,可自行根据需要进行调整优化

          }),
          new ExtractTextPlugin("[name].css", {
              // disable: env == "dev",
              allChunks: true
          }),
          new CortexRecombinerPlugin({
              base: path.resolve(__dirname, relativeToRootPath),
              noBeta:env=='product'

          }),
          new webpack.WatchIgnorePlugin([path.resolve(__dirname, relativeToRootPath, "./node_modules/@cortex")]),
          new WebpackShellPlugin({onBuildStart: ['gulp']})

        ]

-----
1.[CommonsChunkPlugin （4种使用情形 改变chunk依赖）](https://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin)

* 打包多个入口的共同文件(**多页或者是多个执行环境**)   
  从不同入口抽出多个公共的模块，考虑如下简单情形
![](http://p1.meituan.net/dpgroup/d8708a6579e9ed8dd1f93a1024a5583b291432.jpeg)有2个没有引入该插件的entry,得到2个分立的chunk,查看module发现有些模块是重用的
![](http://p0.meituan.net/dpgroup/368fc4c0b7c7b5ae3cdfc23477f96ed5665006.jpeg)
因此可以将这些公共模块抽出，引入CommonsChunkPlugin，得到：
![](http://p0.meituan.net/dpgroup/8d24541db36a4d9be0c4497d15b027bd371215.jpeg)
抽出了公共模块出来   
共享模块分配到一个新的chunk中了:
![](http://p0.meituan.net/dpgroup/6d2ac863ddea5e07d3d7e222f64c899b782958.jpeg)

* 打包工具库文件(**现有项目使用**)  
  zepto,react,…………

* 移动共同依赖模块(**code spliting增加页面初始化时间**)
   针对code spliting产生多个chunk的情形，移动一部分共同的chunk到common中，减少异步请求次数，针对代码分割点比较多又小的情况  

* 异步加载共有common chunk 
  针对code spliting，移动一部分公共的代码块形成一个异步common chunk,没有增加页面初始时间，但会在某个分割点加载分割common代码



-----

2.[DedupePlugin](https://webpack.github.io/docs/list-of-plugins.html#dedupeplugin)   
  去冗余  

3.[OccurrenceOrderPlugin](https://webpack.github.io/docs/list-of-plugins.html#occurrenceorderplugin)   
   重设模块id,使得高使用率的模块获得数字较小的id,表现为内存优先加载

4.[UglifyJsPlugin](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)  
   js压缩

5.[DllPlugin](https://webpack.github.io/docs/list-of-plugins.html#dllplugin)   
   用来生成动态链接库

6.[DllReferencePlugin](https://webpack.github.io/docs/list-of-plugins.html#dllplugin)  
   用来引入动态链接库


-----

7.[CortexRecombinerPlugin](https://webpack.github.io/docs/list-of-plugins.html#dedupeplugin)   
  cortex代码转换，移动

8.[DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin)   
   给代码中引入变量

9.[ProvidePlugin](https://webpack.github.io/docs/list-of-plugins.html#provideplugin)  
  提供全局变量，而不用在每个入口require('zepto')等  

        new webpack.ProviderPlugin({  $:"jquery",  jQuery:"jquery",  "window.jQuery":"jquery"})]

10.[HotModuleReplacementPlugin](https://webpack.github.io/docs/list-of-plugins.html#hotmodulereplacementplugin)  
   热更新插件，需要配合webpack-dev-server

11.[ExtractTextPlugin](https://github.com/webpack/extract-text-webpack-plugin)  
    抽取文本，文本不一定是css,除了有module.exports的js和json文件都可以算是文本

12.………………

#### [4.4 externals](https://webpack.github.io/docs/library-and-externals.html)

假如我们需要通过标签来引入jquery，并且要写一个jquery插件，并且写的这个插件要用webpack打包，可以使用下面的配置


        module.exports = {  
           output: {
              //target也可以为UMD
              libraryTarget: "window",
              library:['jquery','custom'] 
          },   
            externals: ["$", 'jquery']
        }
 
以上的``output``表示给window的jquery.custom对象挂上我们的插件模块
 
        window["jquery"] = window["jquery"] || {}; window["jquery"]["custom"] =XXX
 
``externals``表示代码中遇到require('jquery')或者require('$')不进行依赖判断，直接输出
 
        module.exports = window["jquery"];
 
-----

``externals``也可以写为对象的形式

        externals: { 
          "jquery": "jQuery" 
        }

require('jquery')等价为：window.jQuery，所有的jquery都会被解析为jQuery，
在UMD模式下，如果有CommonJs模块，require('jquery')会被解析为require('jQuery')，

        require('jquery')=>require('jQuery')

要注意此时的2个不同的require，第一个require为webpack中使用的require，第二个为CommonJs的require


-----
#### [4.5 非标准模块引入 shimming-modules](https://webpack.github.io/docs/shimming-modules.html)

1. [imports-loader](https://github.com/webpack/imports-loader)

        require("imports?xConfig=>{value:123}!./file.js")
    
    引入全局变量xConfig={value：123}给到``file.js``

2. [exports-loader](https://github.com/webpack/exports-loader)暴露到变量中

        require("imports?variable=>{}!exports?variable.XModule!./file.js")

    先引入变量variable={}给到``file.js``再暴露XModule在variable对象中

3. [expose-loader](https://github.com/webpack/exports-loader)暴露到全局

        require("expose?XModule!./file.js")

   
-----
#### [4.6 热更新](https://webpack.github.io/docs/hot-module-replacement-with-webpack.html)   
   热更新依赖2个模块，一个是webpack-dev-server，是一个npm包，提供socket，类似静态资源服务器，第二个为HotModuleReplacement，可以通过提供插件或者命令行的形式启动HMR，要启动HMR需要在每个entry中加入，webpack/hot/dev-server或者webpack/hot/only-dev-server,前者在热更新失败的时候会刷新整个页面，后者不刷新，如果更新失败，需要手动刷新
  
-----
热更新原理：   

webpack-dev-server为通过webpack打包生成的资源文件提供Web服务。通过Socket.IO连接着webpack-dev-server服务器的HMR runtime。webpack-dev-server发送关于编译状态的消息到客户端，客户端根据消息作出响应。

![](http://p0.meituan.net/dpgroup/7072779a32354cc869a6e46de2849fc631676.png)
  

-----
## 可视化工具
### [webpack analyse](http://webpack.github.io/analyse/) 

  官方工具
![](http://p1.meituan.net/dpgroup/5facd4d1fe50a9680dabbca9d34ac621812368.jpeg)
模块详情：
![](http://p1.meituan.net/dpgroup/3178311bca0da34238e63ae4c34794e41108255.jpeg)
chunk详情：
![](http://p1.meituan.net/dpgroup/285c22b47d4f672ce08f855dbf73ce651001175.jpeg)
-----

### [webpack visualizer](https://chrisbateman.github.io/webpack-visualizer/) 

   分析模块占比 

![webvisual1](http://p1.meituan.net/dpgroup/c7b4bb1f1d31bbf04d4296147dc49446542927.jpeg)




-----

### [Webpack Chart](https://alexkuz.github.io/webpack-chart/)

![webchart](http://p1.meituan.net/dpgroup/4cf1e0e16389c4f847b97496d5dc3db4379756.jpeg)


-----

### 5. Webpack进阶/总结

#### 5.1 调试

* 使用react-transform-hmr热更新react

* 选择性热更新

         if (module.hot) {
              module.hot.accept('./components/App', render);
          }

* 服务端集成，koa结合webpack-middleware


-----

#### 5.2 代码编写

* 某些模块只加载一部分（lodash,zepto）

* 代码分割（按行为/按环境(微信，DPApp)）

* 整合css-sprite（[sprity](https://github.com/sprity/sprity)）

* 分析打包报告（stats.json 可视化工具）


### 6. 通灵塔编译打包速度优化
随着通灵塔功能越来越强大，其背后引用到的第三方库也越来越多，文件也变得越来越大，如图所示，编译后的模块有1835个，公共库加index.js接近7MB，现在一次编译打包的时间要5-15秒不等，每次启动通灵塔项目都要花一些时间，虽然CommonsChunkPlugin解决了公共库的问题，但是每次webpack编译都要生成一遍公共库common.js，由于common.js里都是相对稳定的三方库，如react,react-dom等，这些三方库应该不需要每次都编译打包，因而可以使用DllPlugin + DllReferencePlugin来解决，通过DllPlugin生成稳定的DLL模块，并让webpack不对这些模块每次都编译打包
![](http://p1.meituan.net/dpgroup/b2f1257a2013ac42114baa12f6460482237625.jpeg)

先在项目路径下引入``webpack.dll.js``，内容如下所示，通过webpack.DllPlugin引入DLL插件，DllPlugin不执行模块代码，只提供了模块的实现，具体的调用还得其他模块来进行，其他模块利用DLL插件生成的映射关系.json文件来进行模块调用，之后利用OccurenceOrderPlugin来重排模块id,增加模块命中概率，利用UglifyJsPlugin来压缩JS，因为这个dll最后还是要写在HTML中以script标签的形式引用
```js
var path = require("path");
var webpack = require("webpack");

module.exports = {
    entry: {
        vendor: ['./vendors.js']
    },
    output: {
        path: path.join(__dirname, "dll"),
        filename: "dll.[name].js",
        library: "[name]"
    },
    plugins: [
        new webpack.DllPlugin({
            path: path.join(__dirname, "dll", "[name]-manifest.json"),
            name: "[name]"
        }),
        new webpack.optimize.OccurenceOrderPlugin(),
        new webpack.optimize.UglifyJsPlugin()
    ],
    resolve: {
        root: path.resolve(__dirname, ""),
        modulesDirectories: ["node_modules"]
    }
};
```
项目入口设置一个vendors.js文件，将稳定的模块都可以放入这个js中，最后执行``webpack --config webpack.dll.js``生成DLL dll.vendor.js，如下所示
![](http://p1.meituan.net/dpgroup/ef877faf09d6a1c9eb0550a9bbb7e34d1084415.jpeg)
得到了一个压缩后1.75MB的dll文件，并且一共有1083个模块，之后在原有项目中再引入DllReferencePlugin插件,由webpack根据manifest.json映射文件生成调用关系
```js
   new webpack.DllReferencePlugin({
            manifest: require("../dll/vendor-manifest.json"),
            context: path.join(__dirname, '..')
        }),
```
manifest.json片段：
```js
  "./node_modules/react/react.js": 1,
    "./node_modules/process/browser.js": 2,
    "./node_modules/moment/moment.js": 3
```
再执行webpack命令，打包得到如下结果
![](http://p0.meituan.net/dpgroup/4ca02c616f59b0020c48706f4970750f266526.jpeg)
common.js+index.js不到3M，一次编译打包也只有857个模块，少了1000多个模块的编译打包，实测打包速度快了2秒左右，根据机器差异和机器资源使用情况webpack编译打包的时间应该能快上1-5秒不等。最后消失的模块都跑到dll.vendor.js中了，为了让页面正常运行需要在页面中引入这个dll
![](http://p0.meituan.net/dpgroup/9ffbb3baf2bb007b232f26adc264dd26289384.jpeg)
之后页面正常运行~  
总结：使用DLL+DllReferencePlugin能在开发阶段节省每次编译打包的等待，在beta或者线上也可以节省脚本执行webpack命令从开始到结束的时间，但是需要再引入一个script标签，需要视情况使用DLL。   

DLL相关代码已push到通灵塔app-yy-ziggurat-dashboard的[dll分支](http://code.dianpingoa.com/ziggurat/app-yy-ziggurat-dashboard/commits/dll)


参考资料：     

1. http://webpack.github.io/docs/
2. http://engineering.invisionapp.com/post/optimizing-webpack/
3. https://github.com/petehunt/webpack-howto
4. http://gaearon.github.io/react-hot-loader/
5. https://github.com/petehunt/webpack-howto#8-optimizing-common-code