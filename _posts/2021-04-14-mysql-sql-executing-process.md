---

layout: post
title:  "mysql执行过程"
date:   2021-04-14 16:56:23
categories: mysql
tags:  mysql
author: ddmcc
---

* content
{:toc}




## 一条SQL查询语句是如何执行的



下面是 `mysql` 查询的基本执行路径示意图：

![markdown](https://ddmcc-1255635056.file.myqcloud.com/40fd5017-8a29-4da8-bfe7-d3bb19ec0599.png)

大体来说，**mysql可以分为 Server 层和存储引擎层两部分**

- **Server** 层包括：连接器、查询缓存、语法解析器、优化器、执行器等大部分核心服务功能，以及所有的内置函数（如日期、时间、数学加密函数等）。所有跨存储引擎的功能都在这一层实现：如存储过程、触发器、视图等
- **存储引擎** 层负责数据的存储和提取。其架构是插件式的，支持 `InnoDB` , `MyISAM` , `Memory` 等多个存储引擎。




