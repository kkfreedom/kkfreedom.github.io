+++
author = ""
title = "Java 反射"
date = "2019-03-05"
description = "通过反射给没有 setter 方法的属性赋值，包括父类属性"
categories = [
    "编程"
]
tags = [
    "编程",
    "Java"
]
image = "cover.jpeg"
+++

# 需求
在 Junit 中，有时需要为子类继承自父类的属性赋值，但是父类中的属性没有提供 setter 方法，此时可以使用反射
-  假设类之间有如下继承关系
	- 父类-RequestBase
		- 子类-SearchRequest
- 其中父类中 filter 字段未提供 setter 方法，但是在 Junit 中需要为 filter 字段设置值以验证某些场景
 # 实现

  ```java
  SearchRequest request = null;
  Class<SearchRequest> clazz = SearchRequest.class;
  request = clazz.newInstance();
  // 获取父类 class 对象
  Class<? super SearchRequest> superclass = clazz.getSuperclass();
  
  List<Filter> filterList = new ArrayList();
  Filter filter = new BuyerFilter();
  filter.setPattern("*");
  filter.setType("java.lang.String");
  filter.setSearchFieldName("name");
  filterList.add(filter);
  // 反射获取父类字段并赋值给子类对象
  Field buyFilter = superclass.getDeclaredField("filter");
  buyFilter.setAccessible(true);
  buyFilter.set(request, buyerFilterList);
  // 子类字段直接赋值
  request.setLocale("en_US");
  ```
