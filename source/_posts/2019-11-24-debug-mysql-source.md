---
layout: post
title:  "debug调试Mysql源码"
date:   2019-11-24 20:10:57
categories: mysql
tags:  mysql
toc: true
---


再上一篇的事务隔离级别中说了mysql是通过两个隐藏列来实现的，后来又看了一些相关文章，发现好像和《高性能Mysql》说的有点不一样。mysql的mvcc是通过Read View和
Undo log来实现的，read view来判断数据行是否可读，undo log用来找到最近的可见版本。为了搞清楚内部是如何实现的（虽然不一定看的懂），所以想debug看下read view是如何生成的。

<!-- more -->


## 需要下载的软件

- Cmake：3.4.0

- Bison：2.4.1

- Perl: 5.26.3

- Boost 1.59.1：下载后解压到任意目录

- VS2013

- mysql5.7.12源码


## 用cmake构建项目

打开cmake-gui，选择源码，指定构建目录等

---
![markdown](https://ddmcc-1255635056.file.myqcloud.com/8677b01b-31bf-4dd4-bbe8-cfa8c6569415.png)
---
好了之后点击 **Configure** ，会弹窗让选择VS版本，选择对应版本

---


![markdown](https://ddmcc-1255635056.file.myqcloud.com/9fa0967b-7944-4248-a8e2-68087d450364.png)
--- 
完成后会在输出框中看见 **Configure Done**，然后再点击 **Generate**，后会看见Generating Done

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/af9c8305-c85f-4b2a-a243-2b697957e9b1.png)

## 打开VS CODE生成方案

打开build目录，双击根目录的 **ALL_BUILD.vcxproj** ，就会在VS2013中打开，之后将/sql/mysqld.cc中的test_lc_time_sz()函数，将其中的DBUG_ASSERT(0)改为DBUG_ASSERT(1)后
按 **F7** 进行生成方案。


## 开始调试

在启动之前先准备 `my.ini` 配置文件，放在C:/Windows或构建目录下，后进入构建目录下的/sql/Debug


- 打开cmd，输入`mysqld --initialize`初始化数据库后

- 输入 `mysqld --skip-grant-tables` 

- 新打开一个对话框，输入`mysql;`，`use mysql;`

- UPDATE user SET authentication_string=PASSWORD("123456") WHERE User="root";

- flush privileges;

注：启动mysql调试也可以在VS2013中进行，但是会提示下载字符。。。


然后就可以到/client/Debug 打开cmd，输入 `mysql -u root -p` 输入密码123456连接


---

启动并连接成功后，打开VS2013点击`调试`，`附加到进程`，找到mysql进程附加进去。

![markdown](https://ddmcc-1255635056.file.myqcloud.com/1d9a9557-2fad-4fe2-af2f-ccc460522773.png)

---
然后就可以debug了！



![markdown](https://ddmcc-1255635056.file.myqcloud.com/a259f14d-7bcc-4482-971a-5924a71c5812.png)