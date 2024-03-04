---
layout: post
title:  "feign基于ribbon实现负载均衡"
date:   2023-07-06 15:34:00
categories: Spring Cloud
tags:  ribbon
toc: true
---

## 前言

上篇分析了 `OpenFeign` 使用 `FeignClient` 注解来生成 [动态代理对象的流程](https://ddmcc.github.io/2023/04/13/2023-04-13-how-feign-works/) 

<!-- more -->