---
title: Javascript的call和apply方法
date: 2018/11/23 22:22:22
tags:
  - javascript
categories: javascript
---

每个**函数**都具有两个非继承而来的方法，call和apply。

call和apply的作用一样，在特定的作用域调用函数，改变函数**内部**this指向，扩充函数依赖的作用域。

this一般指向调用某个方法的**对象**，使用call和apply方法就可以改变this的指向。
<!-- more -->
```javascript
redLight = {
  color: 'red'
};
blueLight = {
  color: 'blue'
};
color = 'white';
function Light() {
  console.log(`Lighting --- use color ${this.color}`)
};
Light();				//Lighting --- use color white
Light.call(redLight);	//Lighting --- use color red
Light.apply(blueLight);	//Lighting --- use color blue
```



## 不同点 : apply和call接受参数的方法不同

**apply接受的是数组参数，call接受的是连续参数**

**apply **

接收两个参数，一个是函数运行的作用域（this），另一个是参数数组。
语法：apply([thisObj [,argArray] ])

说明：如果argArray不是一个有效数组或不是arguments对象，那么将导致一个 
TypeError，如果没有提供argArray和thisObj任何一个参数，那么Global对象将用作thisObj。

**call**

与apply相似，第一个参数是函数运行的作用域，但另一个参数数组需要列举。
语法：call([thisObject[,arg1 [,arg2 [,...,argn]]]])

```javascript
color = 'white';
function Light(mode, time) {
  console.log(`Lighting --- use color ${this.color} in mode ${mode} at ${time}`)
};

Light();				
//Lighting --- use color white in mode undefined at undefined
Light.call(redLight, 'low', 'daylight');
//Lighting --- use color red in mode low at daylight
Light.apply(blueLight, ['high', 'night']);
//Lighting --- use color blue in mode high at night
```

```javascript
Math.max(1,2,3,4,5);				//5
Math.max.apply(Math, [1,2,3,4,5]);	//5
Math.max(...[1,2,3,4,5]);			//5 (es6)
```

