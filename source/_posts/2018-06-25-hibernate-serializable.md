---
layout: post
title:  "Hibernate save插入数据返回主键为0"
date:   2018-06-25 23:06:03
categories: 开发问题 
tags: Hibernate
toc: true
---

## Hibernate 利用模板的save方法插入实体，返回的Serializable id主键为0

在使用HibernateTemplate的save()方法后得不到持久化对象的id值，得到的持久化对象的id值一直为0。

因为表映射实体id 没有加自增，在实体类中添加自增注解 `@GeneratedValue(strategy=GenerationType.AUTO)` 即可

<!-- more -->

```java
@Id
@Column(name = "id", nullable = false)
@GeneratedValue(strategy=GenerationType.AUTO)
public long getId() {
    return id;
}
```

