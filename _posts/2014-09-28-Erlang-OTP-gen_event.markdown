---
layout: post
title:  "理解Erlang OTP gen_event"
date:   2014-09-05 13:11
categories: Erlang
---

##### 目录 #####
	A. gen_event作用
	B. gen_event 与 gen_server 及 supervisor之间的关系
	C. gen_event 细节描述
		1. 各函数作用、参数、返回值，哪几个函数是必需的
		2. 每个函数是使用方法
	D. 一个具体的例子