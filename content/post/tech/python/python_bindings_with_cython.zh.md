---
author: "Dennis Xie"
title: "使用Cython实现Python Bindings"
date: 2022-08-10T21:30:26+08:00
description: "Cython的简单入门使用"
draft: false
tags: ["python"]
categories: ["tech"]
postid: "7fa8acd353713dd9afc254459d49a009211f7064"
---
# 背景

## 语言交互接口(FFI)

一般操作系统会提供系统调用的C API。这种说法其实并不正确，但是我们假设这个说法是正确的。毕竟很多Linux发行版都是自带了GCC编译器的，我们就简单的认为提供了系统调用的C API。然后我们思考一下这么一个问题，一个非C语言是怎么实现和操作系统交互的？主要有两种方式实现，一种是使用中断来实现系统调用，另外一种方式是间接使用C语言的系统调用API。

间接调用C语言的API涉及到了语言交互接口(FFI, Foreign Function Interface)的概念。不少编程语言就通过使用FFI来实现系统调用。很多语言把FFI叫做"Language Bindings"，比如Python Bindings。还有一些语言有自己的叫法，比如Java的JNI。通过使用FFI，像Python这些语言就可以比较简单的实现系统调用。

## Python Bindings

# 使用Cython实现Python Bindings

## Cython简介

## Cython基础使用方法

### Hello, World!
### 静态类型
### 定义函数

## 调用C标准库函数示例

## 构建及调用C代码示例
### C源代码
### Cython代码
### Makefile代码
### 使用

## 回调函数示例
### C源代码
### Cython代码
### 使用

# 总结

# 名词解释
# 引用
1. [Cython official document](https://cython.readthedocs.io/en/latest/index.html)
2. [Shared libraries with GCC on Linux](https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html#fn:pic)
3. [Makefile tutorial](https://makefiletutorial.com/)
