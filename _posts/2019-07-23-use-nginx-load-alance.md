---
layout: post
title:  "nginx负载均衡的简单使用"
date:   2019-07-23 22:09:38
categories: nginx
tags: nginx 负载均衡 tomcat
author: ddmcc
---

* content
{:toc}


## 启动两个Tomcat

**1,编写`docker-compose.yml`**





	version: '3'
	services:
	  tomcat1:
	    image: tomcat
	    container_name: tomcat1
	    ports:
	     - 9090:8080

	  tomcat2:
	    image: tomcat
	    container_name: tomcat2
	    ports:
	      - 9091:8080





**2,`docker-compose up -d` 启动tomcat**

	root@ddmcc:/usr/local/docker/tomcat# docker-compose up -d
	Creating network "tomcat_default" with the default driver
	Creating tomcat1 ... 
	Creating tomcat2 ... 
	Creating tomcat1
	Creating tomcat1 ... done





**3,输入`docker ps`查看,tomcat已正常启动**
![](https://i.loli.net/2019/07/23/5d37182cd7a4086825.png)




**4,在两个tomcat的index页面追加文字以便待会区分。**
输入`docker exec -it tomcat1 /bin/bash`已交互的方式进入容器

进入webapps/ROOT目录在追加端口到末尾

	root@ddmcc:/usr/local/docker/tomcat# docker exec -it tomcat1 /bin/bash
	root@b91b07cde612:/usr/local/tomcat# 
	root@b91b07cde612:/usr/local/tomcat# 
	root@b91b07cde612:/usr/local/tomcat# cd webapps/ROOT/
	root@b91b07cde612:/usr/local/tomcat/webapps/ROOT# echo "9090" >> index.jsp



输入`192.168.132.129:9090` 查看端口是否已追加
![](https://i.loli.net/2019/08/03/W4cC5RfrGwIOd8o.png)

tomcat2重复tomcat1的操作,追加端口9091


## 配置nginx

1,编写`docker-compose.yml`

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





2,创建文件夹/conf,并在其下面创建配置文件nginx.conf

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
	    upstream app {
		server 192.168.132.129:9090 weight=10;
		server 192.168.132.129:9091 weight=10;
	    }

	    server {
		listen       80;
		server_name  192.168.132.129;
		location / {
		    proxy_pass http://app;
		    index  index.html index.htm;
		}

	    }
	}





3,`docker-compose up -d` 启动nginx
![](https://i.loli.net/2019/07/23/5d37182cd7a4086825.png)





4,输入`192.168.132.129:81`可以看到反向代理到了9090和9091,并且实现了负载均衡

![](https://i.loli.net/2019/07/23/5d371f7e6869c97164.png)

![](https://i.loli.net/2019/07/23/5d371f6eed21d12084.png)