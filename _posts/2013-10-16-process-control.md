---
published: true
layout: default
title: Linux C 学习笔记（2）：进程的建立与运行
keywords: 进程，C语言, Linux
tags: Linux, C, 进程
---

### 进程的概念

　　进程是正在运行过程中的程序的实例，是操作系统分配资源的基本单位。一个进程实体包括以下三部分内容：

* 程序代码
* 数据
* 进程控制块

　　每个进程在内核中都有一个进程控制块（PCB）来维护进程相关的信息。Linux 内核的进程控制块是 task_struct 结构体，其中包括进程 id，当前目录，umask 掩码，进程状态，文件描述表，和信号相关的信息，用户 id 和组 id，进程可以使用的资源上限等信息。  
　　一个现有的进程可以复制出一个新进程，原来的进程称为父进程（Parent Process）,新进程称为子进程（ChildProcess）。这样，一个进程又可以启动另一个进程，总体呈现出一个树的层次结构，树的根节点，是一个控制进程，它是名为 `init` 的程序的执行，是系统中所有进程的祖先。我们可以用命令 `pstree` 查看这个树形结构。  
　　在 Shell 下输入命令可以运行一个程序,是因为 Shell 进程在读取用户输入的命令之后会调用 fork 复制出一个新的Shell 进程,然后新的 Shell 进程调用 exec 函数执行新的程序。例如在 bash 中执行程序 /bin/ls，如下图所示：  
　　![block](/images/post/shell_fork.png "shell fork")
　　上面说到的复制出一个新进程(fork)，指的是PCB的复制，也就是说，子进程的当前目录，umask 掩码，用户 id 等信息与父进程的这些信息是相同的。有一个例外，子进程中的进程 id 与父进程的 id 是不同的。  
　　在子进程中调用 exec 函数执行新程序时，这个子进程的程序代码部分，数据部分都会完全被新程序替换，从新的程序开始执行。调用 exec 程序并不创建新进程，所以调用 exec 前后，这个子进程的 id 是不变的。  
　　Linux提供一些进程控制方面的系统调用，其中最重要的有以下几个：

1. `fork()`。它通过复制进程来建立新进程。
2. exec系列函数。这些函数都以 exec 开头，都完成相同的功能：用一个新程序覆盖原内存空间，实现进程的转变。
3. `wait()`。它提供初级的进程同步措施，能使一个进程等待，直到另一个进程结束为止。
4. `exit()`。用于终结一个进程的运行。 


### 进程的建立

　　我们可以使用 fork() 建立新进程。fork() 在 Linux系统库`unistd.h`中声明如下：
```cpp
pid_t fork(void);
```
　　正如它的名字意义告诉我们的，它执行之后，进程的执行会呈现出“分叉”的状态。例如：
```cpp
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
	pid_t pid;

	printf("One\n"); //如果此处无"\n"，这个"One"会被放到缓存中，然后被复制到子进程空间，最终运行结果会多输出一个"One"
	pid = fork();
	printf("Two\n");
}
```
　　图解如下：
	![block](/images/post/fork_before_after.png "fork before and after")
　　它分为 fork() 调用前和调用后两部分。调用前PC(程序计数器)指向当前执行的语句。这时它指向第一个 printf 语句。调用 fork() 后进程 A 和 B 一起运行，进程 B 执行与父进程 A 一样的程序。两个 PC 都指向第二个 printf 语句,即 fork() 调用之后的语句。也就是说，A 和 B 都从程序的相同点开始执行。  
　　一般我们用 fork 是为了在子进程中执行新程序，但是子进程与父进程用的是同样的代码，我们又不想让父进程也做同样的事，否则失去了 fork 的意义。那么能不能找到一个办法让子进程执行子进程的代码，父进程继续执行父进程的代码呢？也就是说，如何区分父进程与子进程呢？  
　　这就要说到 fork() 特殊的返回值。它返回一个 pid_t 类型的值 pid，可以用来区分父进程与子进程。如果 fork() 调用成功，在父进程中，返回的 pid 是一个非 0 的正整数（是新生成的子进程的进程 id），而在子进程中，返回的 pid 被置为0。  
　　从根本上说 fork 是从内核返回的,内核自有办法让父进程和子进程返回不同的值。  
　　fork 的返回值这样规定是有道理的。fork 在子进程中返回 0,子进程仍可以调用 `getpid()` 得到自己的进程 id,也可以调用 `getppid()` 得到父进程的 id。在父进程中用 `getpid()` 可以得到自己的进程id,然而要想得到子进程的 id,只有将 fork 的返回值记录下来,别无它法。  

### 进程的运行
　　有六种以 exec 开头的函数,统称 exec 函数：
```cpp
#include <unistd.h>
int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execve(const char *path, char *const argv[], char *const envp[]);
```
　　这些函数如果调用成功则加载新的程序从启动代码开始执行,不再返回,如果调用出错则返回 -1,所以 exec 函数只有出错的返回值而没有成功的返回值。  
1. 带有字母 l（表示 list）的 exec 函数要求将新程序的每个命令行参数都当作一个参数传给它,命令行参数的个数是可变的,因此函数原型中有"..."，使用时，"..." 中的最后一个可变参数应该是 NULL。
2. 带有字母v(表示vector)的函数,则应该先构造一个指向各参数的指针数组,然后将该数组的首地址当作参数传给它,数组中的最后一个指针也应该是 NULL,就像 main 函数的 argv 参数或者环境变量表一样。
3. 带字母 p（表示 path）的函数:  
  * 如果参数中包含 /,则将其视为路径名。
  * 否则视为不带路径的程序名,在 PATH 环境变量的目录列表中搜索这个程序。
4. 不带 p 的函数，第一个参数必须是程序的相对路径或绝对路径。  
5. 以 e（表示 environment）结尾的 exec 函数,可以把一份新的环境变量表传给它,其他 exec 函数仍使用当前的环境变量表执行新程序。  



