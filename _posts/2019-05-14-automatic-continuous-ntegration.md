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
 - 提交代码到github触发webhooks通知镜像容器服务
 - 镜像容器服务拉取代码根据写好的Dockerfile构建镜像
 - 镜像构建完成触发通知jenkins,jenkins插件Generic Webhook Trigger收到通知
 - 拉取镜像部署
 



## 准备工作

### github仓库
- 新建代码仓库
![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32aijkcn6j30zf09574w.jpg)


### 安装jenkins
- [安装jenkins](https://ddmcc.space/2019/05/15/installing-jenkins-in-ubantu/)
- 系统管理->插件管理->安装 **Generic Webhook Trigger** 插件
- 新建任务,直接到构建触发器如果安装了 **Generic Webhook Trigger** 插件就可以看到选项,勾选它。
填写token,此处填写的token为镜像服务配置链接中 `invoke?itoken={}` 括号的值。


![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32amsd7u5j30wu0o5go2.jpg)



新增header token参数


![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32c1q3xl3j30zp0i1407.jpg)



构建中添加构建步骤,选执行shell。拉取镜像时,如果新建的镜像时puclic则不需要登录,private则需要先登录.写好后应用,保存。



![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32b6r80rzj30ww0fejrz.jpg)


- 生成API token,点击用户名->设置->API token->填写生成.此处填写的token为镜像服务配置链接中 `http://账户名:{}@jenkins地址` 括号的值


![](http://ww1.sinaimg.cn/large/0060GLrDgy1g32awrjvt9j30wc06tt8x.jpg)



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

## 工作

git提交代码或直接github上修改代码触发。

