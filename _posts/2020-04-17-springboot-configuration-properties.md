---
layout: post
title:  "SpringBoot参数配置的坑"
date:   2020-04-17 23:37:36
categories: 坑
tags:  ConfigurationProperties
author: ddmcc
---

* content
{:toc}


### 问题

在yml中配置参数，用 ` @ConfigurationProperties` 注解来注入，发现配置以0开头的字符串得到的结果是错的，比如配置的
**01010807**，实际的值变为 **1010807.0** 





---
![915FA91686ED76C3005305808E4A1D82.png](https://i.loli.net/2020/04/17/BftEkDcwOrdv3A9.png)

---

### 解决

配置的值用加引号 `""`

```yml
the-one:
  todo:
    region: "01010807"
```


