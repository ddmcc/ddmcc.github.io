---
layout: post
title:  "SpringBoot参数配置的坑"
date:   2020-04-17 23:37:36
categories: 开发问题
tags:  开发问题
toc: true
---


### 问题

在yml中配置参数，用 ` @ConfigurationProperties` 注解来注入，发现配置以0开头的字符串得到的结果是错的，比如配置的
**01010807**，实际的值变为 **1010807.0** 


<!-- more -->


---
![markdown](https://ddmcc-1255635056.file.myqcloud.com/abeef1ec-0534-44f0-bdf1-1dfedf42c081.png)

---

### 解决

配置的值用加引号 `""`即可，原因是值被当成了数字类型

```yml
the-one:
  todo:
    region: "01010807"
```


<https://github.com/spring-projects/spring-boot/issues/9389>
