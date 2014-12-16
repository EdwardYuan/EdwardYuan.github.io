---
layout: post
title:  "面向对象编程方法的反思"
date:   2014-12-04 18:46
categories: Programming
---

“面向对象”是每个计算机系学生一定会接触到的概念，也是每个程序员一定会接触到的概念，连我学习测控专业的表弟都问我什么叫“面向对象”。对于这一概念今日的红火，恐怕连艾伦·C·凯创造它时也未曾想到。

面向对象思想提出“一切皆是对象”的说法，并以此应用于计算机程序设计中，提出了面向对象的三个基本要素：封装、继承和多态。
所谓封装，是指把事物抽象为类，暴露对外接口，隐藏内部实现。
所谓继承，是指利用现有类的功能，在无需重写原有类的情况下进行扩展。
所谓多态，是指将父类设置成为和起派生类对象相等的技术。简单说即允许将派生类类型的指针赋给父类类型的指针。

事实上，我们在最初接受此种编程范式时通常是默认了其正确性。之后便尽可能在实际中应用它，却鲜少提出过反对或是怀疑。当我们默认了其正确性之后，无论由此引起何种问题，我们都只会认为其不够完善或是自己使用不当，绝不会去质疑方法本身，因此，在面向对象遇到问题之后，便产生了例如设计模式这类理论和方法来掩盖其先天不足。

对于面向对象的问题，我发现了以下几点：

首先，面向对象方法使我们编写了大量本不必要的代码。例如，当我们要写一个swap函数交换两个变量的值，如果用C语言代码可以这样写：

{% highlight c %}
void swap(int *px, int *py); // 函数体此处略过，这个应该是最基础的内容，想不起来可以翻阅K&R的《C程序设计语言》
{% endhighlight %}
而使用面向对象语言比如java则会是这样：
{% highlight java %}

class ClassA{
    private int value;
    public ClassA(int i) {this.value=i;}
    public void setValue(int value){this.value=value;}
    public int getValue(){return this.value;}
    public static void swap(ClassA a, ClassA b){
        ClassA c=new ClassA(a.getValue());
        a.setValue(b.getValue());
        b.setValue(c.getValue());
    }
}

{% endhighlight %}
很明显，c代码要比java简洁得多。本来一个函数可以解决的问题，OO却告诉我们需要一个类，于是就多出了许多代码。我认为编程时一条重要的原则就即：no more, no less. 上述java代码很难使我们认为它时美的。






