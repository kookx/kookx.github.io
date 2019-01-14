---
layout: post
title: SpringFox+Swagger2踩坑记
category: Blog
tags: [Swagger2,SpringFox]
---

## 主要问题

### 1.原项目Spring-4.0.2，整合Swagger2后升级为4.3.8后，项目里CXF-2.4.3报错不兼容

> 问题及解决方案

1. 提示NoSuchMethod.isCglibProxyClass，Spring高版本移除掉了，这个类在spring-aop包中，建议一致更新高版本解决依赖。

2. 提示找不到cxf-extension-soap.xml，因为高版本将之移除了，注释掉cxf.xml里的引用即可。

链接1：[NoSuchMethodError:isCglibProxyClass](https://blog.csdn.net/laokaizzz/article/details/75734405)

链接2：[找不到cxf-extension-soap.xml](https://bbs.csdn.net/topics/390897128)

---

## 2.依赖丢失

 `添加依赖`

```haxe
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.7.0</version>
        </dependency>
```

运行时发现少包，根据项目需要自行添加其他包

```liquid
        <dependency>
            <groupId>com.fasterxml</groupId>
            <artifactId>classmate</artifactId>
            <version>1.3.3</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>18.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-spi</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-schema</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-common</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-spring-web</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.reflections</groupId>
            <artifactId>reflections</artifactId>
            <version>0.9.11</version>
        </dependency>
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>1.5.13</version>
        </dependency>
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-models</artifactId>
            <version>1.5.13</version>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>1.1.0.Final</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-core</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.plugin</groupId>
            <artifactId>spring-plugin-core</artifactId>
            <version>1.2.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.plugin</groupId>
            <artifactId>spring-plugin-metadata</artifactId>
            <version>1.2.0.RELEASE</version>
        </dependency>
```

### 3.swagger被拦截器拦截

> 修改SwaggerConfig配置类,excludePathPatterns("/swagger-ui.html","/v2/api-docs","/swagger-resources/**","/webjars/**");添加所有swagger需要用到的资源，绕过拦截。

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
@EnableWebMvc
public class WebAppConfig extends WebMvcConfigurerAdapter{
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new SecurityInterceptor())//自定义拦截器
        .addPathPatterns("/api/**")//需要拦截的的url
        .excludePathPatterns("/sys/login","/sys/test");//不需要拦截的url
        super.addInterceptors(registry);
    }
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry){
        registry.addResourceHandler("swagger-ui.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars");
    }
}
```
