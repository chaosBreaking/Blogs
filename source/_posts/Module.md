---
title: Note — Node.js设计模式(三)
date: 2018/10/3 22:22:22
tags:
  - Node.js
  - Module
categories: Node.js
---

# require函数

##构成  

* resolve	模块路径解析函数，根据传入的模块名来解析出完整路径。
* main
* extensions
* cache    模块缓存，以模块的**完整路径**和**该模块的Module对象**作为K-V存储。
<!-- more -->
##导入模块

1.接受模块名称作为输入，使用require.resolve()来完成路径解析

2.如果模块已经在缓存中，则立刻返回(require.cache里模块的路径<id>作为键的对象)

3.如果模块未被加载，需要为首次加载设置运行环境。将会创建一个Module对象，其中包含用控对象字面量初始化的exports属性。此属性将用于模块的代码到处公共API。

4.Module对象被缓存

5.模块源代码被读取和评估,将module.exports的内容表示模块的API，返回给调用者

## module

定义一个模块，要导出的部分需要被分配给module.exports，不然该模块下的内容都是私有的。但是，模块系统暴露了global这个特殊的变量，也就是说在一个模块里对于global的操作会污染导入它的模块的global变量。



## 解析算法

解析算法可以分为以下三个部分：

* 文件模块：若以/开头，则被视为绝对路径，将按照原样返回；若以./等开头则视为相对路径

* 核心模块：如果模块名之前没有/等则会在Node.js核心模块中搜索

* 包模块：如果在核心模块中没有找到，则会在第一个node_modules中查找匹配的模块目录，它从所需模块开始在目录结构中向上导航。该算法通过超找目录书中的下一个node_modules目录继续搜索匹配，知道文件系统的根目录。

对于文件和包模块，单个文件和目录都可以与moduleName相匹配。算法将会尝试匹配以下内容：

* <moduleName>.js

* <moduleName>/index.js

* 在<moduleName>/package.json的main属性中指定的目录/文件

每个包都有自己的私有依赖，这保证了应用中不存在版本兼容性的冲突问题。

## 模块缓存

每个模块只在第一次需要时才被加载和评估，后续都将返回缓存的版本。

产生的影响：

* 使得模块依赖项中可以有循环。
* 在某种程度上保证，在给定包中需要相同的模块时总是返回相同的实例。

## 循环依赖

在a.js中引用b.js，同时在b.js里引用a.js

之后在第三方main.js里引用二者。

```javascript
//main.js
const a = require('./a.js');
const b = require('./b.js');
```

结果是，b.js里引用的a.js不完整，它的状态是到达引用b.js那一刻的状态。



# 模块定义模式

模块系统是用于定义API的工具。API设计主要是考虑私有和公有功能之间的平衡。目标是最大限度的实现隐藏信息和API可用性，同时平衡这些问题与其他软件的质量问题，如可扩展性(extensibility)和代码重用(code reuse)。

几种流行的模块定义模式

## 命名导出

暴露公共API最基本的方法，将所有要公开的值赋给由exports（或module.exports）引用对象的属性。

CommonJS规范仅允许使用exports变量来公开公共API，因此命名导出模式是唯一真正与CommonJS规范兼容的模式。module.exports是Node，js提供的一个扩展，用以支持广泛的模块定义模式。

```javascript
//logger.js
exports.info = (msg) => {
    console.log(`info ${msg}`);
}

//main.js
const logger = require('./logger.js');
logger.info('hello');
```



## 导出函数

将整个module.exports重新分配给一个函数。主要优点是只暴露一个单一的功能，给模块提供了一个明确的入口点。也很好的遵循了*小接触面(small surface area)*原则。同时以主要导出的函数作为其他API的命名空间，我们可以扩展次要或更高级的模块.

```javascript
//logger.js
module.exports = (msg) => {
    console.log(`info: ${msg}`)
}

module.export.error = (msg) => {
    console.log(`error: ${msg}`)
}
```

```javascript
//main.js
const logger = require('./logger.js');
logger('hello');
logger.error('error');
```

单一责任原则（Single Responsibility Principle ,SRP）：

> 每个模块应该对单个功能负责，该责任应完全由模块封装。



## 导出构造函数

```javascript
//logger.js
function Logger(name){
    this.name = name;
}
Logger.prototype.info = (msg) => {
    console.log(`info: ${msg}`);
}
Logger.prototype.error = (msg) => {
    console.log(`error: ${msg}`);
}
module.exports = Logger;

/*------------------- es6 --------------------*/
class Logger {
    constructor(name){
        this.name = name;
    }
    info(msg){
        console.log(`info: ${msg}`);
    }
    error(msg){
        console.log(`error: ${msg}`);
    }
}
module.exports = Logger;
```

以下改进可以防护不使用new指令调用，甚至可以模块作为工厂来使用。

```javascript
function Logger(name){
    if(!(this instanceof Logger)){
        return new Logger(name);
    }
    this.name = name;
}
```

```javascript
//main.js
const Logger = require('./logger');
const dbLogger = Logger('DB');
```

在ES6里有一个语法糖new.target，这是在所有函数里提供的meta属性，如果使用new关键字调用函数，则new.target的值为true。上面的代码可以改写为：

```javascript
function Logger(name){
    if(!new.target){
        return new Logger(name);
    }
    this.name = name;
}
```



## 导出实例

```javascript
function Fun(name){
    this.name = name
}
Fun.prototype.greet = function() {
    console.log('hello');
}

module.exports = new Fun('joe')
```

这种导出模式类似于创建*单例(singleton)*，然而因为在整个应用程序内该模块可能多次被安装所以并不能保证在应用程序内的唯一性。



## 修改其他模块或全局作用域

猴子补丁*（monkey patching）*

> 一个能修改其他模块或全局作用域对象的模块。通常是指在运行时修改现有对象以更改或扩展其行为或应用临时修复的做法。

一个模块可以没有任何导出，其仍然可以修改全局作用域及其中的任何对象，包括缓存中的其他模块。这通常是不安全和不好的。

```javascript
//fun.js
function Fun(name){
    this.name = name
}
Fun.prototype.greet = function() {
    console.log('hello');
}

module.exports = new Fun('joe')

//patch.js
require('./m').eat = function(){
    console.log('eated');
}
```

```javascript
//main.js
f = require('./fun.js');
f.eat(); 	//undefined
require('patch.js');
f.eat();	//eated
```

