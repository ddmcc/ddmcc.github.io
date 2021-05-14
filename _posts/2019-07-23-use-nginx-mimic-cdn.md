---
layout: post
title:  "nginx搭建伪CDN服务器"
date:   2019-07-23 22:09:38
categories: nginx
tags: nginx
author: ddmcc
---

* content
{:toc}


**1,编写`docker-compose.yml`**





	version: '3.1'
	services:
	  nginx:
	    restart: always
	    image: nginx
	    container_name: nginx
	    ports:
	      - 81:80
	    volumes:
	      - ./conf/nginx.conf:/etc/nginx/nginx.conf
	      - ./wwwroot:/usr/share/nginx/wwwroot





**2,编写nginx配置,`nginx.conf`**

	user nginx;

	worker_processes  1;

	events {
	    worker_connections  1024;
	}

	http {
	    include       mime.types;
	    default_type  application/octet-stream;
	    sendfile        on;

	    keepalive_timeout  65;

	    server {
		listen       80;
		server_name  192.168.132.129;
		location / {
		    root /usr/share/nginx/wwwroot/cdn;
		    index  index.html index.htm;
		}

	    }
	}





**3,在nginx目录下创建`/wwwroot/cdn`目录,再在目录下创建`/2019/07/23`目录,将文件**

**`2019-07-23-use-nginx-mimic-cdn.md`上传到该目录**
![markdown](https://ddmcc-1255635056.file.myqcloud.com/e15bddfd-3449-472f-883b-810e99852f79.png)



**4,输入`192.168.132.129:81/2019/07/23/2019-07-23-use-nginx-mimic-cdn.md`可以下载访问该文件了**
