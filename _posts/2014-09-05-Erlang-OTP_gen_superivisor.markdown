---
layout: post
title:  "理解Erlang OTP gen_superivisor 行为模式"
date:   2014-09-05 13:11
categories: Erlang
---

本文介绍OTP行为模式中的gen_supervisor行为，即监督模式，此行为模式主要用于监督和管理服务器，在服务器崩溃后重启进程。 本文介绍监督者模式的原理和结构，具体示例代码请参考官方文档和《Erlang/OTP并发实战》一书。

_ _ _


##### 进程树结构 #####

<img src="/images/gen_supervisor.png" class="left" />


- - -




##### 监督者调用结构 #####

module:start_link/0  --->  supervisor:start_link/3  --->  module:init/1.

其中 supervisor:start_link/3为如下形式：

{% highlight erlang %}
supervisor:start_link({local, Name}, Module, Args)；
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
	{ok, {{RestartStrategy, MaxR, MaxT}, [ChildSpec, ...]}}.
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









