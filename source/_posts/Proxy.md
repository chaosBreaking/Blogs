---
title: Note — Node.js设计模式(五)
date: 2018/11/24 22:22:22
tags:
  - Node.js
  - Proxy
categories: Node.js
---

# 代理模式

代理（proxy）是一个用来**控制**对另一个对象（**本体**）访问的**对象**。它实现了与本体对象相同的接口，我们可以对两个对象进行随意的替换使用，也可以把这种模式称为替代模式。代理对象可以拦截所有或者部分 本来要对本体对象 执行的操作，补充或增强它们的行为。

> 我们讨论的代理模式不是类之间的代理，代理模式指的是**封装本体对象的真实接口**，从而保持其内部状态。

<!-- more -->

代理模式在某些场景下十分有用，比如

*	**数据验证：**代理对象在将输入传递给本体对象之前先进行校验。
*	**安全性：** 代理对象会校验客户端是否被授权操作本体对象。
*	**缓存：** 代理对象内部维护一个缓存系统，当需要访问的数据不在缓存时才将操作传递到本体对象。
*	**延迟初始化：** 如果对本体对象的创建是非常耗费时间和空间的，代理对象可以延迟其创建时机。
*	**日志：**代理对象拦截调用的方法和参数并将他们记录下来。
*	**远程对象代理：** 代理对象可以为远程对象提供本地代表，就像调用本地对象。



## 实现代理模式的方法

当我们要代理一个对象的时候，可以选择拦截其所有方法或者部分方法，将其余的方法直接委托给本体。

### 1.对象组合

组合是指一个对象为了扩展其自身功能或者使用其他对象的功能，将另一个对象合并进来。对于代理模式来说，我们创建一个拥有和本体对象相同接口的新对象，并且以实例变量或闭包变量的形式引用存放在代理内部的本体对象。可以在客户端初始化时注入本体对象或者由代理对象来创建。

```javascript
function createProxy(subject) {
  //获取本体对象的原型
  const proto = Object.getPrototypeOf(subject);
  //代理对象
  function Proxy(subject) {
    this.subject = subject;
  }
  Proxy.prototype = Object.create(proto);
  //代理方法
  Proxy.prototype.hello = function() {
    return this.subject.hello() + ' World!';
  };
  //委托给本体对象的方法
  Proxy.prototype.goodbye = function() {
    return this.subject.goodbye.apply(this.subject, arguments)
  }

  return new Proxy(subject)
}
module.exports = createProxy
```

上例拦截了本体对象的hello方法，将goodbye方法委托给本体对象。

上例为了维护原型链正确，使用了伪继承的方法，如果不需要就可以使用更直接的方法，如下

```javascript
function createProxy2(subject) {
  return {
    hello: () => {
      return subject.hello() + 'World';
    },
    goodbye: () => {
      return subject.goodbye.apply(subject, arguments)
    }
  }
}
```

### 2.对象增强（Object augmentation)

对象增强（或者叫monkey patching)通过替换本体对象方法的方式来实现代理。

```javascript
function createProxy3(subject) {
  const orihello = subject.hello;
  subject.hello = () => {
    return orihello.call(this) + 'World'
  }
}
```



对比以上两种方法，对象组合是创建代理最安全的方法，保证了本体对象无法被外部访问，本体对象的原始行为不会被改变。但是如果只想代理某一个方法，需要将剩余所有的方法委托给本体对象。

对象增强修改了本体对象，但是并没有委托相关的各种不便。在不太关注是否修改本体对象的情况下，对象增强是最实用的方法。



## 生态系统中的代理模式——函数钩子与面向行为编程（AOP）

代理模式多种多样，在Node.js及其生态系统中非常常见。我们可以找到一些帮助简化创建代理对象的库。其中大部分都使用了对象增强的方式。在社区中，这一模式也被称为**函数钩子（function hooking）**,或者有时也被叫做**面向行为编程（AOP）**，实际上这也是代理模式的一个常用领域。在AOP模式中，这些库允许开发者为某一或某一系列方法设置pre或者post钩子，在指定方法运行前或者运行后可以执行自定义的代码。

有些时候代理也被称为**中间件**，因为该模式在中间件模式中很常见，它允许我们对一个函数的输入和输出进行预处理或者后处理，代理模式还允许使用类似中间件管道的方式来为同一个方法注册多个钩子。

## ES2015中的Proxy对象

ES2015规范引入了一个全局对象Proxy，Node.js从V6版本开始支持它。Proxy提供的接口包括一个构造函数，接受**target**和**handler**作为参数。

```javascript
const proxy = new Proxy(target, handler);
```

target表示需要被代理的对象（本体对象），handler是用来**定义代理行为**的特殊**对象**。

handler对象包含了一系列预先定义好名称的可选方法，例如apply,get,set和has。这些方法被称作**捕获方法（trap methods）**，当代理对象实例在执行某些操作时就会自动被调用。

```javascript
const brands = {
  car: 'volkswagon',
  drink: 'coca cola',
}
const uppercaseBrands = new Proxy(brands, {
  get: (target, property) => target[property].toUpperCase()
});
console.log(uppercaseBrands.car, uppercaseBrands.drink);	//VOLKSWAGON COCA COLA
```



如上例所示，使用Proxy提供的方法拦截了所有目标对象，也就是对brands对象属性的访问，并将其属性值转换为大写字符串。

这种方法可以拦截和自定义很多对对象的操作，对一些场景如 *元编程*（meta-programming）、*操作符重载（operator-overloading）*和*对象虚拟化（object virtualization）*提供了很好的支持。

```javascript
const evenNumbers = new Proxy([], {
  get: (target, index) => index * 2,
  has: (target, number) => number%2 === 0 
});
console.log(evenNumbers[0],evenNumbers[1],evenNumbers[2])	//0 2 4
console.log(1 in evenNumbers, 2 in evenNumbers)				//false true
```

上例创建了一个包含偶数的数组，利用的就是Proxy的特性。我们用Proxy代理了一个空数组，在handler中定义了**get**和**has**两个**捕获方法**。

* `get`方法拦截了对数组元素的访问，并根据指定的索引返回特定的偶数。

* `has`方法拦截了对in操作符的使用，并检查给定的数是否为偶数。

Proxy API还提供了很多其他的捕获函数，例如set、delete和construct。