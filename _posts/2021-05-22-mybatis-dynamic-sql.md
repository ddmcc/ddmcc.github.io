---

layout: post
title:  "mybatis动态sql的问题"
date:   2021-05-22 16:56:23
categories: 开发问题
tags:  开发问题 mybatis
author: ddmcc
---

* content
{:toc}





## 遇到的问题

最近组内有个小伙伴求助，说在 `xml` 中写 `mybatis` 动态sql，明明 `if` 条件是成立的，但实际执行的 sql 语句并没有拼接上，而是走了另外的判断分支



## 问题复现

**xml 伪代码：**

```xml
    <select id="select" resultType="com.ddmcc.mybatis.bindings.entity.Book">
        SELECT *
        FROM book
        WHERE 1 = 1
        <if test="book.type = 1">
            AND book_name = '书籍1'
        </if>
        <if test="book.type == 2">
            AND book_name = '书籍2'
        </if>
    </select>
```

如果 `type = 1` ，那么拼接条件 `AND book_name = '书籍1'` ，如果 `type = 2` ，那么拼接条件 `AND book_name = '书籍2'` 


**查询伪代码：**

```java
    @Test
    void select() {
        Book book = new Book();
        book.setType(2);
        System.out.println(bookMapper.select(book));
        System.out.println(book);
    }
```

---

按照想的应该要拼接上 ` AND book_name = '书籍2' ` 的查询条件，因为满足 `type = 2`。然而并没有！

---
![markdown](https://ddmcc-1255635056.file.myqcloud.com/4804949d-fc3b-47a1-a5a1-fc5e1317e819.png)

---

并且在经过查询后，参数对象 `book` 的type字段值变成了1 ！

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/15ea80c7-b3a6-4ae5-a141-431c7cdd2a4f.png)

---


## 发现问题

如果认真一点看就能够发现，在第一个 `if` 标签中，写的是 `=` 号，第二个中写的是 `==` 。所以到这里大概也就能猜到， **是因为第一个 `if` 给type重新赋值变成了1，且
赋值操作返回结果为 true ，导致进了第一个if判断，而第二个if因为type = 2，所以不满足**


## mybatis中动态sql的实现

- 在初始化阶段，mybatis会解析xml文件中的sql，并解析成对象（SqlSource），比如动态sql会被解析成 `DynamicSqlSource`

- 然后在 `XMLScriptBuilder` 中会根据动态标签解析成相应的节点对象，比如 `if` 标签 对应 `IfSqlNode` 节点对象

所有节点对象在 `org.apache.ibatis.scripting.xmltags` 包目录下

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/ee386f3d-5a75-4997-9a36-4642d6cb16d3.png)

---

**如本例子的sql 被解析成5个node**

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/c2334433-a75f-4f58-aece-0b7222ab8094.png)

---

- 调用各自 SqlNode 对象的 apply方法去执行相应判断，拼接sql，如：`StaticTextSqlNode` 静态sql则可以直接拼接

---

```java
  // StaticTextSqlNode ====> apple直接把sql拼接到context
  @Override
  public boolean apply(DynamicContext context) {
    context.appendSql(text);
    return true;
  }
```

---
如果是 `IfSqlNode` 则会用 `Ognl` 执行表达式来获取结果，根据结果判断拼接sql

---

`evaluateBoolean` 方法结果为真，拼接字符串

```java
  // IfSqlNode#33 ====> 执行表达式
  @Override
  public boolean apply(DynamicContext context) {
    if (evaluator.evaluateBoolean(test, context.getBindings())) {
      contents.apply(context);
      return true;
    }
    return false;
  }
```

----


![markdown](https://ddmcc-1255635056.file.myqcloud.com/1a352479-835c-40d5-8542-e3783929ed6e.png)

----

```java
  // OgnlCache#41  ====> 通过Ognl传入参数来执行表达式
  public static Object getValue(String expression, Object root) {
    try {
      Map<Object, OgnlClassResolver> context = Ognl.createDefaultContext(root, new OgnlClassResolver());
      return Ognl.getValue(parseExpression(expression), context, root);
    } catch (OgnlException e) {
      throw new BuilderException("Error evaluating expression '" + expression + "'. Cause: " + e, e);
    }
  }

  // OgnlCache#50 ====> 解析表达式，缓存
  private static Object parseExpression(String expression) throws OgnlException {
    Object node = expressionCache.get(expression);
    if (node == null) {
      node = Ognl.parseExpression(expression);
      expressionCache.put(expression, node);
    }
    return node;
  }
```


## 总结

到这里基本也就清楚了，**mybatis 其实是用 `Ognl` 来对表达式进行判断，本例第一个 `ifSqlNode` 对变量进行赋值操作并返回了结果为1，判断1为真，执行 sql 拼接**




