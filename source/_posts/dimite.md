---
title: 迪米特法则|如何降低类间耦合
date: 2019-03-22 21:54:12
tags: Design
categories: 设计模式
---

![](http://posw9yxeh.bkt.clouddn.com/images/common/gratisography-271-thumbnail-small.jpg)
<!-- more -->


软件开发一直在推崇一个概念-低耦合、高内聚。 那什么样的代码设计才算得上低耦合、高内聚的代码。本文通过迪米特法则来讲解一下如何进行低耦合的代码设计。

> 迪米特法则也叫最小知识原则（Least Knowledge Principle）,即一个类应该对自己需要耦合和调用的类保持最少的认识。也就是一个类对自己依赖的类知道的越少越好。因而迪米特法则应该遵循一下的要义
1. 被依赖者，只应该暴露应该暴露的方法
2. 依赖者，只依赖应该依赖的对象

### 一个案例
David Bock根据迪米特法则给出了一个超市购物的案例。
三个关键信息：消费者、钱包、收银员
定义了三个类，分别是Customer、Wallet、PaperBoy
```
public class Customer {
    private String firstName;
    private String lastName;
    private Wallet myWallet;
    public String getFirstName(){
        return firstName;
    }
    public String getLastName(){
        return lastName;
    }
    public Wallet getWallet(){
        return myWallet;
    }
}
```

```
public class Wallet {
    private float value;
    public float getTotalMoney() {
        return value;
    }
    public void setTotalMoney(float newValue) {
        value = newValue;
    }
    public void addMoney(float deposit) {
        value += deposit;
    }
    public void subtractMoney(float debit) {
        value -= debit;
    }
}

```
```
public class Paperboy {
    public void charge(Customer myCustomer, double payment) {
        Wallet theWallet = myCustomer.getWallet();
        if (theWallet.getTotalMoney() > payment) {
            theWallet.subtractMoney(payment);
        } else {
            //money not enough
        }
    }
}
```
从这三个类可以看出， PaperBoy承担了大多数的功能实现。PaperBoy从消费者那里拿到了钱包，核点钱包的的金钱并自己从中拿去购物的费用。paperBoy既与Customer发生直接交互，又与Wallet发生间接交互，不符合最小知识原则（迪米特法则）。案例主要存在以下问题

+ Wallet暴露太多方法，其实Customer只要能够用钱包进行付钱就行了。所以这违反了迪米特法则的第一条（被依赖者，只暴露应该暴露的方法）
+ 让PaperBoy与Wallet直接交互是错误的行为，Wallet是Customer的私有财物，ParperBoy是无权过问Wallet的情况的， 所以从职责的角度上来看，这是不符合逻辑，违反了迪米特法则的第二条（依赖者，只依赖应该依赖的对象）

### 如何进行修改

+ PaperBoy不再与钱包发生直接关系，直接向customer要钱
+ 钱包只暴露付钱的方法给Customer。 方法暴露越多，后期需求变更的影响越大。

```
public class PaperBoy {
    private Customer customer;

    public PaperBoy(Customer customer){
        this.customer = customer;
    }

    public void charge(float payment){
        customer.pay(payment);
    }
}
```

```
public class Customer {
    private String firstName;
    private String lastName;
    private Wallet myWallet;

    public Customer(String firstName, String lastName, Wallet myWallet) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.myWallet = myWallet;
    }

    public String getFirstName(){
        return firstName;
    }
    public String getLastName(){
        return lastName;
    }

    public void pay(float payment){
        myWallet.pay(payment);
    }
}
```

```
public class Wallet {
    private float value;
    private float getTotalMoney() {
        return value;
    }
    public void setTotalMoney(float newValue) {
        value = newValue;
    }
    private void addMoney(float deposit) {
        value += deposit;
    }
    private void subtractMoney(float debit) {
        value -= debit;
    }

    public void pay(float payment){
        if(getTotalMoney()>payment){
            subtractMoney(payment);
        }else {

        }
    }
}
```

迪米特法则核心观念--- 类间解耦、弱耦合










