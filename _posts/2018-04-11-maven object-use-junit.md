---
layout: post
title:  "maven项目中使用JUnit进行单元测试"
categories: JUnit
date: 2018-04-11 23:27:42
tags:  maven JUnit 测试
author: ddmcc
---

* content
{:toc}

发现通过Spring进行对象管理之后，做测试变得复杂了。因为所有的Bean都需要在 applicationContext.xml中加载好，之后再通过@Resource去取得。如果每次都要整个业务流做的差不多了再去测试，这样效率很 低，也很麻烦。
这时候就需要Spring-text框架整合JUnit进行测试.




## 步骤

### 配置pom.xml

添加spring-text jar包

```js
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-test</artifactId>
	<version>4.1.6.RELEASE</version>
</dependency>
```

添加JUnit包

```js
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
	<scope>test</scope>
</dependency>
```

### 编写测试类

在src/test/java 中编写一个测试类

```js
@RunWith(SpringJUnit4ClassRunner.class)  
@ContextConfiguration(locations = "classpath:applicationContext.xml")  
public class EmpTest {

	@Resource
	private EmpService empService;
	
	@Test
	public void testFindAll(){
		List<Emp> list = empService.findAll(new Emp());
		for(Emp emp : list){
			System.out.println(emp.getEname()+","+emp.getEtags());
		}
	}
}
```

在方法中添加 `@Test` 注解，在类名上添加 `@RunWith(SpringJUnit4ClassRunner.class)`  
`@ContextConfiguration(locations = "classpath:applicationContext.xml")`

### 调用

声明业务层接口，由Spring注入

```js
@Resource
private EmpService empService;
```

***
## 总结

### 结构

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fq95hqo1tsj307c0crjrn.jpg)

### 代码

```java
package com.test;

import java.util.List;
import javax.annotation.Resource;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import com.soft.bean.Emp;
import com.soft.service.EmpService;

@RunWith(SpringJUnit4ClassRunner.class)  
@ContextConfiguration(locations = "classpath:applicationContext.xml")  
public class EmpTest {

	@Resource
	private EmpService empService;
	
	@Test
	public void testFindAll(){
		List<Emp> list = empService.findAll(new Emp());
		for(Emp emp : list){
			System.out.println(emp.getEname()+","+emp.getEtags());
		}
	}
}
```



