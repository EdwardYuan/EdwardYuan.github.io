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

_ _ _

调用关系：

	通用服务器模块							回调模块

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

_ _ _

下面看看各函数的参数和返回值：

{% highlight erlang %}
start_link(Module, Args, Options) -> Result
start_link(ServerName, Module, Args, Options) -> Result
{% endhighlight %}

参数类型：

    ServerName = {local, Name}  %% 在本地将gen_server注册为Name
	| {global, GlobalName}　　   %% 注册全局名称GlobalName
	| {via, Module, ViaName}    %% 使用模块所代表的注册入口进行注册

其中：

    Module 为原子(atom)， Args为列表(term)， Options是由选项构成的列表[Option]，
	可取的值为：
	{debug, Dbgs}         %% 调试
	| {timeout, time}     %% 设置超时
	| {spawn_opt, Sopts}  %% 将Sopts选项列表传递给spawn_opt，创建一个新的gen_server进程

    参数 Dbgs 为一个列表[Dbg]， 可能的取值如下：
    Dbg = trace | log | statistics | {Log_to_file, FileName} | {install, {Func, Function}}
    
    Sopts = [term()]
    
    start_link可能的返回值如下：
	{ok, Pid} | ignore | {error, Error}
    其中Pid为进程id(pid())
    失败时，Error的取值为
	{already_started, Pid}  %% 进程已启动
	| term()

前面已经说过，gen_server:start_link会调用回调模块的init/1函数，该函数在出错时会返回{error, Reason}.

另一种启动方式是 

{% highlight %}
start(Module, Args, Options) -> Result
start(ServerName, Module, Args, Options) -> Result
{% endhighlight %}

这两种方式的区别在于: start_link创建的进程将作为监控树的一部分；而start创建的进程则作为一个独立的gen_server进程，它没有相应的监督者。 深入内容请参阅监督者行为模式的相关内容。


{% highlight %}
call(ServerRef, Request) -> Reply
call(ServerRef, Request, Timeout) -> Reply
{% endhighlight %}

call函数会通过向gen_server的ServerRef发送请求进行同步调用，也就是说发送消息后该函数会等待直到收到回应或超时。gen_server模块会调用回调函数 Module:handle_call/3 来处理请求。

	ServerRef可以取以下值：
- pid
- Name, 当gen_server在本地注册时
- {Name, Node}, 当gen_server全局注册时
- {via, Module, ViaName}, 当gen_server通过可选的进程注册入口注册时

    Request可以是任意Erlang数据项，它会被作为参数传递给 Module:handle_call/3.

    参数Timeout代表超时的时间，以毫秒为单位，取任意大于0的整数，或者infinity表示永不超时.

    返回值 Reply 由回调函数 handle_call/3定义.
	


