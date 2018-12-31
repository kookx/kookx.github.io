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

@EnablePluginRegistries注解是spring-plugin模块提供的一个基于Plugin类型注册PluginRegistry实例到Spring上下文的注解。

@EnablePluginRegistries注解内部使用PluginRegistriesBeanDefinitionRegistrar注册器去获取注解的value属性(类型为Plugin接口的Class数组)；然后遍历这个Plugin数组，针对每个Plugin在Spring上下文中注册PluginRegistryFactoryBean，并设置相应的name和属性。

如果处理的Plugin有@Qualifier注解，那么这个要注册的PluginRegistryFactoryBean的name就是@Qualifier注解的value，否则name就是插件名首字母小写+Registry的格式(比如DocumentationPlugin对应构造的bean的name就是documentationPluginRegistry)。

PluginRegistriesBeanDefinitionRegistrar注册器处理过程：

```
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

   Class<?>[] types = (Class<?>[]) importingClassMetadata.getAnnotationAttributes(
         EnablePluginRegistries.class.getName()).get("value");

   for (Class<?> type : types) {

      BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(PluginRegistryFactoryBean.class);
      builder.addPropertyValue("type", type);

      AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
      Qualifier annotation = type.getAnnotation(Qualifier.class);

      // If the plugin interface has a Qualifier annotation, propagate that to the bean definition of the registry
      if (annotation != null) {
         AutowireCandidateQualifier qualifierMetadata = new AutowireCandidateQualifier(Qualifier.class);
         qualifierMetadata.setAttribute(AutowireCandidateQualifier.VALUE_KEY, annotation.value());
         beanDefinition.addQualifier(qualifierMetadata);
      }

      // Default
      String beanName = annotation == null ? StringUtils.uncapitalize(type.getSimpleName() + "Registry") : annotation
            .value();
      registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
   }
}
```

PluginRegistryFactoryBean是一个FactoryBean，其内部真正构造的bean的类型是OrderAwarePluginRegistry。OrderAwarePluginRegistry实例化过程中会调用create静态方法，传入的plugin集合使用aop代理生成一个ArrayList，这个list中的元素就是Spring上下文中所有的类型为之前遍历的Plugin的bean。  
PluginRegistryFactoryBean的getObject方法：

```java
public OrderAwarePluginRegistry<T, S> getObject() {
   return OrderAwarePluginRegistry.create(getBeans());
}
protected List<T> getBeans() {
   ProxyFactory factory = new ProxyFactory(List.class, targetSource);
   return (List<T>) factory.getProxy();
}
```

这里的targetSource是在PluginRegistryFactoryBean的父类AbstractTypeAwareSupport(实现了InitializingBean接口)中的afterPropertiesSet方法中初始化的(type属性在PluginRegistriesBeanDefinitionRegistrar注册器中已经设置为遍历的Plugin)：

```java
public void afterPropertiesSet() {
   this.targetSource = new BeansOfTypeTargetSource(context, type, false, exclusions);
}
```

BeansOfTypeTargetSource的getTarget方法：

```java
public synchronized Object getTarget() throws Exception {
   Collection<Object> components = this.components == null ? getBeansOfTypeExcept(type, exclusions)
         : this.components;

   if (frozen && this.components == null) {
      this.components = components;
   }

   return new ArrayList(components);
}

private Collection<Object> getBeansOfTypeExcept(Class<?> type, Collection<Class<?>> exceptions) {
  List<Object> result = new ArrayList<Object>();

  for (String beanName : context.getBeanNamesForType(type, false, eagerInit)) {
    if (exceptions.contains(context.getType(beanName))) {
      continue;
    }
    result.add(context.getBean(beanName));
  }

  return result;
}
```

举个例子：比如SpringfoxWebMvcConfiguration中的@EnablePluginRegistries注解里的DocumentationPlugin这个Plugin，在处理过程中会找出Spring上下文中所有的Docket(Docket实现了DocumentationPlugin接口)，并把该集合设置成name为documentationPluginRegistry、类型为OrderAwarePluginRegistry的bean，注册到Spring上下文中。

DocumentationPluginsManager类会在之前提到过的配置类中被扫描出来，它内部的各个pluginRegistry属性都是@EnablePluginRegistries注解内部构造的各种pluginRegistry实例：

```java
@Component
public class DocumentationPluginsManager {
  @Autowired
  @Qualifier("documentationPluginRegistry")
  private PluginRegistry<DocumentationPlugin, DocumentationType> documentationPlugins;
  @Autowired
  @Qualifier("apiListingBuilderPluginRegistry")
  private PluginRegistry<ApiListingBuilderPlugin, DocumentationType> apiListingPlugins;
  @Autowired
  @Qualifier("parameterBuilderPluginRegistry")
  private PluginRegistry<ParameterBuilderPlugin, DocumentationType> parameterPlugins;
  ...
}
```

DocumentationPluginsBootstrapper启动类也会在之前提供的配置类中被扫描出来。它实现了SmartLifecycle接口，在start方法中，会获取之前初始化的所有documentationPlugins(也就是Spring上下文中的所有Docket)。遍历这些Docket并进行scan扫描(使用RequestMappingHandlerMapping的getHandlerMethods方法获取url与方法的所有映射关系，然后进行一系列API解析操作)，扫描出来的结果封装成Documentation并添加到DocumentationCache中：

```java
@Override
public void start() {
  if (initialized.compareAndSet(false, true)) {
    log.info("Context refreshed");
    List<DocumentationPlugin> plugins = pluginOrdering()
        .sortedCopy(documentationPluginsManager.documentationPlugins());
    log.info("Found {} custom documentation plugin(s)", plugins.size());
    for (DocumentationPlugin each : plugins) {
      DocumentationType documentationType = each.getDocumentationType();
      if (each.isEnabled()) {
        scanDocumentation(buildContext(each));
      } else {
        log.info("Skipping initializing disabled plugin bean {} v{}",
            documentationType.getName(), documentationType.getVersion());
      }
    }
  }
}
```
