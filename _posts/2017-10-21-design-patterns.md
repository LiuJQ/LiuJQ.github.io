---
layout: post
title: 设计模式概要
tags: [Java, 设计模式，最佳实践]
---

## 目录
* [简介](#设计模式简介)
* [设计原则](#设计模式原则)
* [模式使用](#设计模式的使用)
    * [共同平台](#开发人员的共同平台)
    * [最佳实践](#最佳的实践)
* [模式类型](#设计模式的类型)
* [常见模式](#常见的几个设计模式)
    * [单例模式](#常见的几个设计模式)
    * [观察者模式](#常见的几个设计模式)
    * [工厂模式](#常见的几个设计模式)
    * [建造者模式](#常见的几个设计模式)
    
# 设计模式简介
&emsp;&emsp;设计模式是一套被反复使用的、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。 毫无疑问，设计模式于己于他人于系统都是多赢的，设计模式使代码编制真正工程化，设计模式是软件工程的基石，如同大厦的一块块砖石一样。项目中合理地运用设计模式可以完美地解决很多问题，每种模式在现实中都有相应的原理来与之对应，每种模式都描述了一个在我们周围不断重复发生的问题，以及该问题的核心解决方案，这也是设计模式能被广泛应用的原因。

# 设计模式原则
&emsp;&emsp;设计模式主要基于以下面向对象设计原则：
- 对接口编程而不是对实现编程；
- 优先使用对象组合而不是继承；

# 设计模式的使用
&emsp;&emsp;设计模式在软件开发中的两个主要用途
## 开发人员的共同平台
&emsp;&emsp;设计模式提供了一个标准的术语系统，且具体到特定的情景。例如，单例设计模式意味着使用单个对象，这样所有熟悉单例设计模式的开发人员都能使用单个对象，并且可以通过这种方式告诉对方，程序使用的是单例模式。
## 最佳的实践
&emsp;&emsp;设计模式已经经历了很长一段时间的发展，它们提供了软件开发过程中面临的一般问题的最佳解决方案。学习这些模式有助于经验不足的开发人员通过一种简单快捷的方式来学习软件设计。

# 设计模式的类型
&emsp;&emsp;设计模式总共有 23 种设计，这些模式可以分为三大类：创建型模式（Creational Patterns）、结构型模式（Structural Patterns）、行为型模式（Behavioral Patterns）。

| **分类**  | **模式** |
| :-----------: | ----------- |
| **创建型模式**  | 工厂模式(Factory Pattern)<br> 抽象工厂模式(Abstract Factory Pattern)<br> 单例模式(Singleton Pattern)<br> 建造者模式(Builder Pattern)<br> 原型模式(Prototype Pattern)  |
| **结构型模式**  | 适配器模式（Adapter Pattern）<br>桥接模式（Bridge Pattern）<br>过滤器模式（Filter、Criteria Pattern）<br>组合模式（Composite Pattern）<br>装饰器模式（Decorator Pattern）<br>外观模式（Facade Pattern）<br>享元模式（Flyweight Pattern）<br>代理模式（Proxy Pattern）<br> |
| **行为型模式**  | 责任链模式（Chain of Responsibility Pattern）<br>命令模式（Command Pattern）<br>解释器模式（Interpreter Pattern）<br>迭代器模式（Iterator Pattern）<br>中介者模式（Mediator Pattern）<br>备忘录模式（Memento Pattern）<br>观察者模式（Observer Pattern）<br>状态模式（State Pattern）<br>空对象模式（Null Object Pattern）<br>策略模式（Strategy Pattern）<br>模板模式（Template Pattern）<br>访问者模式（Visitor Pattern）<br>  |

# 常见的几个设计模式
* 单例模式
* 观察者模式
* 工厂模式
* 建造者模式
