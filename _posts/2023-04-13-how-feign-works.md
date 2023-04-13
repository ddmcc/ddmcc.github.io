---
layout: post
title:  "open-feign的实现原理"
date:   2023-04-13 15:34:00
categories: "open-feign"
tags:  "open-feign"
author: ddmcc
---

* content
  {:toc}


在过去使用feign时，会将feign接口单独放在一个包，并在接口类上声明 `@FeignClient(name = 服务名)`， 在接口方法上声明 `@xxxMapping(接口路径)` ，实现者和使用者都会去引入这个包。对于 **使用者** 来说只需要注入接口类，然后调用对应的方法就能调用到
在另一个服务的实现者实现的逻辑

根据过去的经验我猜测，在调用侧上的原理和mybatis相似，扫描所有被 `@FeignClient` 注解声明的类，然后为其生成代理对象，当执行某个方法时，会被拦截转而请求方法上的链接，不同的是mybatis是找到这个方法对应的MappedStatement对象，然后执行sql
实现侧的话则和普通controller类似，将这些方法封装成一个个handle并注册到handleMapping中，等待被DispatcherServlet调用



