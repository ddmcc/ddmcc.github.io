---
layout: post
title:  "如何用自己的域名来访问博客"
categories: other
date: 2018-04-06 15:57:27
tags:  blog domain
author: ddmcc
---

* content
{:toc}

## 准备

首先需要一个域名,可以在各个服务商购买

  * [腾讯](https://dnspod.cloud.tencent.com/act/yearendsales?from=dnspodqcloud)
  * [阿里](https://wanwang.aliyun.com/?utm_content=se_1010760)
  * [GoDaddy](https://sg.godaddy.com/zh/domains)




我是从[GoDaddy](https://sg.godaddy.com/zh/domains) 买的,不要 <del>998</del> 一年只要7块钱，`不是打广告！`


## 开始

**1. 向你的 Github Pages 仓库添加一个CNAME文件**

在首行填写域名,其中只能包含一个顶级域名

- 注意：只要填写如xxxx.com，注意前面没有http://，也没有www


**2. 在域名提供商中添加域名解析**

- (1)先添加一个CNAME，主机记录写@，后面记录值写上你的xxxx.github.io
- (2)先添加一个CNAME，主机记录写www，后面记录值写上你的xxxx.github.io

这样别人用www和不用www都能访问你的网站

我是在goDaddy买的，在 [腾讯云](https://console.qcloud.com/domain) 解析的，
所以先要在goDaddy上添加DNS服务器

在腾讯控制台找到DNS服务器

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq30hr5htuj30fe02sdfz.jpg)

doGoDaddy处添加DNS服务器

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq30gyext9j30ta07fq32.jpg)

然后在添加解析

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq30knvgcjj30xb07jaan.jpg)


## 访问

等几分钟后就可以访问了，注意 **要等！要等！！**
