---
layout: post
title:  杂谈现代高级编程语言[转]
date:   2015-11-05 11:16
categories: programming
---

之前看到过这篇文章，不记得是谁的了，当时存了PDF放在机器上，正好刚才看到了，就贴出来吧，好象是陈硕的文章。如有其它问题请右键告知，如有法律问题，亦请告知，我会及时删除。以下是原文：

杂谈现代高级编程语言


几个月之前，Slashdot 转载了Robert Harper教授的一篇博客，说卡内基梅隆大学计算机系把”面向对
象编程“从大一新生的必修课中删掉了，其原因是
Object-oriented programming … is both anti-modular and anti-parallel by its very
nature.
这两个原因（anti-modular和anti-parallel）都是很重的指责了；尤其是anti-modular，因为OO的基
本思想通常被理解成“封装”，从而实现模块化。
我是在1995年第一次听说“面向对象”（Object Oriented）这个说法。当时在学习正在成长过程中的
C++，用的是Borland C++ 1.0。从那时开始的很多年里，”类“（class）、“对象”（object）和
“方法”（methods），以及在这些概念之上构建的”继承“（inheritance）和”多态
“（polymorphism）都是我理解中OO最核心的思想。我猜大多数程序员在这方面的认识都和我差不
多。
但是“封装”真的是OO的本质嘛？直到最近为了给iPhone写个玩具程序而学习Objective-C（一种非
常古老和原始的面向对象编程语言）的时候，才注意到早在1998年，OO之父Alan Kay就曾经在一篇
邮件中说，他很后悔发明了“object”这个词，从而误导大家，把注意力都集中到“封装”，而忽视了
OO的本质——messaging（消息传递）。Alan Kay的原话是：
The big idea is “messaging” … . The key in making great and growable systems is
much more to design how its modules communicate rather than what their internal
properties and behaviors should be.
Objective-C的设计是非常强调“消息传递”（messaging）的——对一个object的method的调用，
被称为“给这个object发了一个消息”。为了突出调用method时指定的参数（parameters）实际上
是消息中的一些内容，Objective-C不惜把method的定义方式都做了相对于C的很大的修改，从而把
参数嵌入在method的名字里。比如在一个叫做myWebView对象中搜索一段文字，要求不区分大小写，
从前往后搜索，用Objective-C来描述是：
{% highlight objective-c %}
[myWebView searchFor:myString
direction:YES
caseSensitive:NO
wrap:YES]
{% endhighlight %}
而用C++或者Java来描述，则是
{% highlight java %}
myWebView.searchFor(myString, YES, NO, YES)
{% endhighlight %}
乍看上去，C++ 或者Java的方式更简短，但是Objective-C的方式更强调“发消息”。实际上，上面
Objective-C语句会被翻译成如下C函数调用：
{% highlight c %}
objc_msgSend(myWebView,
searchFor:direction:caseSensitive:wrap:,
myString, YES, NO, YES)
{% endhighlight %}
从强调messaging的角度看，Objective-C确实比C++和Java更符合Alan Kay对OO思想的描述。
OO中的messaging思想不仅体现在Objective-C语言以及在其上构建的NextSTEP/Cocoa GUI编程套
件上。在Cocoa因为Mac OS X和iPhone流行起来之前，很多人都接触过Qt（一种基于C++语言的
GUI开发套件）。Qt对messaing的支持比Objective-C/Cocoa更彻底——每个object可以发出若干
signal，每个signal可以触发这个object自己的或者其他objects的若干个slot。
有意思的是，为了支持messaging，Qt对C++语言做了扩展，而Objective-C对C语言做了扩展。这
两套扩展都利用了起源于C语言的“宏”机制（macro）。类似的做法也可以用于Java，前提是我们在
调用Java编译器之前，先调用一下cpp宏展开程序来预处理一下我们的Java程序。这事儿可以留待
Java爱好者们来搞？
不管是Objective-C还是Qt，都会“尽力”去检查一个object是否支持一个method（或者叫
message），但是并不禁止程序员向一个object发送一个它不认识的message（或者调用一个object
没有的method）。说“尽力”检查，是因为两者都不能保证检查的完备性。这是因为Objective-C和
Qt都支持多态；具体的说，接受message的object可能是表示为一个指针（指向object），所以直到
运行时候，当一个message抵达某个object的时候，系统才能（通过查这个object对应的message
list）知道这个object是否认识这个message。
这种灵活性在Google新推出的Go语言中也同样实现了，而且做的很极致——Go语言中没有class的
概念；换句话说，不需要是class类型的object才能有对应的方法（methods）——Go允许给几乎任
何类型附上methods。而且程序中可以很方便的检测一个object是否支持（一组）methods，比如：
{% highlight go %}
type Stringer interface {
String() string
}
s, ok := v.(Stringer) // Test whether object v implements "String()"
{% endhighlight %}
【和Go的这种灵活性类似的，Objective-C允许给已经定义了的class增加一些methods，而不需要
derive subclass；Objective-C的这种机制被称为category。和Go不同的是，一个category是对某个
class的一个扩展，而Go语言里完全没有class了（但是有interface的概念）。】
说到这里，我觉得差不多可以反过来理解Robert Harper教授对OO的评价了——其实Robert不是在
藐视OO，而是在指责很多imperative OO languages（我理解包括Java和未经Qt扩展的C++；详见
后述），认为这些语言没有完成实现OO中object messaging的核心思想，从而不算实现了“模块化
“（modulization）的思想。
上述都是关于程序的模块化。实际上，模块化的另一个主要方面是对“数据”（data)的模块化。从图灵
机和lambda-calculus开始，计算机科学家们就注意到程序和数据是统一的；比如在冯诺依曼的“二进
制存储电子计算机”模型里，程序和数据都是bit stream。即时我们在讨论高级编程语言的时候，程序
和数据也不应该被分开。因为现代数据操作和模块化的基础是并行程序（parallelism），而有效实现并
行的基础是程序的first-class表达，也就是把程序作为一种基本数据类型。
鉴于这篇帖子已经很长了，这段话就作为下一篇帖子的提纲吧。下一篇帖子里，我们来说说
XML、JSON、MessagePack、Protocol Buffers这些persistent data structure，以及用源于古老的
functional programming paradigm的Go语言和MapReduce实现的并行数据操作。
