---
layout: post
title:  "linux 7.3中安装Java"
date:   2019-05-29 23:13:06
categories: linux
tags: linux Java
author: ddmcc
---

* content
{:toc}




## 准备工作
- 下载jdk包  jdk-8u152-linux-x64.tar.gz
- 将下载包放到服务器上 （本文放在/usr/java中）
- 解压下载包 `tar -zxvf jdk-8u152-linux-x64.tar.gz`


## 配置环境变量

```java
[root@iZuf6fc5fgstkxrkw804kxZ java]# vim /etc/profile
```


在文件后面追加：



```java
#set java environment
export JAVA_HOME=/usr/java/jdk1.8.0_152
export PATH=${JAVA_HOME}/bin:${PATH}
```

## 使Java环境生效
执行命令 `source /etc/profile` ，后输入 `java -version`查看

```java
[root@iZuf6fc5fgstkxrkw804kxZ java]# source /etc/profile
[root@iZuf6fc5fgstkxrkw804kxZ java]# java -version
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
[root@iZuf6fc5fgstkxrkw804kxZ java]# 
```

如果出现下面的版本，说明安装成功
```java
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
```