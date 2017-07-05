---
title: 动手实现一个简易HTML解析器
date: 2017-04-19 21:20:52
tags: 前端基础
---
在动手之前，先说说动机，之前看过浏览器工作原理的一系列文章，如： 
1. [chrome dom树构建源码解析](http://yincheng.site/build-dom)
2. [浏览器的工作原理：新式网络浏览器幕后揭秘](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)

以及一些规范
1.  [html解析构建规范](https://www.w3.org/TR/html5/syntax.html#the-stack-of-open-elements)
2. [css 规范](https://www.w3.org/TR/CSS2/grammar.html)

在了解了原理与一些规范之后就想要自己试试造个轮子，加深对浏览器的理解。在大致梳理好流程之后，有以下目录：
* * *
# 目录
1. [概述](#intro)
2. [HTML解析篇](#1)
3. [CSS解析篇](#2)
4. [应用CSS样式到DOM中](#3)
5. [事件系统](#4)
* * *
## 一.概述
在浏览器中，我们的类有以下的继承关系，如一个div dom 继承自HTMLElemnt,HTMLElemnt继承自Element等。为了方便起见，将一些重要的如dispatchEvent等方法，children，parent，attributes等属性都纳入HTMLElement中。并且依照css object model，构造cssom对象。
![图1 node关系](//p0.meituan.net/dpgroup/731d0d4fe3561d098a0abd9bad04323159173.jpg)

因而实现下面的类，并实现manager来管理，即可实现一个简易的解析器。
![图2 类概览](//p1.meituan.net/dpgroup/de6325f8325f412823e708f962e3039f141915.png)
* * *
## 二.HTML解析篇
解析html标签，最先想到的是正则，但是用正则可以解析，但是有几个问题：

1. **如何建立dom树的父子关系**
2. **如何建立dom树的兄弟关系**
3. **如何实现getElementById方法**
4. **纯文本节点怎么解析**
5. **innerHTML该怎么做**
6. **innerText该怎么做**
7. **addEventListener该怎么做**
8. **怎么构造attributeNode**


<!--more-->
其中`1`,`2`是构建dom树的关键，对于`7`将在[事件篇](#event)中实现。

##### 1. 如何建立dom树的父子关系
[chrome dom树构建源码解析](http://yincheng.site/build-dom)
参考这篇文章，我们需要做的是一个临时栈，以一个例子来说明
如图所示，有一段HTML文本，**我们的解析是从文本的第一个字符到最后一个字符，从上到下，从左到右进行解析**，按照chrome的做法，chrome是先解析文本，得到一系列token，再进行树的构建([参考文章](http://yincheng.site/build-dom))。而利用正则的步进过程，我们可以把2个阶段合成一个进行实现，绿色的部分是我们的临时栈，我们的解析策略是:**正则匹配到一个开始标签时，就new一个Node，然后推入绿色临时栈中，对于<section/>或纯文本的node，也进行new操作，但是不入栈，因为这个node已经算是闭合了**
![闭合标签 出栈](//p1.meituan.net/dpgroup/905528efeb365baa6e66c7ec75ce291347443.png)
图中，有一段HTML文本，先解析到<html>，之后new一个HTMLElement入栈，又解析到<head>，也new一个Element入栈，在遇到</head>闭合之后，便可以pop出<head>，同时把栈顶的元素作为被pop出元素的父元素，进行parent属性与children属性的操作，如图右侧代码所示。

在遇到id为f1的div之后，遇到一个文本dp，临时栈如图绿色部分所示
![文本解析](//p0.meituan.net/dpgroup/49aba37945b7aaedad23b58f0f9b725061305.png)
这个时候new完一个文本节点即可进行父子赋值操作了，如图右侧所示。

同理，类似<section/>操作如下：
![](//p0.meituan.net/dpgroup/1883b49aba549a17f84188510756e71663179.png)
因此dom树便可以在解析完文本之后构建出来
 
整个流程简化如下代码，每次解析到一个正则匹配的tag，便进行操作
```js
generatorHtmlNode(html){
    html = html.replace(/[\r\n]+/g, "");
    html = html.trim();
    //<> </>等的正则匹配
    let tagRegex = /<(\/)?([^\/<>]+)(\/)?>/g;
    //保存root
    let root;
    //记录tag解析位置，用以得到纯文本
    let cursor = 0;
    //用以记录前一个解析到的node
    let preslibingNode;
    //解析dom树过程中的临时栈
    let htmlstack = [];
    //保存正则匹配临时结果
    let regexResult
     while ((regexResult = tagRegex.exec(html))) {
           let holeMatch = regexResult[0];
           let isEnd = !!regexResult[1];
           let matchGroup = regexResult[2];
           let isEndOfOneTag = !!regexResult[3]; //是单闭合标签
           let tagName = findTheHtmlTag(regexResult[2]);
           //解析纯文本
           let planText = html.slice(cursor, regexResult.index);
           let parentNode = getTop(htmlstack);
            if (!flag) {
                let node = new HTMLElement(
                  tagName,
                  parentNode,
                  holeMatch,
                  findAttributeKeyPair(matchGroup)
                  );
                     //临时栈
                htmlstack.push(node);
            }
            else{
                 //遇到结束标识
                popStack(htmlstack);
            }
           /*
           省略
           …………………………………………
           …………………………………………
           …………………………………………
            */
           //更新文本游标
           cursor = regexResult.index + holeMatch.length;
     }
     return root;
}
```

##### 2. **如何建立dom树的兄弟关系**
同dom树的构建过程，我们可以开辟一些临时空间来保存之前创建的Element.
我们用一个变量preslibingNode来存储之前创建的Element.我们重要的是preslibingNode的更新时机，我们应该选择在遇到所有开闭标签以及文本的时候进行更新变量，并要注意赋值判断。以下图的例子来说明。
![兄弟关系构建](//p0.meituan.net/dpgroup/7d44db022557920401fb45a892d671f5119993.png)
左侧有一串html文本，我们的preslibingNode变量在解析过程中的变化过程如右侧所示。preslibingNode初始为undefined，在遇到第一个<div id="f2">并new 它时，preslibingNode为undefined，因而对<div id="f2">的previousSibling属性不作操作，并将preslibingNode更新为<div id="f2">，之后解析遇到<div id="f1">,preslibingNode为其parent,因而不对<div id="f1">的previousSibling属性进行操作，同理更新preslibingNode变量，要注意在遇到如</span>闭合标签的情况下，也要更新preslibingNode变量，但不必更新span节点的preslibingNode信息，如右图。之后遇到dp2的文本的时候，由于preslibingNode有值且不是其父亲节点，因而可以应用previousSibling属性，如代码所示：
```js
      if (
          preslibingNode &&
          preslibingNode != parentNode  
        ) {
          //构建双向链表
          node.previousSibling = preslibingNode;
          preslibingNode.nextSibling = node;
        }
```
之后的过程如图所示，直到所有的文本解析完毕。

##### 3. **如何实现getElementById方法**
实现getElementById方法，可以在解析的时候，每次解析到一个含有id的dom，就保存到一个全局hashMap中（key为id,value为dom），这样要要想获取，可以直接从hashmap中获取，减少遍历耗时，以空间换时间

##### 4. **纯文本节点怎么解析**
利用正则的index特性，再借用一个当前游标，便可以将当前的匹配的html标签与上一个html标签之间的内容保存下来，即为文本的内容
```js
generatorHtmlNode(html){
    //<> </>等的正则匹配
    let tagRegex = /<(\/)?([^\/<>]+)(\/)?>/g;
    let regexResult
     while ((regexResult = tagRegex.exec(html))) {
           let holeMatch = regexResult[0];
           //解析纯文本
           let planText = html.slice(cursor, regexResult.index);
           /*
           省略
           …………………………………………
           …………………………………………
           …………………………………………
            */
           //更新文本游标
           cursor = regexResult.index + holeMatch.length;
     }
     return root;
}
```

##### 5. **innerHTML该如何操作**
可以在解析到某个标签时，将这个标签字符串左边的内容都暂时砍去，然后往右找第一个对应的闭合标签，找到了，截取出来即为innerHTML，具体做法是把正则匹配到的结果result传入一个函数，如下所示：
```js
function getInnerHtml(regexResult, node) {
  //整个HTMLstring,每次都相同
  let holeHtml = regexResult.input;
  let tag = node.name;
  //切割到当前tag的children位置 获得innerHTML的右边部分
  let rightHtml = holeHtml.slice(
    regexResult.index + regexResult[0].length,
    holeHtml.length
  );
  //构造第一个匹配到的相应tag的闭合标签
  let re = new RegExp("<\\/(" + tag + ")>", "i"); // re为/<\/(div)>/i 没有/g标志，可以用match，效果和exec一样
  let regMatched = rightHtml.match(re);
  //进行切割返回
  return rightHtml.substring(0, regMatched.index);
}
```
#####  6. **innerText该如何操作**
innerText也应该很容易，利用innerHTML得到的文本，再利用replace(regex)方法，将标签替换为空即可,这个时候对正则不要分组，会影响性能
利用得到的innerHTML，进行正则替换即可，
```js
    innerHTML.replace(/<\/?[^\/<>]+\/?>/ig, "");
```
##### 7. **addEventListener该怎么做**
应该维护一个全局的hashMap，在调用addEventListener时，用以保存对应事件的listen,hashMap的key应为dom,value也应该为一个hashMap(x)
查看[Event]()部分

#####  8. **怎么构造attributeNode**
对于每一个开始标签，利用正则匹配出标签里面的id='xxx' class='xxx1 xx2'等，既可以构造相应的attributeNode
每个HTMLElement有一个attributeCollection，按正则匹配进行构造：
```js
function findAttributeKeyPair(str) {
  let attributeRegex = /(([\w]+)=(('|")[\s\w-]+('|")))/g;
  let result;
  let pairs = [];
  let attributeCollection = [];
  while ((result = attributeRegex.exec(str))) {
    pairs.push(result[1]);
  }
  for (let i = 0; i < pairs.length; i++) {
    let pair = pairs[i];
    let keyValue = pair.split("=");
    let key = keyValue[0];
    let value = keyValue[1].replace(/['"]+/g, "");
    attributeCollection.push(new attributeNode(key, value));
  }
  return attributeCollection;
}
```
* * * 

## 三.CSS解析篇
DOM-CSS / CSSOM
浏览器css object model对象，通过document.styleSheets可以得到当前文档的所有样式集，每一个<style></style>
标签可以对应解析出一个CSSStyleSheet对象，本文支持解析出CSSStyleSheet对象
CSSStyleSheet对象有一个cssRules属性，其为一CSSRuleList集合，保存有style内含的所有规则，每一个规则为CSSStyleRule。如下：
![](http://www.runoob.com/images/selector.gif)
因而我们可以按照上图构造对象，构造selector，property等属性。按照文章开头的介绍，我们构造了下面的css object model。
![cssom](//p1.meituan.net/dpgroup/6d1bed9532c62564b2608135088421e361265.png)
这个规则对应到我们具体些的css样式，即为每一个大括号，每一个CSSStyleRule有cssText属性保存整个css文本，
selectorText用来保存css选择器文本，此外我们还引入innerStyleRule到这个规则中，innerStyleRule用以递归css的每一个选择器，以匹配dom是否满足样式规则，并引入_weight对象来保存CSSStyleRule的权重。
综上所述，所以我们的目的是解析一段样式文本，得到一个数组，这个数组中保存有多个CSSStyleRule，每一个CSSStyleRule对应一个样式规则
所以有几个问题？
1.如何解析得到每一个css block块？
2.如何得到选择文本？
3.如何得到每个具体样式
对于上述三个问题，都可以用正则来解析得到
```js
cssRuleGenerator(cssSnippet) {
    //选择器 及 cssbody 匹配
    let cssRegex = /([^\{\}]+?){([^\{\}]+)}/ig;
    let result;
    let stylesheet = [];
    while ((result = cssRegex.exec(cssSnippet))) {
      console.log("regex: ", result);
      let holeMatch = result[0];
      //分组拿到选择器
      let selector = result[1];
      //分组拿到cssBody
      let cssRuleBody = result[2];
      let selectorList = selector;
      selectorList = selectorList.trim();
      let groups = selectorList.split(",");
      for (let i = 0; i < groups.length; i++) {
        let seletorTex = groups[i].trim();
        stylesheet.push(
          new CSSStyleRule(
            holeMatch.trim(),
            seletorTex.trim(),
            cssRuleBody.trim()
          )
        );
      }
    }
    return stylesheet;
  }
```
成熟的方案是postcss,postcss有方法能解析cssstr
* * * 

## 四.应用CSS样式到DOM中

先不考虑css属性和伪类伪元素的情形，只考虑类，id,后代，兄弟等常用选择器，那么问题是：
如何把样式应用到对应的node节点中？
对于这个问题，可以拆分成以下的小问题
1.如何匹配css规则到dom 特别是像有后代选择器，兄弟选择器的情形，如何正确匹配？
2.如何保证迭代效率 试想遇到嵌套层级很深的css样式，如果不匹配，又要重新选择路径，大大浪费了性能
3.如何处理css优先级问题
4.css的应用顺序是从左往右还是从右往左
在回答这些问题之前，先介绍cssom的几个类实现，如下：
![cssom](//p1.meituan.net/dpgroup/6d1bed9532c62564b2608135088421e361265.png)
###### 1.CSSStyleRule
对于CSSStyleRule，有一个属性是selectorArray，css中，selector的概念，按MDN，有：
> **basic selector**:
> `Type selectors` elementname
> `Class selectors` .classname
> `ID selectors` #idname
> `Universal selectors` * ns|* *|*
> `Attribute selectors` [attr=value]

> **combine selector**:
> `basic selector[basic selector]`

> **combinator**:
> `Adjacent sibling selectors` A + B
> `General sibling selectors` A ~ B
> `Child selectors` A > B
> `Descendant selectors` A B

这些都是selector，而CSS的选择器是由selectorlist组成的
```js
selectorlist { property: value; [more property:value; pairs] }

...where selectorlist is: selector[:pseudo-class] [::pseudo-element] [, more selectorlists]

```
所以我们可以把selectorlist切开为一个数组，如对于
```
      div.class1 + span >div
```
选择器，可以解析出5个selectors
```
   ['div.class1','+','span','>','div']
```
第一个为combine selector，第二个为combinator，第三个为basic selector，后续类似。
* * *
##### 1.1 innerStyleRule
根据上一步中的selectorArray，我们可以构造出innerStyleRule。
一个innerStyleRule的内部属性如下：
每个innerStyleRule包括一个matchTypes和values数组，matchType数组中目前只会有tag,id,class三种enum类型，values数组保存tag,id,calss对应的tagName，idName，className。
innerStyleRule最重要的一个属性为relation，即combinator，combinator即为关系，包括父子，兄弟等。
innerStyleRule对象示意：
```js
{
    matchTypes:Array(enum):MATCHTYPE,
    values:Array Of String(tagName,className,idName)
    relation:enum:RELATIONTYPE,
}
```

```
    //enum of mathTypes
    export const MATCHTYPE={
        TAG:'Tag',
        ID:'id',
        CLASS:'CLASS'
    }
    //enum of relationType
    export const RELATIONTYPE={
        SUBSELECTOR:'SubSelector',//NO COMBINATOR
        DESCENDANT:'Descendant', // "Space" combinator
        CHILD:  'Child',             // > combinator
        DIRECTADJACENT:'DirectAdjacent',    // + combinator
        INDIRECTADJACENT:'IndirectAdjacent'  // ~ combinator
    }
```
以 `['div.class1','+','span','>','div']这个selectorArray `为例，可以得到一个有3个innerStyleRule元素的数组
```
[
   {
      matchType:['TAG'],
      relation:"CHILD",//>
      values:["div"]
   },
    {
      matchType:['TAG'],
      relation:"DIRECTADJACENT",//+
      values:["span"]
   },
    {
      matchType:['TAG','CLASS'],
      relation:"SUBSELECTOR",//NO COMBINATOR
      values:["div","class1"]
   },
]
```
**这个数组会在dom匹配css的过程中起到作用**，在计算权重的时候也能起到作用。
* * *
##### 2.选择器权重实现
css选择器优先级是按照下图的关系构造的，每位可以是一个16进制数（那最多只有16个，不过超过16个元素的css选择器也不建议实现）
![cssweight](https://css-tricks.com/wp-content/csstricks-uploads/specificity-calculationbase.png)
如一个例子，权重值为0x0113:
![cssweightex](https://css-tricks.com/wp-content/csstricks-uploads/cssspecificity-calc-1.png)
详细参考这篇文章(https://css-tricks.com/specifics-on-css-specificity/)
在1.1中得到的innerStyleRule得到了tag,class,id信息，根据其数据结构可以方便的计算样式的weight。
* * *
##### 3.按样式类别归类
得到的innerStyleRule数组的第0个元素为选择器最右边的元素，根据这个元素可以做一些归类操作，目的是利用hashMap，加速css的匹配过程，这部分的缓存操作能大大提升性能。
![](//p0.meituan.net/dpgroup/fa4f4b69de855315228ead08424ebc0c202319.png)
如图所示，左上方是有html及对应的css，css按从上到下从1到6编号，右侧是按weight进行排过序的css样式，
在样式解析过程中，我们会根据selectorlist最右侧的选择器将 CSS 规则添加到某个哈希表中，如图下侧。这些哈希表的选择器各不相同，包括 ID、类名称、标记名称等，还有一种通用哈希表，适合不属于上述类别的规则。如果选择器是 ID，规则就会添加到 ID 表中；如果选择器是类，规则就会添加到类表中，如图所示。
实现为：
```
   //在创建CSSStyleRule时
   //对内部序列选择器中最右的进行判断，存入全局样式缓存中，主要用于应用样式时的加速
   //CSSRULEHASHMAP为缓存HashMap
    let rightRule = this.innerStyles[0];
    if (rightRule.matchType.indexOf(MATCHTYPE.ID) >= 0) {
      CSSRULEHASHMAP.ID.push(this);
    }
    if (rightRule.matchType.indexOf(MATCHTYPE.CLASS) >= 0) {
      CSSRULEHASHMAP.CLASS.push(this);
    }
    if (rightRule.matchType.indexOf(MATCHTYPE.TAG) >= 0) {
      CSSRULEHASHMAP.TAG.push(this);
    } else {
      CSSRULEHASHMAP.UNKNOW.push(this);
    }
//这部操作为了解析时候的快速命中，试想在创建一个node，如果一个没有id,class属性的html标签node被创建，那么对它应用的样式应该忽略含有id,class的样式
```
之后这个缓存Map会在匹配过程中用到。

* * *

### 4.2 应用CSS样式到DOM中
以解析下图html与css为例： 我们在构建每一个HTMLElement的时候,去看看这个Node符不符合这6个样式，如果符合，即应用这些样式到HTMLElement上。
![](//p0.meituan.net/dpgroup/fa4f4b69de855315228ead08424ebc0c202319.png)
这个时候可以回答之前提到的问题，如何处理CSS优先级。我们可以把所有相关的样式按权重从小到大排序，按图右侧。
在应用样式时，我们先从小到大将低权重样式应用到HTMLElement上，然后用数组后面的高权重的样式去覆盖之前的样式（如果有相同的就覆盖，如果没有相同的就新获得了一个样式）这样依次将整个符合的样式组应用到HTMLElement中。
如下图，当解析到<div class="dp1">节点时，1，2号样式符合，之后从后往前应用样式，依次覆盖，margin被覆盖掉。观察这个<div class="dp1">节点，发现其没有id属性，所以在遍历6个样式的时候，4号样式是可以不用关心的，因为4号样式为#id选择样式，必定跟当前的<div class="dp1">节点不会匹配，所以之前的HashMap就派上用场，我们从hashMap中取出classMap和tagMap就好，这样，遍历的时候只需要遍历5个样式。这样能节约很大一部分的匹配过程，按官方介绍，能提升95%的性能。这就回答了一开始的如何提升性能的问题。
![](http://p1.meituan.net/dpgroup/7aaa30ccee5242efdf0706604da613e6169092.png)
之后解析到<span class="sp1">，需要取出5个样式(没有id)，之后符合一个样式，应用其在dom上。
![单独应用样式](//p1.meituan.net/dpgroup/225e8e7170e342dee2bb31daf342b679114740.png)

如下图，解析到<span id="s1">，需要取出4个样式(没有class)，之后符合两个样式，依次应用其在dom上。

![](//p1.meituan.net/dpgroup/2536b4d14a47e3927f0a85fc1e71bf6b74055.png)

同理，遇到<span>只需要取出3个tagHashMap，之后6匹配成功。
![](//p0.meituan.net/dpgroup/354df822931bb81d46b636ecc0e7345291776.png)

然后回到第一个小问题，css样式如何匹配dom？先考虑下面的例子
![](//p0.meituan.net/dpgroup/0254a5688716623456c9ba82892809b2137111.png)
先看图中的最右测的innerStyleRule数据结构，对于6号样式，为
```
.div1 div.div2 .div3 ~ span{
  font-size:20px;
}
```
其对应的innerStyleRules表达为：
```
[
   {
      matchType:['TAG'],
      relation:"INDIRECTADJACENT",//~
      values:["span"]
   },
    {
      matchType:['CLASS'],
      relation:"DESCENDANT",//space
      values:["div3"]
   },
    {
      matchType:['TAG','CLASS'],
      relation:"DESCENDANT",//space
      values:["div","div2"]
   },
   {
      matchType:['CLASS'],
      relation:"SUBSELECTOR",//nothing
      values:["div1"]
   }
]
```
接下来看看如图HTML部分解析到的<span>是如何匹配成功的。首先在解析HTML文本之前，我们先把CSS的文本解析为了CSSOM，之后开始解析HTML文本，现在解析器解析到了<span>，首先看innerStyleRules数组第一个元素（即为最右侧selector），type为['TAG']，值为['span']，当前HTMLElement为<span>，符合要求，接下来看relation，为`INDIRECTADJACENT`（`~`之后选择器）表示选择之后的兄弟节点，那么我们可以反过来看，如果找到一个满足条件的<span>前面的兄弟节点，那么这一级的`~`是成功的，由于<span>前面的dom已经构建好，因此找到**第一个**满足条件的前面的兄弟节点，即`~`这个选择器关系成立，我们可以观察<span>的previousSibling，以及<span>的previousSibling的previousSibling，逐次判断，如果某个previousSibling（HTMLElement）满足innerStyleRules[1]的条件，即为：
```
  {
      matchType:['CLASS'],
      relation:"DESCENDANT",//space
      values:["div3"]
   }
```
有class,且className为div3，则INDIRECTADJACENT（~）匹配成功，这里匹配到<div class='div3'>。之后以<div class='div3'>为匹配HTMLElement，其  relation为`DESCENDANT`，那么我们找到第一个祖先满足条件（div且有class为div2）：
```
  {
      matchType:['TAG','CLASS'],
      relation:"DESCENDANT",//space
      values:["div","div2"]
   }
```
 找到了<div class='div2'>是其祖先且满足条件，之后其relation也是`DESCENDANT `祖先选择器，同理找到了<div class="div1">。找祖先的过程可以不断的拿parent属性进行匹配，如<div class='div2'>.parent，<div class='div2'>.parent.parent，<div class='div2'>.parent.parent.parent，这样一直下去，直到根节点为止。

在解析HTML过程中，我们每次new 一个HTMLElement，便执行一次embedCsstoHtmlNode，使得样式嵌入到HTMLElement中。
```
            function embedCsstoHtmlNode(node) {
              //从样式hashMap中获得能用上的样式
              let wantedStyles = _constructTheWantedStyleSheets(node);
              //按照权重对sheet进行排序
              quickSort(wantedStyles);
              //根据node有没有id,有没有class  去拿HASHMAP的值 TAG，UNKNOW一定要拿，ID,CLASS如果NODE没有，可以不拿
              for (let i = 0; i < wantedStyles.length; i++) {
                wantedStyles[i]._resetCursor();
                if (matchTheCssStyle(wantedStyles[i], node)) {
                  if (!node.style) node.style = {};
                  _overrideNodeStyle(node, wantedStyles[i].style);
                }
              }
            }
            
            
            //如果node有id,class才会并入需要的样式
            //CSSRULEHASHMAP是styleRule的hashMap，分ID,CLASS,TAG,UNKNOW 4个维度
            function _constructTheWantedStyleSheets(node) {
              let styleSheets = [];
              if (node.getAttribute("id")) {
                styleSheets = styleSheets.concat(CSSRULEHASHMAP.ID);
              }
              if (node.getAttribute("class")) {
                styleSheets = styleSheets.concat(CSSRULEHASHMAP.CLASS);
              }
              //TAG是一定要得，需要过滤的是没有class 和 id的情况
              styleSheets = styleSheets.concat(CSSRULEHASHMAP.TAG);
              styleSheets = styleSheets.concat(CSSRULEHASHMAP.UNKNOW);
              return styleSheets;
            }
```
先从样式hashMap中选择出一部分能匹配中的，之后对这些样式按权重进行排序，然后从第0个开始迭代，这里排序使用了快速排序，之后按照权重从低到高依次调用matchTheCssStyle方法，对node与style匹配成功的即可以进行样式写入，注意在每次样式匹配前调用了 wantedStyles[i]._resetCursor()重置游标方法，这个目的是为了清理在判断style是否与node匹配的过程中产生的内部状态。
`matchTheCssStyle`方法用以判断HTMLElement是否与一个CSSStyleRule匹配，`matchTheCssStyle`需要判断CSSStyleRule.innerstyleRules中所有的innerstyleRule都匹配。
因而在`matchTheCssStyle`方法中我们需要一个判断单个innerstyleRule是否匹配某个HTMLElement的方法:`_isMatchNodeIndex(cssRule, node, index)`按游标进行匹配，如果innerStyleRules中某个游标下的innerStyle与HTMLElement及其属性匹配（主要idName,tagName,className对应到HTMLElement中的idName,tagName,className是否对应匹配，这个方法只是比较tagName，idName等，与关系无关）则返回true。
`matchTheCssStyle`的匹配过程为获得当前innerStyleRules游标对应关系，游标初始为0，由于innerStyleRules按从右到左保存，0对应选择器列表中最右的selector，对应上例中选择的是：
```
   {
         matchType:['TAG'],
         relation:"INDIRECTADJACENT",//~
         values:["span"]
   }
```
如果HTMLElement为span标签，则匹配成功，更新游标信息，之后又去找其第一个匹配兄弟（如果是DESCENDANT，则为找第一个祖先），看是否匹配，如果匹配，则以这个匹配的HTMLElement递归下去。

```
function matchTheCssStyle(cssStyleRule, node) {
  //获得当前innerStyles游标对应关系，游标初始为0
  //由于innerStyles按从右到左保存，0对应选择器列表中最右的selector
  let RuleCombinator = cssStyleRule.innerStyles[cssStyleRule.cursor].relation;
  switch (RuleCombinator) {
    case RELATIONTYPE.DESCENDANT: //后代
      /*
        如innerStyle为
         {
         matchType:['TAG'],
         relation:"CHILD",//>
         values:["div"]
         },
         则当前node为div就算匹配成功
         从右往左开始，如果当前游标对应innerStyles 与 node信息不匹配，返回false
         */
      if (!_isMatchNodeIndex(cssStyleRule, node, cssStyleRule.cursor)) return;
      //进到上一级游标
      cssStyleRule.cursor++;
      //对父元素进行匹配
      let parent = node.parent;
      if (!parent) return;
      //游标已更新
      while (!_isMatchNodeIndex(cssStyleRule, parent, cssStyleRule.cursor)) {
        //由于是后代选择器，找到第一个满足条件的父亲或祖先
        parent = parent.parent;
        //如果找到root根了还没找到，返回false
        if (!parent) return;
      }
      //到这里则最近的祖先找到了
      if (cssStyleRule._isCursorEnd()) {
        return true;
      }
      //祖先匹配到了,并且游标没结束 递归这个找到的祖先元素
      return matchTheCssStyle(cssStyleRule, parent);

    case RELATIONTYPE.SUBSELECTOR: //最左
      //已经是最左，只要返回是否匹配
      return _isMatchNodeIndex(cssStyleRule, node, cssStyleRule.cursor);

    case RELATIONTYPE.CHILD: //儿子
      //>
      if (!_isMatchNodeIndex(cssStyleRule, node, cssStyleRule.cursor)) return;
      cssStyleRule.cursor++;
      let parentC = node.parent;
      if (!parentC) return;
      if (!_isMatchNodeIndex(cssStyleRule, parentC, cssStyleRule.cursor))
        return;
      if (cssStyleRule._isCursorEnd()) {
        return true;
      }
      return matchTheCssStyle(cssStyleRule, parentC);
    case RELATIONTYPE.DIRECTADJACENT: //同胞
      //+
         ……………………
    case RELATIONTYPE.INDIRECTADJACENT: //兄弟
      //~ find all pre
         ……………………
  }
}

```
到这里就可以回答之前的问题，为什么要从右往左开始？
因为从右往左开始不需要依赖没解析到的node,当stylesheets已经生成好了之后，再进行dom树的解析过程中，每遇到一个node,不论是后代关系，还是兄弟关系，在创建这个node的时候，这个关系就已经确立了的，根本原因是
dom树的解析过程是从上到下，从左到右的，或是以解析<>开标签为首要任务的，解析的这个node如果是某个node的后代，那么只要看这个node的parent，往上看，如果是某个node的兄弟，也只需要看previousSibling，这些node在构造这个node的时候已经生成
试想如果是从左往右的话，比如#id1 div，那么如果在构造#id1这个node的时候，就要等其后代们都构造完了，才能把样式用于这些后代，所以要等后代构造完，即是先要构造dom树，再把style，应用上去，这样做会造成屏幕白屏，而且有不必要的树遍历过程
构造dom树，算是一次遍历，应用样式到dom树又有一次遍历，性能会有影响
而从右到左只有一次遍历过程，即为边构建dom树边应用样式，同步进行。
* * * 
## 五.事件系统
在事件系统中，主要有几个问题？
1.dom的addEventListener API怎么实现？
2.e.stopPropagation/e.stopImmediatePropagation怎么实现？
3.dom的dispatchEvent API怎么实现？

按mdn描述：
事件响应会有一个Event对象，用于保存事件属性，包括事件冒泡及捕获过程中的状态
[常用事件参考表]:(https://developer.mozilla.org/en-US/docs/Web/Events)
先来看看[事件对象的mdn定义](https://developer.mozilla.org/zh-CN/docs/Web/API/Event)，我们实现如下：
```
export const EVENTPHASE = {
  BUBBLE: 1,
  CAPTURE: 2,
  TARGET: 3
};

export default class Event {
  // event = new Event(typeArg, eventInit);
  /*
     typeArg
     Is a DOMString representing the name of the event.
     eventInit Optional
     Is an EventInit dictionary, having the following fields:
     "bubbles": (Optional) A Boolean indicating whether the event bubbles. The default is false.
     "cancelable": (Optional) A Boolean indicating whether the event can be canceled. The default is false.
     "scoped": (Optional) A Boolean indicating whether the given event bubbles. If this value is true, deepPath will only contain a target node.
     "composed": (Optional) A Boolean indicating whether the event will trigger listeners outside of a shadow root. The default is false.
     */
  /*
     var event = new MouseEvent('click', {
     'view': window,
     'bubbles': true,
     'cancelable': true
     });
     */
  constructor(typeArg, eventInit) {
    //this.eventPhase enum
    this._isStopPropagation = false;
    this._isStopImmediatePropagation = false;

    this.type = typeArg;
    //todo default config
    this.option = eventInit || {
      bubbles: true,
      cancelable: true
    };
    this.bubbles = this.option.bubbles;
    this.timeStamp = +new Date();
    //该属性总是指向被绑定事件句柄（event handler）的元素
    this.currentTarget;
    this.target;
    this.eventPhase;
  }
  preventDefault() {
    //not stop bubbling
  }
  stopPropagation() {
    if (this.option.cancelable) {
      this._isStopPropagation = true;
    }
  }
  stopImmediatePropagation() {
    //停止冒泡 且停止队列
    if (this.option.cancelable) {
      this._isStopImmediatePropagation = true;
    }
  }
  //event.initEvent(type, bubbles, cancelable);
  /*
     type
     Is a DOMString defining the type of event.
     bubbles
     Is a Boolean deciding whether the event should bubble up through the event chain or not. Once set, the read-only property Event.bubbles will give its value.
     cancelable
     Is a Boolean defining whether the event can be canceled. Once set, the read-only property Event.cancelable will give its value.
     */
  initEvent(type, bubbles, cancelable) {
    //old style
    this.type = type;
  }
}

```
 

[对于旧标准var event = Document.createEvent(),event.initEvent()](https://developer.mozilla.org/zh-CN/docs/Web/API/Event/initEvent)的事件创建方式，我们先不实现，我们使用新的事件创建形式，即为通过[构造函数形式创建事件](https://developer.mozilla.org/en-US/docs/Web/API/Event/Event)
关于事件流程：
> [创建事件与触发事件参考](https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Creating_and_triggering_events)
> [关于事件系统中target 与 currentTarget 的区别](https://developer.mozilla.org/en-US/docs/Web/API/Event/Comparison_of_Event_Targets)

首先我们来实现addEventListener， 基于chrome的实现思路，我们将事件监听函数存入一个全局hashMap中，并把每个node作为key,value为这个节点对应的事件集合。如图所示：
![](//p0.meituan.net/dpgroup/6c68614a2d7e7753795f97417e7fcc6632159.png)
这个实现与V8的实现有一些区别，V8是每个Key对应一个vector,这边是每个key对应一个hashMap 不过都能实现，不影响理解。我们实现如下:
```
  addEventListener(eventName, fn, useCaputrue) {
    let domKey = MD5(this);
      //在V8是用指针即内存地址来作为KEY，这样能保证KEY的唯一
    if (!EVENTMAP[domKey]) EVENTMAP[domKey] = {};
    let listenersQueue = EVENTMAP[domKey] && EVENTMAP[domKey][eventName];
    if (!listenersQueue) EVENTMAP[domKey][eventName] = [];
    EVENTMAP[domKey][eventName].push({
      fn: fn,
      useCaputrue: useCaputrue
    });
  }
```
这就回答了第一个问题。

现在来实现dispatchEvent，按照定义dispatchEvent，应该有以下的方法接口
```
     var cancelled = !elem.dispatchEvent(event);
```
值得一提的是，我们只实现W3C事件，原生事件是通过层层封装的，如下图
![原生事件封装成W3C事件](http://pic1.zhimg.com/v2-35c29bd029f5997ffdfb60dcd5023fac_b.png)
考虑dispatchEvent事件之后，事件要进行捕获和冒泡，因而要得到冒泡捕获路径，路径可以不断遍历parent属性来得到。
![冒泡捕获](https://javascript.info/article/bubbling-and-capturing/eventflow@2x.png)
如上图，我们先进行事件捕获，到达target之后，再进行事件冒泡，实现如下：
 ```
//分发事件
 dispatchEvent(event) {
    event.target = this;
    let nodeChain = _getNodeChain(this);
    //先捕获
    let captureNotCancel = _eventCaptrueTraverse( event, nodeChain);
    // event 这两个event不是要分开成2个单独的，因为如果第一个stopProgation之后，第二个也用的这个event
    //后冒泡
    let bubbleNotCancel = _eventBubbleTraverse(event, nodeChain);
    // return NotCancel
    return captureNotCancel && bubbleNotCancel;
  }
//获得路径
function _getNodeChain(node) {
  let nodeChain = [];
  nodeChain.push(node);
  let parent = node.parent;
  while (parent) {
    nodeChain.push(parent);
    parent = parent.parent;
  }
  return nodeChain;
}

//冒泡流程
function _eventBubbleTraverse( event, nodeChain) {
    if(!event.bubbles)return true;
  for (let i = 0; nodeChain && i < nodeChain.length; i++) {
    //如果stopPropagation之后，flag为true，停止传播
    if (event._isStopPropagation) return false;
    if (i == 0) {
      event.eventPhase = EVENTPHASE.TARGET;
    } else {
      event.eventPhase = EVENTPHASE.BUBBLE;
    }
    let nodec = nodeChain[i];
    let nodeKey = MD5(nodec);
    let queues = EVENTMAP[nodeKey] && EVENTMAP[nodeKey][event.type];
    for (let j = 0; queues && j < queues.length; j++) {
      let fn = queues[j].fn;
      let useCaputrue = queues[j].useCaputrue;
      event.currentTarget = nodec;
       //如果stopImmediatePropagation之后，立即停止传播
      if (event._isStopImmediatePropagation) return false;
      if (!useCaputrue) {
        fn.call(nodec, event);
      }
    }
  }
  return true;
}
```
我们通过parent属性得到链路数组，之后，从前到后，并从后到前循环这个数组，完成捕获与冒泡过程，如果中途有stopPropagation/stopImmediatePropagation则置位flag，停止事件流程，这样就回答了后面2个问题。
* * * 
## 六.总结
到此基本实现了一遍html的简单解析器，但是还有好多没做，比如removeNode API， 并如何处理removeNode之后的内存空间，其事件响应函数还没有被remove，DOM Level 0事件，属于属性事件，需要特殊处理，需要getter，setter等，相关的代码地址后续会贴出来。
参考资料：
[从Chrome源码看浏览器如何构建DOM树](https://zhuanlan.zhihu.com/p/24911872)
[从Chrome源码看浏览器如何计算CSS](https://zhuanlan.zhihu.com/p/25380611)
[从Chrome源码看浏览器的事件机制](https://zhuanlan.zhihu.com/p/25095179)
[浏览器的工作原理：新式网络浏览器幕后揭秘](https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork/)


 dom深度尽量浅。
不要为id选择器指定类名或是标签，因为id可以唯一确定一个元素。
不要给类选择器指定标签，类，代表具有一类属性的标签，不仅是一个，虽然可以实现，但是降低了效率。
避免后代选择符，尽量使用子选择符。原因：子元素匹配符的概率要大于后代元素匹配符。后代选择符;#tp p{} 子选择符：#tp>p{}
避免使用通配符，举一个例子，.mod .hd *{font-size:14px;} 根据匹配顺序,将首先匹配通配符,也就是说先匹配出通配符,然后匹配.hd（就是要对dom树上的所有节点进行遍历他的父级元素）,然后匹配.mod,这样的性能耗费可想而知.
选择器不要超过4层（在scss中如果超过4层应该考虑用嵌套的方式来写）；