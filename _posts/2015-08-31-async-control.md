---
published: true
layout: default
title: 异步任务的简单控制
keywords: async, javascript
tags: javascript
---

　　上篇说到异步控制的一个更复杂的情况：有多个不同的异步任务 t1、t2、t3，在不引入 Promise 机制的情况下，如何串联或并联执行。
　　为方便讨论，我们先定义三个异步任务 t1、t2、t3，将会分别从 Github API 取得不同的 response status code。

```javascript
function ajax(url, callback) {
  fetch(url).then(function(response) {
    callback(response.status);
    return response.blob();
  });
}
var t1 = function(callback) {
  ajax('http://api.github.com', function(response) {
    console.log(response);
    callback(response);
  });
};
var t2 = function(callback) {
  ajax('http://api.github.com/user', function(response) {
    console.log(response);
    callback(response);
  });
};
var t3 = function(callback) {
  ajax('http://api.github.com/xxx', function(response) {
    console.log(response);
    callback(response);
  });
};
```

## 串行初级版本
　　比较简单的版本如下：

```javascript
function series() {
  var tasks = Array.prototype.slice.call(arguments, series.length);
  var task = tasks.shift();
  task(function(response) {
    if (tasks.length > 0) // 继续递归还是返回的分界线
      series.apply(this, tasks); // 用 apply 代替 series(tasks[0], tasks[1], ..) 的形式
  });
}
series(t1, t2, t3);
```

<a class="jsbin-embed" href="http://jsbin.com/kaquxe/6/embed?js,console">JS Bin on jsbin.com</a>

　　对可变参数列表（rest parameters）的处理，用到了 `Function.length` ，这个属性指的是函数声明中除 rest parameters 之外的参数个数。这个版本的缺点是不能收集每个任务的返回值。

## 保存异步任务的返回值
　　为保存每个 task 的返回值，可以在 series 函数之外定义一个数组 results：

```javascript
var results = [];
// 因为 task 的回调函数内方便再继续用 apply，为简化处理，tasks 改为数组形式
function series(callback, tasks) {
  var task = tasks.shift();
  console.log('task', task);
  task(function(response) {
    results.push(response);
    if (tasks.length > 0) {
      series.call(this, callback, tasks);
    } else {
      callback(results);
    }
  });
}
series(function(results) {
  console.log(results);
}, [t1, t2, t3]);
```

　　这涉及到如何在程序执行过程中用合适的方式保存状态。在全局域定义一个用来存储状态的变量，然后在函数中改变它，这是比较 ugly 的做法。有两种方法可以解决，第一种是在 this 下定义一个属性：

```javascript
function series(callback, tasks) {
  var task = tasks.shift();
  var self = this;
  this.results = this.results || [];
  console.log('results:', this.results);
  task(function(response) {
    self.results.push(response);
    if (tasks.length > 0){
      series.call(self, callback, tasks);
    } else {
      callback(self.results);
    }
  });
}
series(function(results) {
  console.log(results);
}, [t1, t2, t3]);
```

　　与传统面向对象语言不同，JavaScript 中，一个函数中的 `this` 所指的并不是某个类的实例，而是指向调用函数的对象。正因为 `this` 是代表调用函数的对象，它所指的对象是随函数使用场合而变化的。可以注意到，在 series 内将 `this` 赋给 `self`，就是因为，在内层的匿名函数中， `this` 已经变了。在 ECMAScript 5 环境下，这种给 `this` 另取一个名字的方式可以用 [Function.prototype.bind](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) 方法代替。bind 方法在原函数的基础上创建了一个新的函数，并可以指定新的函数中 this 所指的对象。
　　仔细分析一下这个方法，其实在全局域调用 series 函数的时候， series 函数里的 this 就是全局对象，执行它的时候会污染全局作用域。所以这个方法还有改进的空间：

```javascript
function Series(callback, tasks) {
  this.results = this.results || [];
  console.log('results:', this.results);
}
Series.prototype.run = function(callback, tasks) {
  var task = tasks.shift();
  var self = this;
  task(function(response) {
    self.results.push(response);
    if (tasks.length > 0){
      self.run(callback, tasks);
    } else {
      callback(self.results);
    }
  });
};
var taskSeries = new Series();
taskSeries.run(function(results) {
  console.log(results);
}, [t1, t2, t3]);
```

<a class="jsbin-embed" href="http://jsbin.com/kaquxe/16/embed?js,console">JS Bin on jsbin.com</a>

　　像这样，用构造函数做封装之后，就不会污染全局作用域了。


<script src="http://static.jsbin.com/js/embed.min.js?3.34.2"></script>