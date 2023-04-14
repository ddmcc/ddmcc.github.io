---
layout: post
title:  "设计模式之组合模式"
date:   2023-04-01 15:34:00
categories: 设计模式
tags:  设计模式 装饰者模式
toc: true
---


## 结构

![markdown](https://ddmcc-1255635056.file.myqcloud.com/0cc84eba-62ec-4fcd-aa5e-1adfb90399a4.png)

>Leaf: 叶子节点
>Composite：组合对象

`Leaf` 叶子节点和 `Composite` 组合对象实现共同接口，不同的是子节点在方法中编写具体的处理逻辑，组合对象的作用是循环所有子节点的处理方法。如上图，`Composite` 维护一个子对象列表 List<Component> ，
当使用者向Composite对象（类型）发送请求时，该请求被转发到所有子Component对象（Leaf和Composite）

<!-- more -->

>为什么说该请求被转发到所有子Leaf和Composite？因为组合对象维护的是List<Component>，是顶层Component，所以可以是一个组合对象Composite


组合对象Composite中，通常还会有提供对子列表操作的方法，用于在运行时动态添加或者结合Spring的自动注入子组件，这样的好处就是外部的子组件也可以被添加进来并被执行

