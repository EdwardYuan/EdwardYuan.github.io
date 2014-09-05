---
layout: post
title:  "理解Erlang OTP gen_server行为模式"
date:   2014-09-05 13:11
categories: Erlang
---


OTP的gen_server行为模式，在《Erlang程序设计》和《Erlang/OTP并发编程实战》两本书里都主要介绍如何使用，至于其中原理提到的并不多，以下内容主要介绍工作原理，主要来自官方的文档。

结构：
	 通用服务器模块 gen_server
	 回调模块 module(例如：my_bank.erl)

调用关系：

	通用服务器模块								回调模块

	gen_server:start_link            ------------>             Module:init/1

	gen_server:call
	gen_server:multi_call            ------------>             Module:handle_call/3

	gen_server:cast
	gen_server_abcast                ------------>             Module:handle_cast/2

	--                               ------------>             Module:handle_info/2
	--                               ------------>             Module:terminate/2
	--                               ------------>             Module:code_change/3

 当回调函数调用失败或返回错误值时，gen_server会中止。
 gen_server不会自动跟踪退出信号。