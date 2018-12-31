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

文档Documentation定义得很清晰，主要由groupName(分组名)、basePath(contextPath)、apiListings(API列表集)、resourceListing(资源列表集)等属性组成。

其中API列表被封装成ApiListing。ApiListing中又持有ApiDesciption集合引用，每个ApiDesciption都持有一个API集合的引用，Operation也就是具体的接口操作，内部包含了该接口对应的http方法、produces、consumes、协议、参数集、响应消息集等诸多元素。

**springfox通过spring-plugin的方式将Plugin注册到Spring上下文中，然后使用这些plugin进行API的扫描工作，这里的扫描工作其实也就是构造Documentation的工作，把扫描出的结果封装成Documentation并放入到DocumentationCache内存缓存中，之后swagger-ui界面展示的API信息通过Swagger2Controller暴露，Swagger2Controller内部直接从DocumentationCache中寻找Documentation。**

下图就是部分Plugin具体构造对应的文档信息：

![plugin](http://onekook.com/bower_components/extend/images/plugin.jpg)

代码细节方面的分析：

很明显，入口处在@EnableSwagger2注解上，该注解会import一个配置类Swagger2DocumentationConfiguration。

Swagger2DocumentationConfiguration做的事情：

1. 构造Bean。比如HandlerMapping，HandlerMapping是springmvc中用于处理请求与handler(controller中的方法)之间映射关系的接口，springboot中默认使用的HandlerMapping是RequestMappingHandlerMapping，Swagger2DocumentationConfiguration配置类里构造的是PropertySourcedRequestMappingHandlerMapping，该类继承RequestMappingHandlerMapping。
2. import其它配置类，比如SpringfoxWebMvcConfiguration、SwaggerCommonConfiguration
3. 扫描指定包下的类，并注册到Spring上下文中

SpringfoxWebMvcConfiguration配置类做的事情跟Swagger2DocumentationConfiguration类似，不过多了一步构造PluginRegistry过程。该过程使用@EnablePluginRegistries注解实现：

```java
@EnablePluginRegistries({ DocumentationPlugin.class,
    ApiListingBuilderPlugin.class,
    OperationBuilderPlugin.class,
    ParameterBuilderPlugin.class,
    ExpandedParameterBuilderPlugin.class,
    ResourceGroupingStrategy.class,
    OperationModelsProviderPlugin.class,
    DefaultsProviderPlugin.class,
    PathDecorator.class,
    ApiListingScannerPlugin.class
})
```
