---
layout: post
title:  "理解Erlang OTP gen_superivisor 行为模式"
date:   2014-09-05 13:11
categories: Erlang
---

本文介绍OTP行为模式中的gen_supervisor行为，即监督模式，此行为模式主要用于监督和管理服务器，在服务器崩溃后重启进程。 本文介绍监督者模式的原理和结构，具体示例代码请参考官方文档和《Erlang/OTP并发实战》一书。

_ _ _



##### 进程树结构 #####

<img src="/images/gen_supervisor.png" class="left" width="700" />

- - -


##### 监督者调用结构 #####

module:start_link/0  --->  supervisor:start_link/3  --->  module:init/1.

其中 supervisor:start_link/3为如下形式：

{% highlight erlang %}
supervisor:start_link({local, Name}, Module, Args);
或
supervisor:start_link({global, Name}, Module, Args).
{% endhighlight %}

该函数创建一个新的监督者进程。
其中参数如下：

	{local, Name} 和 {global, Name} 与 gen_server行为模式中意义相同。
	Module %% 为监督者模块名
	Args %% 传入参数

新的监督者进程将调用监督者模块中的module:init([])函数以期返回{ok, StartSpec}. 

init形式如下：
{% highlight erlang %}
init(...) ->
	{ok, { {RestartStrategy, MaxR, MaxT}, [ChildSpec, ...] } }.
{% endhighlight %}

例如：

{% highlight erlang %}
init(_Args) ->
	{ok, {one_for_one, 1, 60}, [{ch3, {ch3, start_link, []},
	permanent, brutal_kill, worker, [ch3]}]}.
{% endhighlight %}

参数说明：

	MaxR %% 在MaxT时间内最大重启次数
	MaxT %% 限定时间，单位为秒

最大重启频率：

如果在过去MaxT秒内重启次数超过了MaxR，监督者将会终止所有子进程然后终止自身进程。
当监督者进程终止时，下一优先级的监督者可能做如下操作：
- 重启监督者
- 终止监督者
- 终止自身进程

重启策略:

在init/1 中返回值 RestartStrategy 表示监督者重启策略，其可能取值如下：
- one_for_one %% 当一个进程终止时，只有它自身会被重启
- one_for_all %% 当一个子进程终止时，所有子进程（包括该子进程本身）都将会被重启
- rest_for_one %% 当一个子进程终止时，所有“剩余的”子进程和它本身将本重启

子进程规范：

子进程规范是一个元组，用来描述受监督的进程。该元组结构如下：

	{Id, StartFunc, Restart, Shutdown, Type, Modules}

其中参数：

	Id %% 监督者在系统内部识别子进程规范的名称，一般使用模块名
	StartFunc %% 用于启动进程的三元组 {M, F, A}
	Restart %% 指定子进程何时被重启
		取值为：
		- permanet %% 始终重启
		- temporary %% 永不重启
		- transient %% 非正常终止时重启
	Shutdown %% 指明如何终止进程
		取值为：
		- brutal_kill %% 调用 exit(Child, kill)立即终止子进程
		- 整型数值 %% 监督者告知子进程调用 exit(Child, shutdown)来终止，
		  之后等待退出信号返回，如果在指定时间内未收到退出信号，则调用exit(Child, kill)
		- infinity %% 通常表示子进程也是一个监督进程的情况，监督者将给子进程留出足够的时间来终止
	Type %% 表示子进程是监督者(supervisior)还是工作者(worker)
	Modules %% 所依赖模块名称的列表


- - -

添加子进程：

我们可以通过调用以下函数来实现向一个静态监督树中田间动态子进程：

{% highlight erlang %}
supervisor:start_child(Sup, ChildSpec).
{% endhighlight %}

其中：
	sup %% 进程pid或监督者名称
	ChildSpec %% 子进程规范

注意：当监督者进程消亡并重启后，所有动态添加的子进程将会丢失。

终止子进程：

调用以下函数可以终止任何动态或静态的子进程：

{% higlight erlang %}
supervisor:terminate_child(Sup, Id).
{% endhighlight %}

终止后的子进程将会有以下函数删除：

{% highlight erlang %}
supervisor:delete_child(Sup, Id).
{% endhighlight %}

其中：

	Id %% 指在子进程规范中定义的Id

同样，删除的动态添加的子进程将在监督者重启后丢失。


- - -

Simple_One_For_One 监督者：
当监督者的子进程重启策略设置为 simple_one_for_one时，所有子进程都将是动态添加的。此时，当监督者启动时不会启动任何子进程，所有子进程都将由以下函数动态添加：

{% highlight erlang %}
supervisor:start_child(Sup, List).
{% endhighlight %}


- - -


- - -

