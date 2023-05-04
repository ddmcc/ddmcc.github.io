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

#### 扫描@FeignClient注解类

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

这里的意思就是拿出再 `@FeignClient` 指定的配置类，也就是 `configuration` 属性，然后构建一个bean class为FeignClientSpecification。这个类的最主要作用就是将每个client的配置类（`configuration` 属性）封装成一个 `FeignClientSpecification` 的 `BeanDefinition`，注册到spring容器中


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

默认情况下会调用 `eagerlyRegisterFeignClientBeanDefinition` ，所以我们来看这个方法做了哪些事。先构建了一个class为 `FeignClientFactoryBean` 的BeanDefinition，这个class实现了FactoryBean接口，spring在生成bean的时候判断BeanDefinition中bean的class如果是FactoryBean的实现的话，会调用这个实现类的getObject来获取对象。

到这里生成动态代理对象的准备工作就基本做完了，再来总结一下前面做了哪些：根据  `@EnableFeignClients` 注解的配置扫描指定（不指定就默认路径下的）包下所有加了 `@FeignClient` 注解的类，然后每个类都会生成一个BeanDefinition，随后 **遍历每个BeanDefinition** ，然后取出每个 `@FeignClient` 注解的属性，构造class为 `FeignClientFactoryBean` 的新的BeanDefinition，随后注册到spring容器中，同时有配置类（注解上configuration属性）的也会将配置类构件出一个class为 `FeignClientSpecification` 的BeanDefinition注册到spring容器中

#### 生成动态代理对象

上面每个 `@FeignClient` 都生成了一个 class为 `FeignClientFactoryBean` 的BeanDefinition，后面就会根据这个BeanDefinition来生成动态代理对象。因为FeignClientFactoryBean实现了FactoryBean接口，所以会调用 `getObject()` 来获取对象：

```java
@Override
public Object getObject() {
    return getTarget();
}


<T> T getTarget() {
    // FeignClientFactory 里面保存了每个client的配置，也就是FeignClientSpecification（具体可以看FeignAutoConfiguration）
    FeignClientFactory feignClientFactory = beanFactory != null ? beanFactory.getBean(FeignClientFactory.class)
            : applicationContext.getBean(FeignClientFactory.class);

    // 获取builder，Feign.Builder对象在FeignClientsConfiguration中配置
    Feign.Builder builder = feign(feignClientFactory);
    
    // 判断是否配置了url属性，url链接直连访问
    if (!StringUtils.hasText(url) && !isUrlAvailableInConfig(contextId)) {

        if (LOG.isInfoEnabled()) {
            LOG.info("For '" + name + "' URL not provided. Will try picking an instance via load-balancing.");
        }
        if (!name.startsWith("http")) {
            url = "http://" + name;
        }
        else {
            url = name;
        }
        url += cleanPath();
        // 负载均衡访问
        return (T) loadBalance(builder, feignClientFactory, new HardCodedTarget<>(type, name, url));
    }
    if (StringUtils.hasText(url) && !url.startsWith("http")) {
        url = "http://" + url;
    }
    String url = this.url + cleanPath();
    Client client = getOptional(feignClientFactory, Client.class);
    if (client != null) {
        if (client instanceof FeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
        }
        if (client instanceof RetryableFeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((RetryableFeignBlockingLoadBalancerClient) client).getDelegate();
        }
        builder.client(client);
    }

    applyBuildCustomizers(feignClientFactory, builder);

    Targeter targeter = get(feignClientFactory, Targeter.class);
    return targeter.target(this, builder, feignClientFactory, resolveTarget(feignClientFactory, contextId, url));
}
```

先获取FeignClientFactory，这里保存了每个client的配置，然后获取到一个Feign.Builder，然后调用feign方法。feign方法主要事设置一些默认的组件，如果需要更改上面的这些组件，可以通过 `@FeignClient` 的 `configuration` 属性配置配置类，在配置类里面替换


```java
protected Feign.Builder feign(FeignClientFactory context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(type);

    Feign.Builder builder = get(context, Feign.Builder.class)
            // required values
            .logger(logger)
            .encoder(get(context, Encoder.class))
            .decoder(get(context, Decoder.class))
            .contract(get(context, Contract.class));

    // 从配置文件中读取feign的配置
    configureFeign(context, builder);

    return builder;
}
```

下面就是这段逻辑：

```java
if (!StringUtils.hasText(url) && !isUrlAvailableInConfig(contextId)) {

    if (LOG.isInfoEnabled()) {
        LOG.info("For '" + name + "' URL not provided. Will try picking an instance via load-balancing.");
    }
    if (!name.startsWith("http")) {
        url = "http://" + name;
    }
    else {
        url = name;
    }
    url += cleanPath();
    // 负载均衡访问
    return (T) loadBalance(builder, feignClientFactory, new HardCodedTarget<>(type, name, url));
}
```

先判断有没有指定url，也就是在 `@FeignClient` 注解中指定的url属性，如果配置了这个属性，就不通过注册中心，直接访问链接，`isUrlAvailableInConfig()` 也是判断在配置文件中是否配置了url属性 。一般情况下这个是不配置的，因为得从注册中心获取服务的ip和端口列表，然后进行负载均衡访问。所以从这也也可以看出，没有注册中心，feign也是能够跑的，只要配置url属性就行

下面就是拼接url，name就是我们在 `@FeignClient` 配置的value，一般是服务名，这段代码拼出来的结果就类似 `http://ServiceA`，之后就会走loadBalance方法，传入一个HardCodedTarget参数，封装了feign客户端接口的类型、服务的名称、还有刚构建的url


```java
protected <T> T loadBalance(Feign.Builder builder, FeignClientFactory context, HardCodedTarget<T> target) {
    Client client = getOptional(context, Client.class);
    if (client != null) {
        builder.client(client);
        applyBuildCustomizers(context, builder);
        Targeter targeter = get(context, Targeter.class);
        return targeter.target(this, builder, context, target);
    }

    throw new IllegalStateException(
            "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-loadbalancer?");
}
```

首先从feign客户端对应的ioc容器中获取一个Client，这个client对象由配置类注册，默认情况下由 `DefaultFeignLoadBalancerConfiguration` 提供，使用了 `spring-cloud-starter-loadbalancer` 组件，可能就由 `FeignRibbonClientAutoConfiguration` 去提供。在旧版本openfeign下使用了开启了ribbon(`spring.cloud.loadbalancer.ribbon.enabled`)的话由 `FeignRibbonClientAutoConfiguration` 提供

获取到Client后，接下来获取到Targeter，Targeter是通过 `FeignAutoConfiguration` 来配置的，默认是DefaultTargeter，如果整合hystrix或sentinel就是FeignCircuitBreakerTargeter（看有没有CircuitBreakerFactory）。然后调用DefaultTargeter的target方法：

```java
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignClientFactory context,
        Target.HardCodedTarget<T> target) {
    return feign.target(target);
}
```

然后调用Feign.Builder的tartget方法：

```java
public <T> T target(Target<T> target) {
  return build().newInstance(target);
}


public Feign build() {
  super.enrich();

  MethodHandler.Factory<Object> synchronousMethodHandlerFactory =
      new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors,
          responseInterceptor, logger, logLevel, dismiss404, closeAfterDecode,
          propagationPolicy, options, decoder, errorDecoder);
  ParseHandlersByName<Object> handlersByName =
      new ParseHandlersByName<>(contract, encoder, queryMapEncoder,
          synchronousMethodHandlerFactory);
  return new ReflectiveFeign<>(handlersByName, invocationHandlerFactory, () -> null);
}
```


构建了一个ReflectiveFeign，然后调用ReflectiveFeign的newInstance方法，传入target，也就是前面传入的HardCodedTarget


```java
@SuppressWarnings("unchecked")
public <T> T newInstance(Target<T> target, C requestContext) {
TargetSpecificationVerifier.verify(target);

Map<Method, MethodHandler> methodToHandler =
    targetToHandlersByName.apply(target, requestContext);
InvocationHandler handler = factory.create(target, methodToHandler);
T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
    new Class<?>[] {target.type()}, handler);

for (MethodHandler methodHandler : methodToHandler.values()) {
  if (methodHandler instanceof DefaultMethodHandler) {
    ((DefaultMethodHandler) methodHandler).bindTo(proxy);
  }
}

return proxy;
}
```

这个方法解释一下做了什么，首先 `targetToHandlersByName.apply` 通过target拿到client接口类型，去遍历接口内所有的方法，然后通过 `Contract` 解析所有方法注解封装成 `MethodMetadata` ，然后根据 `MethodMetadata` 等生成 `MethodHandler` ，返回的map的key就是方法，值为该方法的处理器，处理器里有该方法解析好的 `RequestTemplate` 等

>Contract 主要是用来解析方法上的注解的 默认是 SpringMvcContract，所以能支持MVC的注解

后面就通过 `InvocationHandlerFactory` ，获取到一个InvocationHandler，之后通过jdk的动态代理，生成一个代理对象，InvocationHandler默认是 `ReflectiveFeign.FeignInvocationHandler`，在 `FeignInvocationHandler#invoke` 方法中会根据方法获取对应MethodHandler，具体的请求逻辑就在其invoke方法中，方法处理器有同步/异步请求等


```java

@Override
public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
  return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
}


FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
  this.target = checkNotNull(target, "target");
  this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
}


FeignInvocationHandler#invoke

@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if ("equals".equals(method.getName())) {
    try {
      Object otherHandler =
          args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
      return equals(otherHandler);
    } catch (IllegalArgumentException e) {
      return false;
    }
  } else if ("hashCode".equals(method.getName())) {
    return hashCode();
  } else if ("toString".equals(method.getName())) {
    return toString();
  }

  return dispatch.get(method).invoke(args);
}
```

#### 总结

![](https://ddmcc-1255635056.file.myqcloud.com/060a59aa-6a36-480b-a856-d979f9bd8364.png)



## 实现侧


开头说实现侧和 **普通controller类似，将这些方法封装成一个个handle并注册到handleMapping中，等待被DispatcherServlet调用** ，经过上面对 `@FeignClient` 注解的分析，发现并没有关于接口这块处理。那是怎么被识别为 `Controller` 接口的呢？

**先来回顾一下怎么定义接口：**

- 首先定义feign接口类

```java
@FeignClient(value = AppConfig.APP_NAME)
public interface ICompanyClient {

    String API_PREFIX = "/client/company";


    @GetMapping(API_PREFIX + "/find-by-id/{id}")
    R<CompanyApiVo> findById(@PathVariable String id);

}
```

一般feign接口会单独放在一个jar包中，接口实现和调用方分别引入该jar包


- 实现feign接口

现在接口定义已经被单独放在了一个jar中了，在具体实现的包中引入它，然后编写具体实现：

```java
@Slf4j
@RestController
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class CompanyClientImpl implements ICompanyClient {

    private final ICompanyService companyService;

    @Override
    public R<CompanyApiVo> findById(String id) {
        final Company company = companyService.findById(id);
        return R.data(CompanyConverter.INSTANCE.toCompanyApiVo(company));
    }
}
```


嗯。。。实际上这些方法能被识别并封装成一个个handle注册到handleMapping中，是因为实现类上加上了 `@RestController` 注解。在解析方法的时候又通过 `findMergedAnnotation` 获取方法的注解，所以标记在父类方法上的 `@xxxMapping` 也能获取到