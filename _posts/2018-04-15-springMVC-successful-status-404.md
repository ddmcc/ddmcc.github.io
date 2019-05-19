---
layout: post
title:  "SpringMVC后台执行正常,但是前端显示404"
date:   2018-04-15 23:54:40
categories: springMVC
tags: springMVC
author: ddmcc
---

* content
{:toc}




## 问题

今天在做springMVC查询数据时,前端通过ajax发送请求，后台数据查询正常,且无异常提醒，但前端却提示404，数据也没有传过来

## 解决方案

检查发现原来是后端 `controller` 层处理方法没有返回状态码，在方法中添加 `@ResponseBody` 注解后，问题解决

```java
@RequestMapping(value = "/find.action" , method = RequestMethod.POST ,produces = "application/json;charset=utf-8" )
public @ResponseBody Object findStore() {
	List<Store> store = storeService.find();
	return JSONObject.toJSON(store);
}
```

