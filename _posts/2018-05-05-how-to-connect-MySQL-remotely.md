---
layout: post
title:  "远程连接MySql"
date:   2018-05-05 14:37:53
categories: MySql
tags: MySql
author: ddmcc
---

* content
{:toc}




## 远程访数据库

```js
- 1、mysql -u root -p

- 2、use mysql;

- 3、update user set host='%'  where user='用户名';  

- 4、flush privileges;
```

这样就可以远程访问数据库了


## 更改密码验证

在mysql 5.6对密码的强度进行了加强，推出了validate_password 插件。

### 查看密码规则参数
```java
mysql> select @@validate_password_policy;  
+----------------------------+  
| @@validate_password_policy |  
+----------------------------+  
| MEDIUM                     |  
+----------------------------+  
1 row in set (0.00 sec)


mysql> SHOW VARIABLES LIKE 'validate_password%';  
+--------------------------------------+--------+  
| Variable_name                        | Value  |  
+--------------------------------------+--------+  
| validate_password_dictionary_file    |        |  
| validate_password_length             | 8      |  
| validate_password_mixed_case_count   | 1      |  
| validate_password_number_count       | 1      |  
| validate_password_policy             | MEDIUM |  
| validate_password_special_char_count | 1      |  
+--------------------------------------+--------+  
6 rows in set (0.08 sec)
```

### 参数解释

```js
validate_password_dictionary_file
插件用于验证密码强度的字典文件路径。

validate_password_length
密码最小长度，参数默认为8，它有最小值的限制，最小值为：validate_password_number_count + validate_password_special_char_count + (2 * validate_password_mixed_case_count)

validate_password_mixed_case_count
密码至少要包含的小写字母个数和大写字母个数。

validate_password_number_count
密码至少要包含的数字个数。

validate_password_policy
密码强度检查等级，0/LOW、1/MEDIUM、2/STRONG。有以下取值：
Policy                 Tests Performed                                                                                                        
0 or LOW               Length                                                                                                                      
1 or MEDIUM         Length; numeric, lowercase/uppercase, and special characters                             
2 or STRONG        Length; numeric, lowercase/uppercase, and special characters; dictionary file      
默认是1，即MEDIUM，所以刚开始设置的密码必须符合长度，且必须含有数字，小写或大写字母，特殊字符。

validate_password_special_char_count
密码至少要包含的特殊字符数。
```

### 修改参数

```js
mysql> set global validate_password_policy=0;  
Query OK, 0 rows affected (0.05 sec)  
  
mysql> set global validate_password_mixed_case_count=0;  
Query OK, 0 rows affected (0.00 sec)  
  
mysql> set global validate_password_number_count=3;  
Query OK, 0 rows affected (0.00 sec)  
  
mysql> set global validate_password_special_char_count=0;  
Query OK, 0 rows affected (0.00 sec)  
  
mysql> set global validate_password_length=3;  
Query OK, 0 rows affected (0.00 sec)  
  
mysql> SHOW VARIABLES LIKE 'validate_password%';  
+--------------------------------------+-------+  
| Variable_name                        | Value |  
+--------------------------------------+-------+  
| validate_password_dictionary_file    |       |  
| validate_password_length             | 3     |  
| validate_password_mixed_case_count   | 0     |  
| validate_password_number_count       | 3     |  
| validate_password_policy             | LOW   |  
| validate_password_special_char_count | 0     |  
+--------------------------------------+-------+  
6 rows in set (0.00 sec) 
```


这样就可以设置简单的密码了

