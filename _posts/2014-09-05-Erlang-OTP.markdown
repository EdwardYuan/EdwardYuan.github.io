---
layout: post
title:  "理解Erlang OTP gen_server行为模式"
date:   2014-09-05 13:11
categories: Erlang
---



OTP的gen_server行为模式，在《Erlang程序设计》和《Erlang/OTP并发编程实战》两本书里都主要介绍如何使用，至于其中原理提到的并不多，以下内容主要介绍工作原理，主要来自官方的文档。


######总览######


结构：
- 通用服务器模块 gen_server

- 回调模块 module(例如：my_bank.erl)

_ _ _

调用关系：

	通用服务器模块						回调模块

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

######gen_server函数详解######

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

    Module 为原子(atom)， Args为数据项(term)， Options是由选项构成的列表[Option]，
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

{% highlight erlang %}
start(Module, Args, Options) -> Result
start(ServerName, Module, Args, Options) -> Result
{% endhighlight %}

这两种方式的区别在于: start_link创建的进程将作为监控树的一部分；而start创建的进程则作为一个独立的gen_server进程，它没有相应的监督者。 深入内容请参阅监督者行为模式的相关内容。


{% highlight erlang %}
call(ServerRef, Request) -> Reply
call(ServerRef, Request, Timeout) -> Reply
{% endhighlight %}


call函数会通过向gen_server的ServerRef发送请求进行同步调用，也就是说发送消息后该函数会等待直到收到回应或超时。gen_server模块会调用回调函数 Module:handle_call/3 来处理请求。

	ServerRef可以取以下值：

	- Name, 当gen_server在本地注册时
	- {Name, Node}, 当gen_server全局注册时
	- {via, Module, ViaName}, 当gen_server通过可选的进程注册入口注册时

    Request可以是任意Erlang数据项，它会被作为参数传递给 Module:handle_call/3.

    参数Timeout代表超时的时间，以毫秒为单位，取任意大于0的整数，或者infinity表示永不超时.

    返回值 Reply 由回调函数 handle_call/3定义.

{% highlight erlang %}
cast(ServerRef, Request) -> ok
{% endhighlight %}


cast函数与call类似，其区别是call进行同步调用，而cast则是异步调用，即cast将立即返回，不论目标节点或通用服务器是否存在。其参数的类型也和call相同。

{% highlight erlang %}
reply(Client, Reply) -> Result.
{% endhighlight}

当回应无法在Module:handle_call/3中定义时，reply会向调用call或multi_call函数的客户端明确发送一个返回消息。

	Client必须是想回调函数提供的From参数
	Reply = term()   %% Reply是Erlang数据项
	Result = term()  %% Result是Erlang数据项

{% highlight erlang %}
enter_loop(Module, Options, State)
enter_loop(Module, Options, State, ServerName)
enter_loop(Module, Options, State, Timeout)
{% endhighlight %}

enter_loop函数在通用服务器中调用已存在的进程，此函数不返回任何值。调用的进程将会进入通用服务器的接收消息循环中并成为一个通用服务器。其中参数与前面介绍函数中同名参数类型相同，此函数通常在比gen_server行为模式所提供的初始化更加复杂的情况下使用。

---

######回调函数######

{% highlight erlang %}
Modult:init(Args) -> Result
{% endhighlight %}

gen_server:start或gen_server:start_link函数将会调用此函数进行初始化操作。其参数类型如下

	Args = term()  %% 启动函数所需的参数
	Result = {ok, State} | {ok, State, Timeout} | {ok, State, hibernate} | {stop, Reason} | ignore
	State = term()
	Timeout = int() >= 0 | infinity
	Reason = term()

其中hibernate参数表示进程将会进入休眠状态等待下一条消息到来(调用 proc_lib:hibernate/3)。
如果初始化失败，将会返回{ok, Reason}。

{% highlight erlang %}
Module:handle_call(Request, From, State) -> Result
{% endhighlight %}

gen_server:call或gen_server:multi_call将会调用此函数来处理请求。其中参数类型：

	Request = term()
	From = {pid(), Tag}
	State = term()
	Result = {reply, Reply, NewState} | {reply, Reply, NewState, Timeout} | {reply, Reply, NewState, hibernate} | {noreply, NewState, Timeout} | {noreply, NewState, hibernate} | {stop, Reason, Reply, NewState} | {stop, Reason, NewState}
	Reply = term()
	NewState = term()
	Timeout = int() >= 0 | infinity
	Reason = term()

当此函数返回{reply, ...}时，响应Reply将会发送给From参数所指的进程；返回{noreply, ...}时，gen_server将会以新状态NewState继续执行，如果需要返回响应，则必须显式调用gen_server:reply/2。
返回{stop, Reason, Reply, NewState}时，函数会把Reply返回给From的进程，{stop, Reason, NewState}则需要明确调用gen_server:reply/2返回。然后gen_server会调用Module:terminate(Reason, NewState}。

{% highlight erlang %}
Module:handle_cast(Request, State) -> Result.
{% endhighlight %}

如前所述，gen_server:cast/2 或 gen_server:abcast/2, 3将会调用此函数来处理发来的请求。其参数与handle_call中相似。

{% highlight erlang %}
Module:handle_info(Info, State) -> Result.
{% endhighlight %}

gen_server收到同步和异步以外的消息或系统消息以及超时的情况下会调用此函数，其参数即返回如下：

	Info = timeout | term()
	State = term()
	Result = {noreply, NewState} | {noreply, NewState, Timeout} | {noreply, NewState, hibernate} | {stop, Reason, NewState}
	NewState = term()
	Timeout = int() >= 0 | infinity
	Reason = normal | term()

{% highlight erlang %}
Module:terminate(Reason, State)
{% endhighlight %}

gen_server将要终止时会调用此函数，它的工作与Module:init/1相反，同时进行其他必要的清理工作。其参数：

	Reason = normal  %% 正常终止
	| shutdown %% 当设置了跟踪退出信号 或 监督者子进程定义了特定的整数超时值时
	| {shutdown, term()}
	| term()
	State = term()

任何 normal, shutdown, {shutdown, Term} 以外原因出现时，gen_server都会假定出错并用error_logger:format/2报告错误。

{% highlight erlang %}
Module:code_change(OldVsn, State, Extra) -> {ok, NewState} | {error, Reason}
{% endhighlight %}

当版本更新时会使用此函数进行模块热替换，其中参数比较简单：
	OldVsn = Vsn | {down, Vsn}
		Vsn = term()
	State = NewState = term()
	Extra = term()
	Reason = term()

成功时此函数会返回更新后的内部状态，如果由于某种原因出错，升级将会失败并回滚到之前状态。

{% highlight erlang %}
Module:format_status(Opt, [PDict, State]) -> Statue.
{% endhighlight %}

此函数是可选的，当请求服务器状态或异常终止并记录错误日志是gen_server进程会调用它。其参数如下：

	Opt = normal | terminate
	PDict = [{Key, Value}]
	State = term()
	Status = term()

PDict是gen_server进程字典的当前值，State是gen_server的内部状态。


- - -

######小结######

综上所述，gen_server行为模式的基本流程如下：
回调模块的start调用gen_server:start_link（或gen_server:start）来启动服务器，并进入主循环，在回调模块的接口函数（处理具体的业务逻辑）中调用gen_server的call或cast来处理请求，在gen_server的call和cast中进一步调用回调模块中的handle_call和handle_cast来执行具体的业务逻辑处理，回调模块的terminate来终止进程。

其中通用服务器模块和回调模块的接口函数在上面做了简单说明，具体各个函数内部细节如何实现以及调用了哪些相关库函数请查阅官方文档。

完




