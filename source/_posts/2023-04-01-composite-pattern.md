---
layout: post
title:  "设计模式之组合模式"
date:   2023-04-01 15:34:00
categories: 设计模式
tags:  设计模式 装饰者模式
toc: true
---

组合模式（Composite Pattern），又叫部分整体模式，是用于把一组相似的对象当作一个单一的对象。 组合模式依据树形结构来组合对象，用来表示部分以及整体层次。 这种类型的设计模式属于结构型模式，它创建了对象组的树形结构。 这种模式创建了一个包含自己对象组的类

<!-- more -->

## 结构

![markdown](https://ddmcc-1255635056.file.myqcloud.com/0cc84eba-62ec-4fcd-aa5e-1adfb90399a4.png)

>Leaf: 叶子节点
>Composite：组合对象

`Leaf` 叶子节点和 `Composite` 组合对象实现共同接口，不同的是子节点在方法中编写具体的处理逻辑，组合对象的作用是循环所有子节点的处理方法。如上图，`Composite` 维护一个子对象列表 List<Component> ，
当使用者向Composite对象（类型）发送请求时，该请求被转发到所有子Component对象（Leaf和Composite）

>为什么说该请求被转发到所有子Leaf和Composite？因为组合对象维护的是List<Component>，是顶层Component，所以可以是一个组合对象Composite


组合对象Composite中，通常还会有提供对子列表操作的方法，用于在运行时动态添加或者结合Spring的自动注入子组件，这样的好处就是外部的子组件也可以被添加进来并被执行

## 源码中的例子

在 `Spring Cloud Gateway` 中，除了可以通过配置文件的方式还可以从外部的注册中心动态拉取服务列表，比如网关项目整合了nacos，网关就能将路由到我们注册nacos中的服务，那这是怎么做到的呢？

首先 `Spring Cloud Gateway` 提供了统一获取服务列表的接口 `DiscoveryClient` ，各个服务注册框架都会去适配它，比如nacos中的 `NacosDiscoveryClient` 。那gateway是怎么知道又怎么调用到它的呢？


**DiscoveryClient接口：** 

```java
public interface DiscoveryClient extends Ordered {
    int DEFAULT_ORDER = 0;

    String description();

    List<ServiceInstance> getInstances(String serviceId);

    List<String> getServices();

    default int getOrder() {
        return 0;
    }
}
```

#### CompositeDiscoveryClient

gateway定义了一个组合对象 `CompositeDiscoveryClient` 创建时注入了所有的 `DiscoveryClient` ，并且 `CompositeDiscoveryClient` 声明为 `@Primary`，这意味着后面获取 `DiscoveryClient` 时，如果有多个会被优先获取

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(SimpleDiscoveryClientAutoConfiguration.class)
public class CompositeDiscoveryClientAutoConfiguration {

	@Bean
	@Primary
	public CompositeDiscoveryClient compositeDiscoveryClient(
			List<DiscoveryClient> discoveryClients) {
		return new CompositeDiscoveryClient(discoveryClients);
	}

}
```

在组合对象的实现逻辑中，实际是去循环调用 `discoveryClients` 列表，`discoveryClients` 也就是创建 `CompositeDiscoveryClient` 时传入的

```java
public List<ServiceInstance> getInstances(String serviceId) {
    if (this.discoveryClients != null) {
        Iterator var2 = this.discoveryClients.iterator();

        while(var2.hasNext()) {
            DiscoveryClient discoveryClient = (DiscoveryClient)var2.next();
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
            if (instances != null && !instances.isEmpty()) {
                return instances;
            }
        }
    }

    return Collections.emptyList();
}

public List<String> getServices() {
    LinkedHashSet<String> services = new LinkedHashSet();
    if (this.discoveryClients != null) {
        Iterator var2 = this.discoveryClients.iterator();

        while(var2.hasNext()) {
            DiscoveryClient discoveryClient = (DiscoveryClient)var2.next();
            List<String> serviceForClient = discoveryClient.getServices();
            if (serviceForClient != null) {
                services.addAll(serviceForClient);
            }
        }
    }

    return new ArrayList(services);
}
```


通过上面可以看出，gateway使用组合模式来实现获取服务节点列表，并且结合了spring容器注入，所以 `NacosDiscoveryClient` 想要被调用到，还需要注册到spring容器中

```java
public class NacosDiscoveryClientConfiguration {

	@Bean
	public DiscoveryClient nacosDiscoveryClient(
			NacosServiceDiscovery nacosServiceDiscovery) {
		return new NacosDiscoveryClient(nacosServiceDiscovery);
	}
}
```