---
title: Java web 中几种编码错误的原因及解决方案
date: 2018-05-31 17:16:41
tags: [Java]
---

Java 的编码问题好烦人，写个小项目都能碰见好几个……记录一些我遇到并解决的几个编码问题。


<!--more-->

### Java 获得页面上传的参数乱码
```java
request.setCharacterEncoding("utf-8");
```

### 前端的页面显示乱码
```java
response.setContentType("text/html;charset=utf-8");
```

### 数据库查询乱码 && 无法正常查询
修改 Java 的数据库连接字符串，添加如下字段
```java
?useUnicode=true&amp;characterEncoding=utf8
```