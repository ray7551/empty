---
published: true
layout: default
title: 异步任务的简单控制：串联
keywords: async, javascript
tags: javascript
---

　　上篇说到异步控制的一个更复杂的情况：有多个不同的异步任务 t1、t2、t3，在不引入 Promise 机制的情况下，如何串联执行。对于那些不想引入 Promise 的小类库来说，这个问题有重要意义。  
　　引入 Promise 会带来 Promise 的复杂性，对于没有完全理解它的人而言，很容易错误地使用它。在我翻译的[这篇文章](https://github.com/mattdesl/promise-cookbook/blob/master/README.zh.md#小模块中的-promise)中，还提到了一些其它的不使用 Promise 的原因。  
　　解决这个问题，有三个思路：

1. 使用类似 async 库的做法，使用原始的回调式写法，控制异步任务流程；
2. 基本原理和第一个思路一样基于回调，区别在于用构造函数封装变量；
3. 基于 ES6 Generator 实现。

　　本文将从最简单原始的实现开始，依次实现三种做法。  
　　为方便讨论，我们先定义三个异步任务 t1、t2、t3，将会分别从 GitHub API 取得不同的 response status code。

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

<a class="jsbin-embed" href="http://jsbin.com/kaquxe/6/embed?js,console">在线预览</a>

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

　　这涉及到如何在程序执行过程中用合适的方式保存状态。在全局域定义一个用来存储状态的变量，然后在函数中改变它，这是比较 ugly 的做法。有多种方法可以解决。

## 纯闭包版本

　　第一种方法，可以参考 async 库的做法，需要灵活使用闭包的能力：

```javascript
var series = function(tasks, endCallback) {
  var getKey = _nextKey(Object.keys(tasks));
  var key = getKey.next();
  var results = {};
  executor(tasks, key, function(currentKey, data) {
    results[currentKey] = data;
    console.log('results now: ', results);
  }, function() {
    endCallback(results); // 把结果集 results 传递给最顶层的 endCallback
  });

};

var executor = function(tasks, key, loopCallback, endCallback) {
  tasks[key.current](function(data) {
    console.log(key.current, ' :', data);
    // 每个异步任务完成后，调用 loopCallback，将结果写入结果集 results
    loopCallback(key.current, data);
    if (key.next === undefined) {
      // 所有异步任务完成后，调用最顶层的 endCallback
      endCallback();
      return;
    }
    executor(tasks, key.next(), loopCallback, endCallback);
  });
};

var _nextKey = function(keys, current) {
  current = current===undefined ? -1 : current;
  var next;
  if (keys[current+1]) {
    next = function() {
      return _nextKey(keys, current+1);
    };
  }
  return {
    next: next,
    current: keys[current]
  };
};

series({
  t1: t1,
  t2: t2,
  t3: t3
},
function(results) {
  console.log('End: results=',results);
});
```

<a class="jsbin-embed" href="http://jsbin.com/kaquxe/21/embed?js,console">在线预览</a>

　　事实上，要理清楚 async 库的 `async.series` 这部分代码，着实不易。上面的版本是去除错误处理，可变参数列表之后的修改版本，看起来还是很绕。简单分析一下上面的 `series` 函数，它做了这么几件事：

- 用 `_nextKey ` 方法取得 tasks 的 `key`，并且可以用 `key.next()` 链式地取得下一个key；
- 初始化结果集 `results`
- 异步任务执行交给 `executor` 方法：
    - 每个异步任务完成后，调用 loopCallback，将结果写入结果集 results
    - 所有异步任务完成后，把结果集 results 传递给最顶层的 endCallback

## 使用 `this` 的版本

　　第二种实现的代码比较清晰，而且比较容易理解。在 this 下定义一个属性：

```javascript
function series(callback, tasks) {
  var task = tasks.shift();
  var self = this;
  this.results = this.results || [];

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
  console.log('results:', results);
}, [t1, t2, t3]);
```

<a class="jsbin-embed" href="http://jsbin.com/kaquxe/18/embed?js,console">在线预览</a>

　　像这样，用构造函数做封装之后，就不会污染全局作用域了。

## ES6 Generator 版本
　　到此为止，上面所述各种方法，都是基于回调的：在执行器函数内，每个异步任务的回调中调用执行器本身去执行下一个任务，用回调 + 递归的方式完成循环。回调的方式虽然管用，但是基于回调的代码是层层嵌套的结构，让人难以理清楚它的执行流程，有没有可能让异步代码也像同步的代码一样清晰可读呢？  
　　借助 ES6 的 Generator，就可以实现：

```javascript
function ajax(url, callback) {
  fetch(url).then(function(response) {
    callback(response.status);
  });
}
var t1 = function() {
  ajax('https://api.github.com', function(response) {
    console.log(200);
    series.next(200);
  });
};
// t2, t3...

var seriesGenerator = function *(tasks) {
  var results = {};
  for (var key in tasks) {
    if(tasks.hasOwnProperty(key)) {
      console.log('start executing: ' + key);
      results[key] = yield tasks[key]();
    }
  }
  console.log('Results: ', results);

};

var series = seriesGenerator({
  t1: t1,
  t2: t2,
  t3: t3
});
series.next();
```
<!-- <a class="jsbin-embed" href="http://jsbin.com/kaquxe/28/embed?js,console">在线预览</a> -->

以下是控制台输出：

```
start executing: t1
200
start executing: t2
401
start executing: t3
404
Results:  Object {t1: 200, t2: 401, t3: 404}
```

　　`seriesGenerator` 内的代码就是一个简单的 for-in 循环，在每次循环中发起 ajax 请求，用 `yield` 中断迭代器 `series` 执行，在 ajax 请求完成之后执行 `series.next(response.status)`，来恢复迭代器运行，同时将 `response.status` 返回给 `results[key]`。

## 处理数据依赖
　　借助 Generator，可以方便地处理任务间的数据依赖。比如，我们要从 GitHub API 列表中找到搜索版本库的 API，然后搜索“async”，取结果中第一个版本库的 owner 的姓名，我们可以这样安排三个任务：

```javascript
var t1 = () => {
  ajax('https://api.github.com', function(response) {
    series.next(response.repository_search_url);
  });
};
var t2 = (repoSearchUrl, search) => {
  repoSearchUrl = repoSearchUrl.slice(0, repoSearchUrl.indexOf('?') + 3);
  search = search || 'async';
  search_url = repoSearchUrl + search;
  ajax(search_url, function(response) {
    series.next(response.items[0].owner.url);
  });
};
var t3 = (ownerUrl) => {
  ajax(ownerUrl, function(response) {
    series.next(response.name);
  });
};
```

　　然后稍微修改一下 `seriesGenerator`：

```javascript
var seriesGenerator = function *(tasks) {
  var results = {};
  var prevKey;
  for (var key in tasks) {
    if(tasks.hasOwnProperty(key)) {
      console.log('start executing: ' + key);
      results[key] = yield tasks[key](results[prevKey]);
      prevKey = key;
    }
  }
  console.log('Results: ', results);

};
```
<!-- <a class="jsbin-embed" href="http://jsbin.com/kaquxe/30/embed?js,console">在线预览</a> -->


<!-- 根据数据依赖确定执行顺序 -->
<!-- 错误处理 -->
<!-- 支持可变参数列表 -->


## 结语
　　以上的几种实现，都没有引入 Promise。对于那些不想引入 Promise 的小类库来说，async 库是一个不错的选择。具体到串联执行异步任务这个问题，除了 async 库使用的纯回调式实现之外，还可以使用更清晰 `this` 和构造函数的版本。最后，借助 Generator，我们成功地消除了大部分的回调，异步的代码变得和同步代码一样清晰易读。  
　　很多人认为最终的解决方案，必须是 [ES7 的 Async Functions](http://tc39.github.io/ecmascript-asyncawait/)。在现阶段，基于 Generator 的方案已经可以满足基本要求了。

<script src="http://static.jsbin.com/js/embed.min.js?3.34.2"></script>