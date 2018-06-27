---
title: 用js实现一门简单语言
date: 2018-02-27 21:20:52
tags: 编译 前端
---
## 1.前言
假如我们要来实现一门语言，语言有int,string,char,bool等类型，写其他类型会报错
用`~`表示字符串，支持常见表达式，for循环，条件语句，注释等，每个语句需要`;`结束
且函数定义用关键字
`def`,类似于js的function,函数调用利用invoke关键字，如下面的声明了一个函数`logParam`,
接收一个参数，函数体调用函数log，入参是a
```js
  def logParam(int a){        
        invoke(log,a);
    }
```

示例代码
[parse_compiler](https://github.
com/klfzlyt/parse_compiler)

## 2.目标
给入下面一段程序片段，我们希望用js写出编译器，得到对应的结果
```js
  //定义log
    def logParam(int a){        
           invoke(log,a);
    }
    //定义累加
    def accu(int begin,int end,int skip){
        int accu= 0,a;
        for(a=begin;a<=end;a=a+1){
            if(a == skip ){

            }
            else{
                accu = accu+a;     
            }
        }
        return accu;
    }
    // 空格需要
    string a = ~iAmString~ ;
    int b = invoke(accu,0,50,0);
    invoke(log,a,b);
    invoke(logParam,123421212,a);
    //从0加到100，跳过10
    invoke(log,invoke(accu,0,100,10));
```


## 3.词法分析
这一步主要是分析出程序片段的token，即单词片段，意为最细粒度单词单元
对于上面的语言，我们需要定义一些概念，哪些是关键字，哪些是类型定义
我们定义一些静态tag，除开符号`; ) (`等以外，所有的有意义token都有对应的tag
```js
export default class Tag {
  static BASIC = 'BASIC';
  static IF = 'IF';
  static FOR = 'FOR';
  static ELSE = 'ELSE';
  static WHILE = 'WHILE';
  static AND = 'AND';
  static OR = 'OR';
  static LE = 'LE';
  static LEE = 'LEE';
  static GEE = 'GEE';
  static NOT = 'NOT';
  static GE = 'GE';
  static MINUS = 'MINUS';
  static DO = 'DO';
  static ID = 'ID';
  static NUM = 'NUM';
  static SWITCH = 'SWITCH';
  static CASE = 'CASE';
  static ASSIGN = 'ASSIGN';
  static BREAK = 'BREAK';
  static REAL = 'REAL';
  static EQUAL = 'EQUAL';
  static NE = 'NE';
  static TRUE = 'TRUE';
  static FALSE = 'FALSE';
  static NOTE = 'NOTE';
  static COMMENT = 'COMMENT';
  static DIVIDE = 'DIVIDE';
  static DIVIDEE = 'DIVIDEE';
  static ADD = 'ADD';
  static SUB = 'SUB';
  static MUTIPLE = 'MUTIPLE';
  static RETURN = 'RETURN';
  static DEF = 'DEF';
  static INVOKE = 'INVOKE';
}

```

再来定义单词   
下面定义了部分单词，有意义的token都可以作为word
```js
import Token from './token.js';
import TAG from './tag.js';
import Tag from './tag.js';

export default class Word extends Token {
  constructor(value, tag) {
    super(tag);
    this.value = value;
  }
  static and = new Word('&&', TAG.AND);
  static or = new Word('||', TAG.OR);
  static less = new Word('<', TAG.LE);
  static greater = new Word('>', TAG.GE);
  static greater_equal = new Word('>=', TAG.GEE);
  static less_equal = new Word('<=', TAG.LEE);
  static equal = new Word('==', TAG.EQUAL);
  static assign = new Word('=', TAG.ASSIGN);
  static not = new Word('!', TAG.NOT);
  static not_equal = new Word('!=', TAG.NOTE);
  static divide = new Word('/', TAG.DIVIDE);
  static divide_equal = new Word('/=', TAG.DIVIDEE);
  //保留字
  static if = new Word('if', TAG.IF);
  static for = new Word('for',Tag.FOR);
  static else = new Word('else', TAG.ELSE);
  static while = new Word('while', TAG.WHILE);
  static do = new Word('do', TAG.DO);
  static switch = new Word('switch', TAG.SWITCH);
  static true = new Word('true', TAG.TRUE);
  static false = new Word('false', TAG.FALSE);
  static def = new Word('def', TAG.DEF);
  static invoke = new Word('invoke',TAG.INVOKE);
  static return = new Word('return', TAG.RETURN);

  toString() {
    return this.value;
  }
}

export const RESERVEDWORDS = [
  Word.if,
  Word.else,
  Word.while,
  Word.do,
  Word.switch,
  Word.true,
  Word.false,
  Word.def,
  Word.invoke,
  Word.for,
  Word.return
].map(ele => {
  return ele.toString();
});
```


再来定义类型,类型作为单词的子类，也是单词的一种
我们定义int，bool等类型
```js
import Word from './word.js';
import Tag from './tag.js';
export default class Type extends Word {
	constructor(value, length) {
		super(value, Tag.BASIC);
		this.value = value;
		this.length = length;
	}
	static int = new Type('int', 1);
	static bool = new Type('bool', 1);
	static float = new Type('float', 8);
	static char = new Type('char', 1);
	static string = new Type('string', 8);
	toString() {
		return this.value;
	}
}
export const TYPEWORDS = [Type.string , Type.int, Type.bool, Type.float, Type.char].map(ele => ele.toString());

```
#### 分析
分析token比较简单，我们从程序的第一个字符开始，按照从左至右，从上到下的顺序，依次输入进词法分析器
1.遇到字母，即保存，遇到第一个非标识符组成字符就停止，如果得到的一串是保留字，就new一个保留字的token,如果不是保留字，那么作为标识符保存，遇到`, ; ） （ {`等就作为一个普通的token保存
2.如果遇到数字，如果遇到`.`或者数字都可以进行下去，如果数字后遇到`.`,说明是小数，可以保留下来用`parseFloat`处理
3.如果遇到`/`,那么后续有可能是`/= // /*`所以要根据后一个字符来判断是什么token
如果是`// /*`前者需要正则匹配到换行，更改游标，后者需要匹配第一个`*/`，再更改游标
4.类似的还有`>= <= != == || &&`都要匹配到`> < ! = | &`再向后看一个字符
5.遇到空格，换行 我们可以不做处理，直接绕过

之后我们可以得到一系列token串

```js

        
            function addA(d) {
            return a + d;
            }
      
```
![](//p0.meituan.net/dpgroup/c46808e228205df3be6171d94607b056102091.jpg)


以分析下面的函数定义为例
运行我们的词法分析，得到
```js
    def logParam(int a){        
           invoke(log,a);
    }
```
能得到下面的一些token
![](//p1.meituan.net/dpgroup/923be195dd111235058632453d98807d335069.jpg)


## 3.语法分析


```js        
    function addA(d) {
        return a + d;
    }
```

一个函数声明的语法树
![](https://p1.meituan.net/dpgroup/5e74e7017aebae4043bae57e586578a3100615.jpg)

每一个语句，都有具体的组成部分
我们先把语言拆开 可以得到下面的parse
有些是parse一个表达式，有些是parse一个标识符，有些是parse一个类型等，
还需要知道ast节点，如函数调用是，有callee和arguments两部分
```js
export class CallExpression {
  // readonly type: string;
  // readonly callee: Expression | Import;
  // readonly arguments: ArgumentListElement[];
  constructor(callee, args) {
    this.type = Syntax.CallExpression;
    this.callee = callee;
    this.arguments = args;
  }
}
```

我们还要定义一些移动函数，用以移动token，输入进入语法解析器
我们在移动过程中跳过注释  
```js
move() {
    // debugger
    do {
      this.index++;
      this.lookahead = this.tokens[this.index] || {};
    // 我们在移动过程中跳过注释  
    } while (this.lookahead.tag === Tag.COMMENT);
  }

  error(s) {
    throw new Error(
      'near line : ' + s + ' ' + this.lookahead.value || this.lookahead.tag
    );
  }

  match(t) {
    if (this.lookahead.tag == t) {
      // 匹配成功 移动
      this.move();
      return true;
    } else {
      this.error(
        'syntax error at ' + this.lookahead.value || this.lookahead.tag
      );
    }
  }
  next() {
    return this.tokens[this.index + 1] || {};
  }
```



把这些基础parse模块结合起来，就可以得到ast
parseFactorExpression
parseFunctionDeclaration
parseReturnStatement
parseForStatement
parseStatement
parseCallExpression
parseScript
parseBlock
parseStatementListItem
parseUnaryExpression
parseMutilOrdivideExpression
parseAssignmentExpression
parseArithExpression
parseRelationExpression
parseEqulityExpression
parseAndExpression
parseOrExpressgion
parseBinaryExpression
parseConditionalExpression
parseType
parseVariableDeclaration
parseIfStatement


关于表达式的解析，我们这样来
我们规定三元运算符的优先级最低，因此假如出现一个三元表达式，那么所处的位置应该位于ast树的根附近，我们采用深度优先遍历 depth-first
三元表达式在表达式中的优先级最低

关于二元表达式
//parseBinaryExpression
可以由 || && == != 等，定义一个优先级，即为谁先该进行计算，由于是深度优先，越远离根节点的节点，其优先级越高， 因为他们将得到优先计算。
因此对于`()`中的部分，优先级最高，也应该放到叶子节点中
对于 a||b&&c
我们规定优先级从低到高
1. || 
1. &amp;&amp;
1. != == 
1. &lt; &lt;= &gt; &gt;= 
1. 减号 +
1. / *
1. ! - (单目) 
1. 数字 标识符 保留字 函数调用 字符串 括号中部分
因此我们在解析过程中，遇到||，就解析左右部分，因为其优先级最低，如果解析完毕，又遇到||，就把开始解析得到的表达式作为||的左操作符，继续去解析右边
我们按照优先级从低到高，写作
```js
 parseOrExpressgion() {
    //Or的优先级最低，先去解析它的下一个优先级And
    let left = this.parseAndExpression();
    while (this.lookahead.tag == Tag.OR) {
      this.move();
      left = new Nodes.BinaryExpression('||', left, this.parseAndExpression());
    }
    return left;
  }
  parseAndExpression() {
    //And的优先级比==，!=低，先去解析它的下一个优先级==，!=
    let left = this.parseEqulityExpression();
    while (this.lookahead.tag == Tag.AND) {
      this.move();
      left = new Nodes.BinaryExpression(
        '&&',
        left,
        this.parseEqulityExpression()
      );
    }
    return left;
  }
```
就可以按照优先级依次往下写
对于单目运算 要注意，由于单目可以反复出现，如!!!!!!a，这样的话要一直递归解析单目，直到非单目出现为止
最后到了优先级最高的部分
数字 标识符 保留字 函数调用 字符串 括号中部分(还是表达式)
```js
parseUnaryExpression() {
    if (this.lookahead.tag == '-') {
      this.move();
      //递归解析单目
      return new Nodes.UnaryExpression('-', this.parseUnaryExpression());
    } else if (this.lookahead.tag == '!') {
      this.move();
      //递归解析单目
      return new Nodes.UnaryExpression('!', this.parseUnaryExpression());
    } else return this.parseFactorExpression();
  }
```
对于`()`，里面的还是表达式，我们就可以从parseConditionalExpression最低优先级的开始
最后的最高优先级到了树的叶子节点部分
```js
  parseFactorExpression() {
    let expr;
    switch (this.lookahead.tag) {
      case '(':
        this.move();
        expr = this.parseConditionalExpression();
        this.match(')');
        return expr;
      case Tag.NUM:
      case Tag.REAL:
      case Tag.TRUE:
      case Tag.FALSE:
        expr = new Nodes.Literal(this.lookahead.value, this.lookahead.value);
        this.move();
        return expr;
      case '~':
        this.match('~');
        expr = new Nodes.Literal(
          this.lookahead.value.toString(),
          this.lookahead.value
        );
        this.match(Tag.ID);
        this.match('~');
        return expr;
      case Tag.ID:
        let id = new Nodes.Identifier(
          this.lookahead.value || this.lookahead.tag
        );
        this.move();
        return id;
      case Tag.INVOKE:
        return this.parseCallExpression();
      default:
        this.error('syntax error');
        return expr;
    }
  }
```

表达式树
```js
'lyt+2323&&(33223-22||(ff)||fe<---3332-3&&few==21&&feww==3223||ffffff==3232||!f&&f>=21)';
```
![](http://p0.meituan.net/dpgroup/d1dfd8ef3eda9de1d63f73bca69cc673200345.png)

程序的stmt一般由表达式，或者只有表达式组成(这种情况叫表达式语句 ExpressionStatement)，
除了循环(forStatement)，条件(ifStatement)，声明(variableDeclaration)，一般的语句都是ExpressionStatement
所以解析了表达式，工作就完成了一大半
在解析完表达式后就可以处理，一些Statement，只要按照固定的格式走就可以了
如：parseForStatement
```js
  parseForStatement() {
    this.match(Tag.FOR);
    this.match('(');
    let init = this.parseAssignmentExpression();
    this.match(';');
    let test = this.parseConditionalExpression();
    this.match(';');
    let update = this.parseAssignmentExpression();
    this.match(')');
    let body = this.parseBlock();
    return new Nodes.ForStatement(init, test, update, body);
  }
```
如parseBlock,Block中有statement列表，用个数组保存
```js
parseBlock() {
    //const node = this.createNode();
    this.match('{');
    const block = [];
    while (true) {
      if (this.lookahead.tag == '}') {
        break;
      }
      let stmt = this.parseStatementListItem();

      stmt && block.push(stmt);
    }
    this.match('}');
    return new Nodes.BlockStatement(block);
  }
```

声明一般有变量声明，函数声明；


## 4.遍历
一般的parse都有一个traverser，用来遍历ast树，并在特定节点做操作，如用户想在遍历到`CallExpression`时进行一些操作，把log变成console.log，可以如下操作
```js
traverser(ast, {
  CallExpression: {
    enter: function(node, parent) {
      // 改变ast    
      if (node.callee == 'log') {
        // 把log变为console.log
        node.callee = `console.log`;
      }
    },
  },
});
```
`enter`表示进入到ast这个节点的时刻，这个时刻traverser还不知道这个节点的children的信息，只是到这个节点，它的后续节点信息不知道，只有当`exit`方法,也就是再次回到这个节点，traverser知道了这个节点的所有字节点信息

## 5.代码生成
`codeGenerator`根据ast来生成代码，我们对每一个ast节点，写一个对应的代码生成即可
比如
递归的调用codeGenerator就好
```js
  case 'CallExpression':
      return (
        node.callee + '(' + node.arguments.map(codeGenerator).join(', ') + ')'
      );
  case 'ReturnStatement':
      return `return ${codeGenerator(node.argument)};`;
  case 'ConditionalExpression':
      return `${codeGenerator(node.test)} ? ${codeGenerator(
        node.consequent
      )}:${codeGenerator(node.alternate)} `;
  case 'AssignmentExpression':
      return `${node.left.name}${node.operator}${codeGenerator(node.right)}`;    
```

## 6.参考资料
http://resources.jointjs.com/demos/javascript-ast
https://astexplorer.net/#
https://the-super-tiny-compiler.glitch.me/traverser