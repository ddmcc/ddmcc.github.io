---
layout: post
title:  "github+阿里云容器镜像服务+jenkins自动化持续部署"
date:   2019-05-15 20:27:29
categories: 自动化部署
tags: jenkins
toc: true
---

## 工作流程
 - 提交代码到github触发webhooks通知镜像容器服务
 - 镜像容器服务拉取代码根据写好的Dockerfile构建镜像
 - 镜像构建完成触发通知jenkins,jenkins插件Generic Webhook Trigger收到通知
 - 拉取镜像部署

<!-- more -->

## 准备工作

### github仓库

- 新建代码仓库

![markdown](https://ddmcc-1255635056.file.myqcloud.com/67496643-1e74-4c05-a8b9-aa23bfe1d140.png)


### 安装jenkins
- [安装jenkins](https://ddmcc.cn/2019/05/15/installing-jenkins-in-ubuntu/)
- 系统管理->插件管理->安装 **Generic Webhook Trigger** 插件
- 新建任务,直接到构建触发器如果安装了 **Generic Webhook Trigger** 插件就可以看到选项,勾选它。
填写token,此处填写的token为镜像服务配置链接中 `invoke?itoken={}` 括号的值。


![markdown](https://ddmcc-1255635056.file.myqcloud.com/0d3b988f-8681-4831-88fd-ac816a7950ba.png)



新增header token参数


![markdown](https://ddmcc-1255635056.file.myqcloud.com/a54f2d89-0b38-46c1-9848-df89110eb638.png)



构建中添加构建步骤,选执行shell。拉取镜像时,如果新建的镜像时puclic则不需要登录,private则需要先登录.写好后应用,保存。



![markdown](https://ddmcc-1255635056.file.myqcloud.com/2dd0d4e7-e398-4c60-98e3-956ebd7d7182.png)


- 生成API token,点击用户名->设置->API token->填写生成.此处填写的token为镜像服务配置链接中 `http://账户名:{}@jenkins地址` 括号的值


![markdown](https://ddmcc-1255635056.file.myqcloud.com/4fc3a556-70c5-4681-968d-9163c1ebda87.png)



### 容器镜像服务
- [容器镜像服务](https://cr.console.aliyun.com)上新建镜像仓库
- 绑定github账号,代码源设置github仓库,勾选`代码变更自动构建镜像`
- 新建构建规则,选定版本变更构建或分支代码构建,设置Dockerfile路径,版本


#### 新建仓库
![markdown](https://ddmcc-1255635056.file.myqcloud.com/1f945bdc-32c6-457b-8e13-286d386b1b9e.png)
![markdown](https://ddmcc-1255635056.file.myqcloud.com/3506699a-796e-4786-9744-a154521da685.png)

#### 新建构建规则
![markdown](https://ddmcc-1255635056.file.myqcloud.com/95d0e38c-a042-4cef-97b3-0c907b4f3191.png)

#### 新建触发器
![markdown](https://ddmcc-1255635056.file.myqcloud.com/fce17da7-6037-4977-9beb-93a03891d21a.png)


格式为: **http://账户名:加密API TOKEN@jenkins地址/generic-webhook-trigger/invoke?token=jenkins任务配置的token**

## 工作

git提交代码或直接github上修改代码触发。

