---
layout: post
title:  "openFeign整合ribbon实现负载均衡"
date:   2023-05-29 15:34:00
categories: Spring Cloud
tags:  ribbon OpenFeign
toc: true
---

通过 [上篇文章](https://ddmcc.github.io/2023/04/13/2023-04-13-how-feign-works/) 了解了 `OpenFeign` 生成动态代理的整个过程，并且知道了它是基于 `JDK` 动态代理来实现的。这篇文章主要来看看 `OpenFeign` 是如何基于 `ribbon` 实现负载均衡的，还会了解到 `ribbon` 
的一些主要对象及运行原理，最后还有一个自定义 `ribbon` 负载均衡规则的代码案例

<!-- more -->

### 回顾



















`feign` 客户端接口的调用是基于 `JDK` 动态代理来实现的，在所有的方法调用的时候最终都会走 `InvocationHandler` 接口，默认实现类是 `ReflectiveFeign.FeignInvocationHandler`，下面就跟着代码执行流程，看看如何实现rpc调用及整合ribbon负载均衡
