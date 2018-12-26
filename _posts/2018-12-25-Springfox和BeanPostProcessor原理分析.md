---
layout: post
title: Springfox和BeanPostProcessor原理分析
category: Blog
tags: [Swagger2,SpringFox]
---

> springfox 是什么

`springfox`的前身是`swagger-springmvc`，用于`springmvc`与`swagger`的整合

        鉴于`swagger`的强大功能，`Java`开源界大牛`spring`框架迅速跟上，它充分利用自已的优势，把`swagger`集成到自己的项目里，整了一个`spring-swagger`，后来便演变成`springfox`。`springfox`本身只是利用自身的`aop`的特点，通过`plug`的方式把`swagger`集成了进来，它本身对业务`api`的生成，还是依靠`swagger`来实现。

        在项目启动过程中，`spring`上下文初始化时自动跟据配置加载`swagger`相关的`bean`到当前上下文中，并自动扫描系统中可能需要生成`api`文档那些类，并生成相应的信息缓存起来。如果项目`MVC`控制层用的是`springMvc`那么会自动扫描所有`Controller`类，跟据这些`Controller`类中的方法生成相应的`api`文档。

如若在springboot项目中使用springfox，需要3个步骤：

1. maven添加springfox依赖
2. 启动类加上@EnableSwagger2注解
3. 构造Docket bean用于展示API

配置完之后进入[http://](http://link.zhihu.com/?target=https%3A//yq.aliyun.com/articles/599809%3Futm_content%3Dm_1000002417){path}:{port}/swagger-ui.html 即可查看controller中的接口信息，并按照Docket中配置的规则进行展示。

> springfox如何定义Documentation：

![springfox](http://onekook.com/bower_components/extend/images/springfox.jpg)
