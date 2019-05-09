---
layout: post
title:  "ubantu中安装Maven"
date:   2019-05-09 20:27:29
categories: ubantu
tags: ubantu maven
author: ddmcc
---

* content
{:toc}


## 准备工作
- 下载maven包  apache-maven-3.5.3-bin.tar.gz
- 将下载包放到ubantu上 （本文放在/opt/maven中）
- 解压下载包 `tar -zxvf apache-maven-3.5.3-bin.tar.gz`




## 配置环境变量

```java
root@ddmcc:/opt/maven# ll
total 8608
drwxrwxrwx 3 root root    4096 May  9 21:17 ./
drwxr-xr-x 5 root root    4096 May  9 21:16 ../
drwxr-xr-x 6 root root    4096 May  9 21:17 apache-maven-3.5.3/
-rw-r--r-- 1 root root 8799579 Apr  8 20:51 apache-maven-3.5.3-bin.tar.gz
root@ddmcc:/opt/maven# rm -fr apache-maven-3.5.3-bin.tar.gz 
root@ddmcc:/opt/maven# ll
total 12
drwxrwxrwx 3 root root 4096 May  9 21:18 ./
drwxr-xr-x 5 root root 4096 May  9 21:16 ../
drwxr-xr-x 6 root root 4096 May  9 21:17 apache-maven-3.5.3/
root@ddmcc:/opt/maven# vim /etc/profile
```


在文件后面追加：



```java
#set maven environment
export MAVEN_HOME=/opt/maven/apache-maven-3.5.3
export PATH=${MAVEN_HOME}/bin:$PATH
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

#set maven environment
export MAVEN_HOME=/opt/maven/apache-maven-3.5.3
export PATH=${MAVEN_HOME}/bin:$PATH
```

## 使环境生效
执行命令 `source /etc/profile` ，后输入 `mvn -v`查看

如果出现下面的版本，说明安装成功
```java
Apache Maven 3.5.3 (3383c37e1f9e9b3bc3df5050c29c8aff9f295297; 2018-02-25T03:49:05+08:00)
Maven home: /opt/maven/apache-maven-3.5.3
Java version: 1.8.0_152, vendor: Oracle Corporation
Java home: /opt/java/jdk1.8.0_152/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "4.4.0-21-generic", arch: "amd64", family: "unix"
```