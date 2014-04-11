---
published: false 
layout: default
title: linux C学习笔记（3）-C标准I/O库函数
keywords: IO函数，C语言
tags: C
---

###标准I/O库函数

####stdin/stdout/stderr
　　UNIX的传统是Everything is a file,键盘、显示器、串口、磁盘等设备在/dev 目录下都有一个特殊的设备文件与之对应,这些设备文件也可以像普通文件一样打开、读、写和关闭,使用的函数接口是相同的。从键盘输入和输出到屏幕都是对终端设备进行读和写的操作，为什么`printf`和`scanf` 不用打开就能对终端设备进行操作呢?
　　因为在程序启动时(在main 函数还没开始执行之前)会自动把终端设备打开三次,分别赋给三个`FILE *`指针`stdin`、`stdout`和`stderr`,这三个文件指针是`libc`中定义的全局变量,在`stdio.h`中声明,`printf`向`stdout`写,而`scanf` 从`stdin` 读,后面我们会看到,用户程序也可以直接使用这三个 文件指针。这三个文件指针的打开方式都是可读可写的,但通常`stdin` 只用于读操作,称为标准输入(Standard Input), `stdout`只用于写操作,称为标准输出(Standard Output), `stderr`也只用于写操作,称为标准错误输出(Standard Error),通常程序的运行结果打印到标准输出, 而错误提示(例如gcc报的警告和错误)打印到标准错误输出,所以`fopen`的错误处理写成这样更符合惯例:

```cpp
	if ( (fp = fopen("/tmp/file1", "r")) == NULL){ 
	    fputs("Error open file /tmp/file1\n", stderr);
	    exit(1);
	｝
```
　　不管是打印到标准输出还是打印到标准错误输出效果是一样的, 都是打印到终端设备(也就是屏幕)了,那为什么还要分成标准输出和标准错误输出呢?
　　有时候我们需要把正常的运行结果输出和错误提示分开，比如我们想要让运行结果输出到屏幕，而错误提示输出到某个文件，就可以用重定向操作，把标准错误输出重定向到这个文件，而标准输出仍然对于屏幕。这样就可以把正常的运行结果和错误提示分开,而不是混在一起打印到屏幕了。

####errno与perror函数
　　很多系统函数在错误返回时将错误原因记录在`libc` 定义的全局变量`errno` 中,每种错误原因对应一个错误码,`errno` 在头文件`errno.h`中声明,是一个整型变量。
　　如果在程序中打印错误信息时直接打印`errno` 变量,打印出来的只是一个整数值,仍然看不出是什么错误。比较好的办法是用`perror`或`strerror` 函数将`errno` 解释成具体原因再打印。

```cpp
#include <stdio.h>
void perror(const char *s);
```
　　perror函数将错误信息打印到标准错误输出,首先打印参数s所指的字符串,然后打印:号,然后根据当前`errno` 的值打印错误原因。例如:

```cpp
#include <stdio.h>
#include <stdlib.h>
int main(void)
{
	FILE *fp = fopen("abcde", "r");
	if (fp == NULL) {
		perror("Open file abcde");
		exit(1);
	}
	return 0;
}
```
　　如果文件abcde 不存在,`fopen()` 返回-1并设置`errno` 为`ENOENT`,紧接着perror函数读取`errno` 的值,将`ENOENT`解释成字符串No such file or directory 并打印,最后打印的结果是: `Open file abcde: No such file or directory 。`
　　需要注意的是，`perror()`是个系统函数，调用它的过程中，有可能会改变`errno`变量。所以一个系统函数错误返回后应该马上检查`errno` ,在检查`errno` 之前不能再调用其它系统函数。
　　`perror()`已经很好用了，`strerror()`用来干什么呢？
　　strerror 函数可以根据错误号返回错误原因字符串。

```cpp
#include <string.h>
char *strerror(int errnum);
```
　　有些函数的错误码并不保存在errno 中,而是通过返回值返回,就不能调用perror打印错误原因了,这时strerror 就派上了用场:

```cpp
fputs(strerror(errno), stderr);
```


###系统Unbuffered I/O函数
  全角字符如何输入？？全角的空格如何输入？
