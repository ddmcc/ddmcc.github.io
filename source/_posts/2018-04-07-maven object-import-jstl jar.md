---
layout: post
title:  "maven项目导入jstl包,tomcat启动找不到jar文件,页面无法使用jstl标签"
categories: maven
date: 2018-04-07 01:52:42
tags:  maven jstl tomcat
---

在pom.xml配置文件中，已经添加jstl包，服务器启动报异常，jstl jar包不能初始化，成功添加jar包后页面jstl不解析，导致无法取数据

<!-- more -->

## 问题

在pom.xml配置文件中，已经添加jstl包，服务器启动报异常，jstl jar包不能初始化，成功添加jar包后页面jstl不解析，导致无法取数据

```java
org.apache.jasper.JasperException: The absolute uri: [http://java.sun.com/jsp/jstl/core] cannot be resolved in either web.xml or the jar files deployed with this application
```




## 解决

网上查发现是项目中并没有导入jar包，一脸懵逼？？？pom.xml不是导入了吗

没导入就没导入吧，那我手动导入行不行？？

把jar包手动拷贝到 `/WEB-INF` 目录下

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq3gh18e2qj308v0b5wer.jpg)

有报错，虽然不知道什么原因，但是对于一个强迫症患者来说这怎么行？？

于是乎直接把jar包扔到了 `tomcat` 下的 `lib` 目录下了，哈哈哈这下总可以了吧

启动

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq3gldrvkpj30vz0hjjrs.jpg)

完美！看见登录页面了！

下面登录就可以登录了，走你

ε=(´ο｀*)))唉？页面好像不太对，数据呢？

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq3go6ldl9j30n50dt74d.jpg)

我的jstl表达式怎么出来了啦？Σ(⊙▽⊙"a表达式好像没有解析 `${user.nickname },${user.gender eq 0? '男':'女'} ${user.province },${user.city }`

这样还不行？继续查之~~~~

终于在页面添加

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq3grklz4mj30s9031dg0.jpg)

**`<%@ page isELIgnored="false"%>`** 问题解决！！

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq3h0apdr3j31cc0n017r.jpg)


## 总结

* **(1)，pom.xml配置jstl jar包**

```java
<!-- https://mvnrepository.com/artifact/javax.servlet/jstl -->
	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>jstl</artifactId>
		<version>1.2</version>
	</dependency>
```

* **(2)，tomcat的lib目中添加jstl jar包**

* **(3)，jsp页面添加 `<%@ page isELIgnored="false"%>`**


