---
layout: post
title:  "maven添加oracle jar包"
date:   2018-04-05 21:08:15
categories: maven
tags: maven
toc: true
---

## 问题

在 Maven 中央仓库上的几个ojdbc包依赖配置,都没法通过Maven自动下载到本地目录,tomcat服务器启动错误

<!-- more -->

## 解决方案

### 1,选择导入jar

在空白处点击右键 -> Import -> Maven -> Install or deploy an artifact to a Maven repository，然后点击"Next"按钮




![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq268opay7j30eb0f7gme.jpg)


### 2,进入导入界面

> * Artifact file：本地的jar路径
> * Group Id,Artifact Id,Version: pom.xml导入的名字与版本

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq26eg6mgbj30ed0fa3z2.jpg)

### 3,导入后即可在本地仓库看到ojdbc目录

配置pom.xml

```
<dependency>
    <groupId>ojdbc</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.4</version>
</dependency>
```

