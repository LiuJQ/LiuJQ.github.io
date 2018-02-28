---
layout: post
title: Design Patterns -- 建造者模式
tags: [Java, 设计模式, Design Patterns]
---

&emsp;&emsp;介绍建造者（Builder）模式之前，让我们来看一个案例。
## 案例
&emsp;&emsp;假设Bill是某银行软件开发中心Java团队的一员，他设计了一个银行账户Java类BankAccount 1.0版本。看上去差不多是下面这个样子的：(PS:用double作为金额的数据类型是不可靠的，演示方便)
```java
public class BankAccount {
    private long accountNumber;
    private String owner;
    private double balance;
    public BankAccount(long accountNumber, String owner, double balance) {
        this.accountNumber = accountNumber;
        this.owner = owner;
        this.balance = balance;
    }
    //Getters and setters omitted for brevity.
}
```
&emsp;&emsp;这是合理直观的实现方式，我们可以用它来初始化一个银行账户：
```java
BankAccount account = new BankAccount(123L, "Bart", 100.00);
```
&emsp;&emsp;Bill的主管考虑到客户银行账户的信息太简单，要加入一些信息诸如存款月利率、开户日期、开户分行等。Bill考虑了一下，不是很复杂，于是他更新了银行账户Java类BankAccount 2.0版本：
```java
public class BankAccount {
    private long accountNumber;
    private String owner;
    private String branch;
    private double balance;
    private double interestRate;
    public BankAccount(long accountNumber, String owner, String branch, double balance, double interestRate) {
        this.accountNumber = accountNumber;
        this.owner = owner;
        this.branch = branch;
        this.balance = balance;
        this.interestRate = interestRate;
   }
    //Getters and setters omitted for brevity.
}
```
&emsp;&emsp;有了2.0版本的BankAccount银行账户类，我们可以给客户录入更多的信息：
```java
BankAccount account = new BankAccount(456L, "Marge", "Springfield", 100.00, 2.5);
BankAccount anotherAccount = new BankAccount(789L, "Homer", null, 2.5, 100.00);  //Oops!
```
&emsp;&emsp;这段代码编译器认为是对的，不会报错。但实际上Homer的账户金额每个月都会翻一翻！（如果有人知道哪个银行有此漏洞，请马山联系我$_$）发现了原因没？构造函数初始化参数传递顺序错误！

&emsp;&emsp;如果构造函数有多个连续的相同数据类型的参数，很容易就会搞错它们的次序。编译器又不会报错，出现Bug的时候会非常难找到问题原因。另外，越来越多的构造函数参数会让代码可读性越来越差。更糟糕的是，有时候有些参数是可选的，当我们不需要初始化这些参数的时候，还需要传递一个null。

&emsp;&emsp;你可能会想到让BankAccount提供一个空的构造函数，通过setter方法来设置其他账户信息。但这又会引发另一个问题，万一Bill忘记了调用某必要账户信息字段（比如accountNumber）的setter方法呢？此时Bill初始化的账户信息是不完整的，编译器也发现不了问题。

&emsp;&emsp;**这个时候，就需要Builder模式出场了。**

## Builder Pattern
&emsp;&emsp;Builder模式可以让我们写出可读性强、扩展性高的代码来初始化一个高度复杂的对象。

### 实现方式
&emsp;&emsp;在BankAccount类内部提供一个静态内部类Builder，Builder类包含所有BankAccount类所需要的字段，并提供public方法设置每一个字段，最后Builder类提供一个返回BankAccount对象的public方法；与此同时，我们可以去掉BankAccount类的复杂构造函数并提供一个private的空构造函数，如此一来，要创建一个银行账户就必须通过Builder类来实现。

### 实现代码
&emsp;&emsp;3.0版本的BankAccount类看上去应该是下面这样的：
```java
public class BankAccount {
    private long accountNumber;
    private String owner;
    private String branch;
    private double balance;
    private double interestRate;

    public static class Builder {
        private long accountNumber; //This is important, so we'll pass it to the constructor.
        private String owner;
        private String branch;
        private double balance;
        private double interestRate;
        public Builder(long accountNumber) {
            this.accountNumber = accountNumber;
        }
        public Builder withOwner(String owner){
            this.owner = owner;
            return this;  //By returning the builder each time, we can create a fluent interface.
        }
        public Builder atBranch(String branch){
            this.branch = branch;
            return this;
        }
        public Builder openingBalance(double balance){
            this.balance = balance;
            return this;
        }
        public Builder atRate(double interestRate){
            this.interestRate = interestRate;
            return this;
        }
        public BankAccount build(){
            //Here we create the actual bank account object, which is always in a fully initialised state when it's returned.
            BankAccount account = new BankAccount();  //Since the builder is in the BankAccount class, we can invoke its private constructor.
            account.accountNumber = this.accountNumber;
            account.owner = this.owner;
            account.branch = this.branch;
            account.balance = this.balance;
            account.interestRate = this.interestRate;
            return account;
        }
    }
    //Fields omitted for brevity.
    private BankAccount() {
        //Constructor is now private.
    }
    //Getters and setters omitted for brevity.
}
```
### 使用方式
&emsp;&emsp;创建银行账户的方式：
```java
BankAccount account = new BankAccount.Builder(1234L)
            .withOwner("Marge")
            .atBranch("Springfield")
            .openingBalance(100)
            .atRate(2.5)
            .build();
BankAccount anotherAccount = new BankAccount.Builder(4567L)
            .withOwner("Homer")
            .atBranch("")
            .openingBalance(100)
            .atRate(2.5)
            .build();
```

### 模式总结
&emsp;&emsp;使用Builder模式之后，代码结构更清晰，可读性更强，出错概率也会随着降低。

&emsp;&emsp;当你发现某个类需要增加过多的构造参数使得程序复杂度变高可读性变差的时候，也许你应该考虑使用Builder模式了。
