---
published: true
layout: default
title: JavaScript原型链继承
keywords: JavaScript, 原型, 继承
tags: JavaScript, 继承, 面向对象
---


　　有类继承语言（比如Java，  C++， PHP） 学习经验的开发者在学习Javascript的时候会像我一样执着于这样几个个问题：构造函数的prototype属性到底是用来干什么的？里面有什么东西？原型链是什么样的？如何用js实现继承？本文将围绕这些问题进行讨论。  


###prototype属性的来由
　　在围绕一个问题进行学习的时候，往往因为涉及的东西太多而无法入手，比如我们现在讨论的js原型链，相信很多人会和我一样看了一大堆资料却仍感觉似懂非懂，过不了多久又忘得差不多了。这时候不妨从最根本的问题问起：为什么要有`prototype`属性？  
　　我们在chrome的控制台输入以下语句：  

```javascript
function Dog(name) {
  this.name = name;
  this.species = "animal";
  //构造函数中的this关键字，它就代表了新创建的实例对象。
  this.Introduce = function () {
    alert("My name is " + this.name);
  };
};
var alice = new Dog("alice");
var bob = new Dog("bob");
```
　　C++和Java使用new命令时，都会调用"类"的构造函数（constructor）。所以JavaScript的创造者就做了一个简化的设计，在Javascript中，"new" 后面跟的不是类，而是构造函数，或者称为构造器。  
　　这样用构造函数生成实例对象，好像已经实现了继承用构造函数生成实例对象，但我们运行一下下面的代码：  

```javascript
console.log(alice.name == bob.name);            //false
console.log(alice.species == bob.species);      //true
console.log(alice.Introduce == bob.Introduce);  //false
```
　　分别得到了 false，true，false。按照预想，Introduce属性应该和species属性一样，是共享的，但是在这里，它们是不同的，这会造成空间浪费。这显然不是我们想要的。  
　　在JavaScript中，每一个构造函数都有一个prototype属性，指向一个对象，这个对象的所有属性和方法，都会被构造函数的实例继承。可以说，构造函数的prototype属性就是为了 **被继承** 而设计的。  
　　构造函数有了prototype属性，我们就可以这样做：

```javascript
function Dog(name) {
  this.name = name;
};
Dog.prototype.species = "animal";
Dog.prototype.Introduce = function () {
    alert("My name is " + this.name);
  };
var alice = new Dog("alice");
var bob = new Dog("bob");

console.log(alice.name == bob.name); //false
console.log(alice.species == bob.species); //true
console.log(alice.Introduce == bob.Introduce);   //true
```
　　我们把一些不变的属性放到prototype里了。在一些js类库中，常常会见到用这个方法来为js内置类型的原型对象添加一些非标准的函数。  
　　为保持可读性，常常会写成这样的形式：

```javascript
function Dog(name) {
  this.name = name;
};
Dog.prototype = {
    species : "animal",
    Introduce : function () {
        alert("My name is " + this.name);
    }
};
```

　　需要注意的是，`prototype`属性里的东西，也 **仅仅用于被继承** ，也就是说，构造函数的prototype里定义的属性，并不是构造函数本身拥有的：  

```javascript
Dog.species;   //undefined
Dog.Introduce;  //undefined
```
　　
###原型链
　　对应于构造函数的`prototype`属性，派生的对象有一个 `__proto__` 属性，用于指向它所继承的那个prototype对象（ `Dog.prototype === alice.__proto__` ）。
　　如果你在控制台输入 `console.dir(Dog.prototype)` ，就会清楚地看到构造函数的`prototype`属性所指的那个对象到底有些什么东西。你会发现，除了我们在代码里面定义的`species`和`Introduce`之外，还有一个`__proto__`和一个`constructor`。其中`constructor`会被继承，因为`alice.constructor === Dog.prototype.constructor`,而`__proto__`则始终指向这个对象的构造器的prototype：

```javascript
Dog.__proto__ === Dog.constructor.prototype;        //true
alice.__proto__ === alice.constructor.prototype;    //true
```

####new 运算符是如何工作的
　　介绍了`__proto__`，就容易理解使用new+构造函数生成对象的过程了。这个过程分为三步：  
1. 创建一个对象实例。这个对象的 __proto__ 属性设置为构造器的prototype 。
2. 初始化实例。构造函数被传入参数并调用，关键字 this 被设定为该实例。
3. 返回实例。  

　　可以用JS代码来解释：

```javascript
//http://blog.vjeux.com/2011/javascript/how-prototypal-inheritance-really-works.html
function New (f) {
    var n = { '__proto__': f.prototype }; /*step 1*/
    return function () {
        f.apply(n, arguments);            /*step 2*/
        return n;                         /*step 3*/
    };
}
```
　　我们可以用下面的代码来测试一下：

```javascript
carol = (New(Dog))('carol');
carol.__proto__ === alice.__proto__;  //true
```

####原型链
　　有了`__proto__`这个属性，我们可以从`alice`向上找到等同于`Dog.prototype`的`alice.__proto__`，并找到我们在Dog的原型里定义的属性。事实上Dog的原型也有自己的原型，可以通过`alice.__proto__.__porto__`访问到。  
　　《JavaScript秘密花园》里，关于查找对象的属性有这样一句话：
> 当查找一个对象的属性时，JavaScript 会向上遍历原型链，直到找到给定名称的属性为止。  

　　用js代码来解释就是这样：  

```javascript
function getProperty(obj, prop) {
    if (obj.hasOwnProperty(prop)){
        return obj[prop];
    }else if(obj.__proto__ !== null){
        return getProperty(obj.__proto__, prop);
    }else{
        return undefined;
    }
};
```

　　需要注意的是，`__proto__` 是一个不应在你代码中出现的非正规的用法，这里仅仅用它来解释JavaScript原型继承的工作原理。

###JavaScript原型继承的一种实现
　　在实际应用中，我们常常需要继承某个对象，而对象是不能像构造函数那样调用的，如何实现对对象的继承呢？  
　　Douglas Crockford 在他的[Prototypal Inheritance in JavaScript](http://javascript.crockford.com/prototypal.html)一文中，提出一个借助中间函数来实现原型继承的方式：

```javascript
function object(o) {
    function F() {}
    F.prototype = o;
    return new F();
}
```
　　如果我们允许使用 __proto__ ，那我们也可以这样写：

```javascript
object = function (parent) {
    return { '__proto__': parent };
};
```
　　下面这段代码就实现了原型继承：

```javascript
var Dog = {
    name: '',
    species: 'animal',
    Introduce: function() {
        alert("My name is " + this.name);
    }
};

var dave = object(Dog);
dave.name = 'dave';
dave.Introduce();
```
　　值得注意的是，我们把原先的Dog构造函数改写成了对象，这实际上是对一个对象的继承。  
　　对于如何用js实现继承，或者说如何用js写出面向对象风格的代码这一问题，答案是有很多的，这只是其中一种。感兴趣的同学可以看看下面参考阅读里面Crockford的另一篇文章《JavaScript中的类继承》和阮一峰的面向对象编程系列博文。  


###参考阅读
1. 阮一峰的四篇博文：  
[Javascript 继承机制的设计思想](http://www.ruanyifeng.com/blog/2011/06/designing_ideas_of_inheritance_mechanism_in_javascript.html)  
[Javascript 面向对象编程（一）：封装](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html)  
[Javascript 面向对象编程（二）：构造函数的继承](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html)  
[Javascript 面向对象编程（三）：非构造函数的继承](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance_continued.html)  
2. JavaScript原型继承工作原理 [译文](http://www.ituring.com.cn/article/56184)  [原文]( http://blog.vjeux.com/2011/javascript/how-prototypal-inheritance-really-works.html)
3. Douglas Crockford [Prototypal Inheritance in JavaScript](http://javascript.crockford.com/prototypal.html)
4. Douglas Crockford [JavaScript中的类继承](http://javascript.crockford.com/zh/inheritance.html) 
5. [Standard ECMA-262](http://www.ecma-international.org/ecma-262/5.1/#sec-4.2.1)
6. [JavaScript对象模型-执行模型](http://www.cnblogs.com/RicCC/archive/2008/02/15/JavaScript-Object-Model-Execution-Model.html)


