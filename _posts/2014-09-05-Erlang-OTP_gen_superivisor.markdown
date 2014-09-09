---
layout: post
title:  "理解Erlang OTP gen_superivisor 行为模式"
date:   2014-09-05 13:11
categories: Erlang
---

本文介绍OTP行为模式中的gen_supervisor行为，即监督模式，此行为模式主要用于监督和管理服务器，在服务器崩溃后重启进程。

_ _ _


##### 进程树结构 #####

<img src="/images/gen_supervisor.png" class="left" />

