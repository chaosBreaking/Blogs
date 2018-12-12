---
title: Note — Node.js设计模式(三)
date: 2018/11/20 22:22:22
tags:
  - Node.js
categories: Node.js
---

# 工厂模式

工厂模式允许我们将对象的创建从实现中分离出来。从本质上来说，工厂方法包装了一个新实例的创建，这给了我们很大的灵活性。

<!-- more -->

```javascript
function createImage(imageName){
  if(imageName.match(/\.jpeg$/)){
    return new jpegImage(imageName)
  }
  else if(imageName.match(/\.gif$/)){
    return new gifImage(imageName)
  }
  else if(imageName.match(/\.png$/)){
    return new pngImage(imageName)
  }
  else {
    throw new Exception('Unsupported format')
  }
}
```

如上面的代码所示，createImage就是一个"图片工厂"，负责由给入的参数来生成具体的图像实例，如果我们要支持新的图片格式，只需要修改该工厂函数即可完成扩展。

**工厂模式允许我们不暴露创建对象的构造函数，避免其被继承和修改，符合了小暴露原则。

## 一种封装的机制

得益于闭包，工厂方法也可以用来进行功能的封装。

> **封装（encapsulation）**是指通过阻止外部代码直接操作对象来控制访问内部细节的技术。只能通过公共的接口与对象进行交互，将外部代码和对象内部运行的细节隔离开来。这种做法也被称作信息隐藏。与继承、多态和抽象一样，封装也是面向对象设计的基本准则。

Javascript没有访问级别的修饰符（比如无法申明一个私有变量），所以实现封装的唯一方式是函数作用域和闭包。使用工厂方法来实现私有变量是非常直接的，如下所示：

```javascript
function createPerson(name){
    const privateProperties = {};
    const person = {
        setName: name => {
            if(!name) throw new Error('A person must have a name');
            privateProperties.name = name;
        },
        getName: () => {
            return privateProperties.name
        }
    };
    return person;
}
```

以上代码利用闭包创建了两个对象，一个是返回的person对象，另一个是包含不能被直接访问的私有属性的privateProperties，内部的属性只能通过提供的setName和getName方法来修改和访问。

## 可组合的工厂函数

可组合的工厂函数是一类特殊的工厂函数，他们可以被组合到一起来构建新的增强的工厂函数。当我们想要创建从多个源继承一些行为和属性的对象，却不想构建复杂的类结构时就可以使用组合工厂函数。

可以用一个简单的例子来阐释组合工厂函数，需要引用stampit模块。该模块本质上允许我们通过一个方便的接口来定义工厂函数，使生成的对象拥有一系列特定的属性和方法。

```javascript
const stampit = require('stampit');
const character = stampit({
  props: {
    name: 'Anonymous',
    lifePoint: 100,
    x: 0,
    y: 0
  }
})
const mover = stampit({
  methods: {
    move(x, y) {
      this.x += x;
      this.y += y;
      console.log(`${this.name} moved to (${this.x},${this.y})`);
    }
  }
})
const shooter = stampit({
  props: {
    bullets: 6
  },
  methods: {
    shot() {
      if(this.bullets <= 0) {
        console.log('run out of bullets');
        return 0;
      }
      this.bullets -= 1;
      console.log(`${this.name} shooted`);
    }
  }
})
```

以上代码利用stampit创建了三种基本类型分别是人物，可行走的人物和可射击的人物。接下来我们创建一个组合类，其实例拥有人物属性，可以行走和射击。

```javascript
let Player = stampit.compose(character, mover, shooter);
let player1 = Player()
player1.name = 'kimmy';
player1.move(1,1);	//kimmy moved to (1,1)
player1.shot();		//kimmy shooted
```

通过stampit的compose方法，我们可以将基本类型进行组合，从而创建出具有三者属性和方法的新类型Player。