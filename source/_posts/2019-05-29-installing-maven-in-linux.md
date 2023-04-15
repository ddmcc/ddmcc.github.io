---
layout: post
title:  "阿里云服务器中安装Maven"
date:   2019-05-29 23:24:09
categories: linux
tags: linux
toc: true
---


## 准备工作
- 下载maven包  apache-maven-3.5.3-bin.tar.gz
- 将下载包放到ubantu上 （本文放在/usr/maven中）
- 解压下载包 `tar -zxvf apache-maven-3.5.3-bin.tar.gz`

<!-- more -->

## 配置环境变量

```java
[root@iZuf6fc5fgstkxrkw804kxZ maven]# ll
total 8600
drwxr-xr-x 6 root root    4096 May 29 23:26 apache-maven-3.5.3
-rw-r--r-- 1 root root 8799579 Apr  8 20:51 apache-maven-3.5.3-bin.tar.gz
[root@iZuf6fc5fgstkxrkw804kxZ maven]# vim /etc/profile
```


在文件后面追加：



```java
#set maven environment
export MAVEN_HOME=/opt/maven/apache-maven-3.5.3
export PATH=${MAVEN_HOME}/bin:$PATH
```


追加后：


```java
.
.
.
.

#set java environment
export JAVA_HOME=/usr/java/jdk1.8.0_152
export PATH=${JAVA_HOME}/bin:${PATH}

#set maven environment
export MAVEN_HOME=/usr/maven/apache-maven-3.5.3
export PATH=${MAVEN_HOME}/bin:$PATH
```

## 使环境生效
执行命令 `source /etc/profile` ，后输入 `mvn -v`查看

如果出现下面的版本，说明安装成功
```java
[root@iZuf6fc5fgstkxrkw804kxZ maven]# source /etc/profile
[root@iZuf6fc5fgstkxrkw804kxZ maven]# mvn -v
Apache Maven 3.5.3 (3383c37e1f9e9b3bc3df5050c29c8aff9f295297; 2018-02-25T03:49:05+08:00)
Maven home: /usr/maven/apache-maven-3.5.3
Java version: 1.8.0_152, vendor: Oracle Corporation
Java home: /usr/java/jdk1.8.0_152/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-514.26.2.el7.x86_64", arch: "amd64", family: "unix"
[root@iZuf6fc5fgstkxrkw804kxZ maven]# 
```