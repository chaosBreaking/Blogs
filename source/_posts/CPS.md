---
title: Note — Node.js设计模式(二)
date: 2018/10/3 23:33:33
tags:
  - Node.js
  - CPS
categories: Node.js
---

# CPS(Continuation Passing Style)

在JS中，回调是一个作为参数传递到一个函数的函数，当操作完成时将调用该结果。在函数式编程中，这种传播结果的方式称为**CPS**。这是个通用的概念，并不是总与异步操作相关联。其表示的是通过将结果传递给另一个函数(回调)而使结果传播，而不是直接返回给调用者。

<!-- more -->

##同步CPS

一个简单的同步函数

```javascript
function add(a, b){
    return a + b;
}
```

可见该函数使用return将结果返回给调用者，我们称此为**Direct Style(直接风格)**

该函数等效的CPS写法是

```javascript
function add(a, b, callback){
    callback(a,b);
}
```

对其进行调用

```javascript
console.log('before');
add(1, 2, res => console.log(`Result: ${res}`));
console.log('after');
```

因为add函数是同步的，所以结果是

```
before
Result: 3
after
```

这个add函数是一个同步CPS函数，意味着只有回调执行完成时它才返回值。

##异步CPS

改写上面的add函数使其成为异步的

```javascript
function asyncAdd(a, b, callback){
    return setTimeout(()=> callback(a + b), 100);
}
```

以上代码通过调用setTimeout来模拟异步调用，接下来调用这个异步函数

```javascript
console.log('before');
asyncAdd(1, 2, res => console.log(`Result: ${res}`));
console.log('after');
```

通过运行之后发现结果是这样的

```javascript
before
after
Result: 3
```

可见在setTimeout函数触发异步操作后，asyncAdd函数立即返回，将控制权交回给事件循环以处理新事务。当异步操作完成时，从提供给异步函数的回调开始将重新启动该过程，从而引发退绕。执行将从事件循环开始，因此将有一个新的堆栈。由于是**闭包**，即使在不同时间的环境下调用回调函数也不用维护异步调用的上下文。

## 同步或异步

某些函数编写过程中可能会同时牵涉到异步和同步，就以书中例子来看

```javascript
const fs = require('fs');
const cache = {};
function readFile(filename, callback){
    if(cache[filename])
        return callback(cache[filename]);
    else{
        fs.readFile(filename, 'utf8', (err, data) => {
            cache[filename] = data;
            callback(cache[filename]);
        });
    }
}
```

分析以上代码可知，如果文件是缓存过的，那么这是一个同步调用，会立即执行callback。但如果文件没有缓存过，那么会调用异步操作来读取文件，直到readFile返回结果。

之后按照书中的例子，我们调用这个函数来创建一个文件读取监听器，当文件读取完毕时将调用所有的监听器。

```javascript
function createFileReader(filename){
    const listeners = [];
    readFile(filename, value => {
        //调用readFile并且给一个回调函数，使得读取文件后执行listeners里的所有回调函数
        listeners.forEach(listener => listener(value));
    });
    return {
       	//返回一个对象用来给外部注册监听器
        onDataReady: listener => listeners.push(listener)
    };
}
const reader1 = createFileReader('data.txt');
reader1.onDataReady(data => {
   console.log(`first call data : ${data}`);
   const reader2 = createFileReader('data.txt');
    reader2.onDataReady(data => {
        console.log(`second call data : ${data}`);
    })
});
```

调用的结果是

```
first call data : somedata......
```

分析第二个操作不会调用的原因：

1.在reader1的创建过程中，readFile函数以异步的方式运行，因为没有缓存的内容。因此有足够的时间来注册监听器，这些监听器在文件读取操作完成时在之后的另一个事件循环中将被调用。

2.在reader2的创建过程中，readFile函数是同步运行的，因为已经有了缓存。因此它会立即执行回调函数也就是createFileReader，那么所有的监听器也会被立即调用。而此时监听器并没有被注册(下一步的时候才注册reader.onDataReady...)，因此我们给reader2注册的监听器永远不会被调用。

因此归根结底，是因为readFile函数在不同情况下表现出不一致的行为，有时异步有时同步。这种令人难以捉摸的函数被类比为释放的Zalgo

## So?

所以吸取教训，API必须保证其特性明确，要么同步要么异步。

那么有两种改写的思路：全同步或全异步

全同步的写法是

```javascript
function readFileSync(filename, callback){
    if(cache[filename]){
        callback(cache[filename]);
    }
    else{
        cache[filename] = fs.readFileSync(filename, 'utf8');	//同步读取，没有回调函数
        callback(cache[filename]);
    }
}
```

当然，Node.js里同步无异于开历史倒车，同步的缺点太多所以我们改成完全异步。在Node.js里可以使用一个方法来延迟执行操作：process.nextTick()。它的作用是**延迟一个函数的执行直到下一个事件循环的到来**。它的功能就是将回调作为参数，将其推到事件队列的顶部，在任何待处理的IO事件之前返回。一旦事件循环再次运行，该回调回被执行。

```javascript
function readFileAsync(filename, callback){
    if(cache[filename]){
        process.nextTick(() => callback(cache[filename]));
    }
    else{
        fs.readFile(filename, 'utf8', (error, data) => {
            cache[filename] = data;
            callback(data);
        });
    }
}
```

现在这个函数将会始终表现为异步执行。

另一个用于延迟执行代码的API是setImmediate()。它与process.nextTick()作用相似，但语义是完全不同的。

process.nextTick()延迟的回调在任何已经调度的IO之前运行，在某些情况下可能会导致IO饥饿（回调插队导致IO任务没有被安排，使得IO空闲），例如递归调用。

setImmediate()延迟的回调将在队列中已有的任何IO事件后排队。



## Node.js回调约定

* 回调函数置尾

  ```javascript
  fs.readFile(filename, 'utf8', callback)
  ```

* 暴露错误优先 

  ```javascript
  fs.readFile(filename, 'utf8', (err, data) => {
      if(err)
          errorHandler(err);
      else
          processData(data);
  });
  ```


* 传播错误

  在异步CPS风格中，可以将错误简单的传播到链中的下一个回调来进行处理，但是要用return。类似于

  ```javascript
  ......
  return callback(err)
  ......
  ```

* 未捕获的异常
