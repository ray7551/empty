---
published: false
layout: default
title: Nginx 学习笔记
keywords: Nginx
tags: Nginx
---

### 变量
Nginx 变量的创建和赋值操作发生在不同的阶段。Nginx 变量的创建发生在 Nginx 配置加载的时候，即 Nginx 启动的时候；而赋值操作则发生在实际处理请求的时候。这意味着：
1. 不创建而直接使用变量会导致启动失败
2. 我们无法在请求处理时动态地创建新的 Nginx 变量

Nginx 变量的生命期是只在一个请求之内。  
Nginx 变量一旦创建，其变量名的可见范围就是整个 Nginx 配置，甚至可以跨越不同虚拟主机的 server 配置块。
```
server {
  listen 8080;

  location /foo {
    echo "foo = [$foo]";
  }

  location /bar {
    set $foo 32;
    echo "foo = [$foo]";
  }
}
```

命令行上用 curl 工具访问这两个接口的结果：

```
$ curl 'http://localhost:8080/foo'
foo = []

$ curl 'http://localhost:8080/bar' # 赋值操作只会在访问 /bar 的请求中执行。
foo = [32]

$ curl 'http://localhost:8080/foo' # 每个请求都有所有变量的独立副本，上次对变量的赋值不影响本次请求
foo = []
```

###变量插值
变量只能存放一种类型的值(字符串)

```
server {
  listen 8080;

  location /test {
    set $first "hello "; # set 指令创建变量并赋值给变量，变量必须以 $ 开头
    echo "${first}world"; # 类似 perl 和 php 的变量插值
  }
}
```

### 预定义变量