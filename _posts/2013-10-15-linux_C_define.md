---
published: true
layout: default
title: linux C学习笔记（1）-宏定义中的do-while
keywords: 宏，C语言，do while
tags: C, do while
---
　　PHP源码中大量使用了宏操作，比如PHP5.3新增加的垃圾收集机制中的一段代码：

```cpp
#define ALLOC_ZVAL(z) \
    do { \
        (z) = (zval*)emalloc(sizeof(zval_gc_info)); \
        GC_ZVAL_INIT(z); 							\
    } while (0)
```
　　这段代码，在宏定义中使用了 `do{...}while(0)` 语句格式。如果我们搜索整个PHP的源码目录，会发现这样的语句还有很多。在其他使用 C/C++ 编写的程序中也会有很多这种编写宏的代码，多行宏的这种格式已经是一种公认的编写方式了。为什么在宏定义时需要使用`do-while`语句呢?
　　原因有二。
###1. 不会破坏使用环境的代码结构。
　　如下所示代码：

```cpp
#define TEST(a, b) a++;b++;
if (expr)
TEST(a, b);else
do_else();
```
　　代码进行预处理后，会变成：

```cpp
if (expr)
a++;b++;else
do_else();
```
　　这样`if-else`的结构就被破坏了if后面有两个语句，这样是无法编译通过的。
　　你也许会想，如果要不破坏使用环境的结构，为什么非要用`do-while`而不是简单的用`{}`括起来呢，这就要说到第二点。
###2. 保证在宏定义后加分号时能编译通过
　　例如上面的例子，出于习惯，我们在使用宏TEST的时候后面加了一个分号。 那如果是把宏里的代码用`{}`括起来，加上最后的那个分号。 还是不能通过编译。所以一般的多表达式宏定义中都采用`do-while(0)`的方式。
　　小结一下，这种定义方式 **适用于宏定义中存在多语句的情况。**
