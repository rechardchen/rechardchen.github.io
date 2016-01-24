---
layout: post
category: code
title: "Lua源码阅读笔记(0)"
tags: "Lua"
---

## Lua源码概述

[Lua](http://www.lua.org)是一门小而美的脚本语言。

小，主要指的是它的语言标准简洁从而极为容易上手，同时它的小也体现在代码量上。以目前的版本（2016年1月）V5.3.2为例，整个源码包的大小是281KB，解压后统计src目录下的源码总的行数是19768行（包括详细的注释在内）。而[CPython3](http://www.python.org)的实现往往一个C源文件就要洋洋洒洒的超过几千行。

美，主要指的是它的效率极高，以及优雅的设计。Lua是主流脚本语言中执行效率最高的，而设计上，简洁的C API设计、元表、环境都是极其出彩的部分。

## 源码下载和实验环境

在[官网](http://www.lua.org)找到[V5.3.2](http://www.lua.org/ftp/lua-5.3.2.tar.gz)版本下载，解压后可以在src目录下找到所有的源文件。\*nix系统下直接make即可，而在Windows下，新建VS工程后将所有的代码文件（除去luac.c）添加到工程中即可编译运行交互式解释器。

如果是以阅读源码为目的的话，强烈推荐使用Visual Studio2013以上版本。

## 模块概述

* lmathlib,lstrlib,ltablib等内嵌库。读这些源码可以熟悉lua的C API使用方式，然后就可以熟练的为lua写C语言扩展。lstrlib里有一个600行的正则表达式实现值得一看
* lapi是Lua的C API的实现
* lobject是对象操作函数，看Lua中如何定义对象
* lstate是lua的全局状态机。实际上，真正的全局状态机是global\_State，而lua\_State对应的是lua的线程
* lopcodes定义了lua字节码指令的格式，看lua是如何把指令压缩到一个int中
* lvm,ldo是lua虚拟机和运行时线程堆栈的实现。ldo中也包括了协程的实现，协程库对外的接口则是通过内嵌库lcorolib提供出来
* ltm是元方法的实现
* llex,lparse是lua的手写的语法解析器，lcode是代码生成器。ldump序列化编译的lua字节码，lundump反序列化
* lmem内存管理接口,lgc垃圾收集

总的来说，lua的模块划分清晰。每个模块被写在单独的C源文件中，有自己的API前缀，比如运行时堆栈ldo模块的内部API的格式就是luaD\_xxx，虚拟机模块的API则以luaV\_开头。而对外暴露的接口则定义在lua.h和lauxlib.h中，以lua\_和luaL\_为前缀。