---
layout: post
title:  "ubantu中安装Java"
date:   2019-05-09 20:27:29
categories: ubantu
tags: ubantu Java
author: ddmcc
---

* content
{:toc}


## 准备工作
- 下载jdk包  jdk-8u152-linux-x64.tar.gz
- 将下载包放到ubantu上 （本文放在/opt/java中）
- 解压下载包 `tar -zxvf jdk-8u152-linux-x64.tar.gz`




## 配置环境变量

```java
root@ddmcc:/opt/java# vim /etc/profile
```


在文件后面追加：



```java
#set java environment
export JAVA_HOME=/opt/java/jdk1.8.0_152
export PATH=${JAVA_HOME}/bin:${PATH}
```


追加后：


```java
# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).

if [ "$PS1" ]; then
  if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "`id -u`" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi

#set java environment
export JAVA_HOME=/opt/java/jdk1.8.0_152
export PATH=${JAVA_HOME}/bin:${PATH}
```

## 使Java环境生效
执行命令 `source /etc/profile` ，后输入 `java -version`查看

如果出现下面的版本，说明安装成功
```java
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
```