---
title: 单一职责原则
date: 2019-03-25 23:25:15
tags: Design
categories: 设计模式
---

### 单一职责原则

应该有且只有一个原因引起累的变更。

### 一个例子去理解单一职责原则

```
public interface IPhone {
    //接通
    public void dial(String phoneNumber);
    //聊天
    public void chat(Object o);
    //挂断
    public void hangup();
}
```
定义了一个Iphone的接口，包含了电话的三个功能，接通、聊天、挂断。 试想一下这个接口符合单一职责原则吗？（一个类或者接口只有一个原因引起变化） 

很明显，IPhone包括了两个职责,应该设计成两个接口。

1. 信号的接通与果断
2. 通话（数据传输）

```
public interface IConnectionManager {
    void dial(String phoneNumber);

    void hangup();
}
```

```
public interface IDataTransfer {
    void transfer();
}
```

### 单一职责原则的好处
1. 降低类的复杂度，职责清晰、明确
2. 复杂度降低，可读性提高
3. 可维护性提高
4. 变更的风险降低

### 职责没有量化的标准
类的单一职责原则受非常多的网因素制约，从理论上是非常优秀，但从实际的角度上来讲，单一职责原却难以落地。类职责的划分没有量化的标准，因为职责和变化原因都是不可度量的，因项目而异，因环境而异。





