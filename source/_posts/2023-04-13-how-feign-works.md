---
layout: post
title:  "OpenFeign的实现原理"
date:   2023-04-13 15:34:00
categories: Spring Cloud
tags:  OpenFeign
toc: true
---

## 前言

在过去使用feign时，会将feign接口单独放在一个包，并在接口类上声明 `@FeignClient(name = 服务名)`， 在接口方法上声明 `@xxxMapping(接口路径)` ，实现者和使用者都会去引入这个包。对于 **使用者** 来说只需要注入接口类，然后调用对应的方法就能调用到在另一个服务的实现者实现的逻辑

<!-- more -->


根据过去的经验我猜测，在调用侧上的原理和mybatis相似，扫描所有被 `@FeignClient` 注解声明的类，然后为其生成代理对象，当执行某个方法时，会被拦截转而请求方法上的链接，不同的是mybatis是找到这个方法对应的MappedStatement对象，然后执行sql。实现侧的话则和普通controller类似，将这些方法封装成一个个handle并注册到handleMapping中，等待被DispatcherServlet调用


## 调用侧

为了验证上面的猜想，我们从 `@EnableFeignClients` 入手，看看启动时都做了什么

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    
}
```

可以看到注解 `EnableFeignClients` 通过 `@Import` 注解导入一个配置类 `FeignClientsRegistrar`，它实现了 `ImportBeanDefinitionRegistrar` 接口，这么说在启动的时候，`FeignClientsRegistrar` 类中的 `registerBeanDefinitions` 方法会被调用，来往Spring容器中注册 `BeanDefinition`

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 解析 @EnableFeignClients 注解中的配置，并注册到容器
    registerDefaultConfiguration(metadata, registry);
    // 扫描被 @FeignClient 注解声明的接口，并注册到容器
    registerFeignClients(metadata, registry);
}
```

`registerFeignClients` 方法是重点，这里扫描到所有 `@FeignClient` 注解声明的接口后，并做了一系列的处理

```java
public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
    // 获取EnableFeignClients注解配置属性
    Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
    // 获取配置的clients，如果有配置，则不再进行类路径扫描
    final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);
        // 指定要扫描的注解
        scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
        // 获取EnableFeignClients中配置的包路径，如果没有就默认为注解所在类的包路径
        Set<String> basePackages = getBasePackages(metadata);
        for (String basePackage : basePackages) {
            // 扫描并构建ScannedGenericBeanDefinition
            candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
        }
    } else {
        for (Class<?> clazz : clients) {
            candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
        }
    }

    for (BeanDefinition candidateComponent : candidateComponents) {
        if (candidateComponent instanceof AnnotatedBeanDefinition beanDefinition) {
            // verify annotated class is an interface
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            Assert.isTrue(annotationMetadata.isInterface(), "@FeignClient can only be specified on an interface");

            Map<String, Object> attributes = annotationMetadata
                    .getAnnotationAttributes(FeignClient.class.getCanonicalName());

            String name = getClientName(attributes);
            String className = annotationMetadata.getClassName();
            // 注册client的配置到容器  -> FeignClientSpecification
            registerClientConfiguration(registry, name, className, attributes.get("configuration"));
            // 创建client的BeanDefinition，注册到容器
            registerFeignClient(registry, annotationMetadata, attributes);
        }
    }
}
```

先获取 `@EnableFeignClients` 的配置，看是否有配置指定的client，有的话就只生成配置的BeanDefinition，没有的话再用ClassPathScanningCandidateComponentProvider来扫描被 `@FeignClient` 注解标记的类，扫描的路径的话看有没有配置，如果没有配置则默认为 `@EnableFeignClients` 注解所在类的所在包开始向下扫描。扫描到的类都会生成一个BeanDefinition，可以把BeanDefinition看成对每个标有 `@FeignClient` 注解的类信息的封装。拿到所有BeanDefinition之后，遍历调用 `registerClientConfiguration` 和 `registerFeignClient` 


**registerClientConfiguration**

```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, 
                                        Object name, 
                                        Object className, 
                                        Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(className);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(name + "." + FeignClientSpecification.class.getSimpleName(), builder.getBeanDefinition());
}
```

这里的意思就是拿出再 `@FeignClient` 指定的配置类，也就是 `configuration` 属性，然后构建一个bean class为FeignClientSpecification。这个类的最主要作用就是将每个client的配置类封装成一个FeignClientSpecification的BeanDefinition，注册到spring容器中


**registerFeignClient**

```java
private void registerFeignClient(BeanDefinitionRegistry registry, 
                                AnnotationMetadata annotationMetadata,
                                Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    if (String.valueOf(false).equals(
            environment.getProperty("spring.cloud.openfeign.lazy-attributes-resolution", String.valueOf(false)))) {
        eagerlyRegisterFeignClientBeanDefinition(className, attributes, registry);
    }
    else {
        lazilyRegisterFeignClientBeanDefinition(className, attributes, registry);
    }
}

private void eagerlyRegisterFeignClientBeanDefinition(String className, Map<String, Object> attributes,
        BeanDefinitionRegistry registry) {
    validate(attributes);
    // 构建class为FeignClientFactoryBean的BeanDefinition
    BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
    definition.addPropertyValue("url", getUrl(null, attributes));
    definition.addPropertyValue("path", getPath(null, attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    String contextId = getContextId(null, attributes);
    definition.addPropertyValue("contextId", contextId);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("dismiss404", Boolean.parseBoolean(String.valueOf(attributes.get("dismiss404"))));
    Object fallback = attributes.get("fallback");
    if (fallback != null) {
        definition.addPropertyValue("fallback",
                (fallback instanceof Class ? fallback : ClassUtils.resolveClassName(fallback.toString(), null)));
    }
    Object fallbackFactory = attributes.get("fallbackFactory");
    if (fallbackFactory != null) {
        definition.addPropertyValue("fallbackFactory", fallbackFactory instanceof Class ? fallbackFactory
                : ClassUtils.resolveClassName(fallbackFactory.toString(), null));
    }
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    definition.addPropertyValue("refreshableClient", isClientRefreshEnabled());
    String[] qualifiers = getQualifiers(attributes);
    if (ObjectUtils.isEmpty(qualifiers)) {
        qualifiers = new String[] { contextId + "FeignClient" };
    }
    // This is done so that there's a way to retrieve qualifiers while generating AOT
    // code
    definition.addPropertyValue("qualifiers", qualifiers);
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);
    // has a default, won't be null
    boolean primary = (Boolean) attributes.get("primary");
    beanDefinition.setPrimary(primary);
    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, qualifiers);
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
    registerRefreshableBeanDefinition(registry, contextId, Request.Options.class, OptionsFactoryBean.class);
    registerRefreshableBeanDefinition(registry, contextId, RefreshableUrl.class, RefreshableUrlFactoryBean.class);
}
```
