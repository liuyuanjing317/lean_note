#### java_设计模式_七大原则_笔记

设计模式的目的：

编写软件过程中，程序员面临着来自耦合性，内聚性以及可维护性，可扩展性，重用性，灵活性等多方面的挑战，设计模式是为了让程序(软件)，具有更好

l)代码重用性(即:相同功能的代码，不用多次编写)

2)可读性(即:编程规范性,便于其他程序员的阅读和理解)

3)可扩展性(即:当需要增加新的功能时，非常的方便，称为可维护)

4)可靠性(即:当我们增加新的功能后，对原来的功能没有影响)

5)使程序呈现高内聚，低耦合的特性

 

1.设计模式的七大原则

1.单一原则：一个类或者接口只负责一项职责

2.接口隔离：客户端不应该依赖他不需要的接口，一个类对另一个类得依赖应该建立在最小接口上

3.依赖倒转：

   1)高层模块不应该依赖低层模块，二者都应该依赖其抽象

   2)抽象不应该依赖细节，细节应该依赖抽象

   3)依赖倒转(倒置)的中心思想是面向接口编程

   4)依赖倒转原则是基于这样的设计理念:相对于细节的多变性，抽象的东西要稳定的多。以抽象为基础搭建的架构比以细节为基础的架构要稳定的多。在java中，抽象指的是接口或抽象类，细节就是具体的实现类

   5)使用接口或抽象类的目的是制定好规范，而不涉及任何具体的操作，把展现细节的任务交给他们的实现类去完

4.里斯替换：

5.开闭原则：对扩展开放，对修改关闭

6.迪米特原则：

 