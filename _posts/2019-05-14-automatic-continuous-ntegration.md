---
layout: post
title:  "github+阿里云容器镜像服务+jenkins自动化持续部署"
date:   2019-05-15 20:27:29
categories: 持续部署
tags: jenkins github 阿里云容器镜像服务
author: ddmcc
---

* content
{:toc}


## 工作流程
  提交代码到github触发webhooks通知镜像容器服务->镜像容器服务拉取代码根据写好的Dockerfile构建镜像->
镜像构建完成触发通知jenkins,jenkins插件Generic Webhook Trigger收到通知->拉取镜像部署
 




## 准备工作

### github仓库

### 安装jenkins
- [安装jenkins](https://ddmcc.space/2019/05/15/installing-jenkins-in-ubantu/)
- jenkins安装 **Generic Webhook Trigger** 插件


### 容器镜像服务
- [容器镜像服务](https://cr.console.aliyun.com)上新建镜像仓库
- 绑定github账号,代码源设置github仓库,勾选`代码变更自动构建镜像`
- 新建构建规则,选定版本变更构建或分支代码构建,设置Dockerfile路径,版本


#### 新建仓库
![](http://ww1.sinaimg.cn/large/0060GLrDgy1g329s4tva6j30qv0nhwfg.jpg)
![](http://ww1.sinaimg.cn/large/0060GLrDgy1g329tiatufj30r10fwq3x.jpg)

#### 新建构建规则
![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32a2nr20bj30ko0g9aal.jpg)

#### 新建触发器
![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32a6hsk86j30jt0dimxo.jpg)


格式为: **http://账户名:加密API TOKEN@jenkins地址/generic-webhook-trigger/invoke?token=jenkins任务配置的token**


### 

