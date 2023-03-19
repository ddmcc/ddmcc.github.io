---
layout: post
title:  "Spring Cloud Gateway路由的管理"
date:   2023-03-14 15:34:00
categories: "spring-cloud-gateway"
tags:  "spring-cloud-gateway"
author: ddmcc
---

* content
{:toc}




## 路由的构建步骤


不管是在yml文件配置的 `routes` 节点，还是通过 `DiscoveryClient` 加载的注册中心的服务，亦或是通过 `/gateway/routes/{id}` 接口增加的，在网关中都会生成对应一个个 `RouteDefinition` ，再通过 
`RouteDefinition` 生成对应的 `route` 。


#### **RouteDefinition** 加载生成


`RouteDefinition` 通过 `RouteDefinitionLocator` 接口来定位获取，这里面使用了组合的设计模式（`Composite Pattern`）


![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/6fd6c1bd0f599226e7c6e39551f9f66.png)

如上图，就是它们的UML类图，`RouteDefinitionLocator` 是抽象接口，提供了 `getRouteDefinitions()` 方法用于外部获取 `RouteDefinition` 列表， `CompositeRouteDefinitionLocator` 作为组合对象里面持有一个 `List<RouteDefinitionLocator>` ，当调用
`getRouteDefinitions()` 方法时，会循环调用列表中具体的定位器的 `getRouteDefinitions()` 方法获取，然后合并成一个列表返回


![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/ad7dee95c04382c354f9196042d8954.png)

默认有上图几个定位器，他们的作用分别是：

- `PropertiesRouteDefinitionLocator：` 加载配置文件中 `spring.cloud.gateway.routes` 配置项的路由
- `DiscoveryClientRouteDefinitionLocator：` 加载注册中心的服务作为路由
- `InMemoryRouteDefinitionRepository：` 通过web接口提交的路由或者其它在程序内通过接口（RouteDefinitionWriter#save）增加的


![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/24c609347b577a257759b40c6482e7c.png)
![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/d6dad4a06177a3214d18d6c4e11f810.png)

上面这几个 `RouteDefinitionLocator` 由框架创建并注册到 `Spring` 容器中，可以看到 `CompositeRouteDefinitionLocator` 被注解 `@Primary` 标记，代表后续要自动注入 `RouteDefinitionLocator` 时会被优先注入

当然我们也可以自定义 `RouteDefinitionLocator`，只需要实现 `RouteDefinitionLocator`，然后将它注册到 `Spring` 容器即可，在创建 `CompositeRouteDefinitionLocator` 时会被自动注入到参数列表中。如下例子：定义一个名为 `MyRouteDefinitionLocator`
的定位器，并用 `@Component` 注解声明


**实现自定义 RouteDefinitionLocator：**
```java
/**
 * @author ddmcc
 */
@Component
public class MyRouteDefinitionLocator implements RouteDefinitionLocator {

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        // 自定义新增RouteDefinition
        return null;
    }

}
```

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/d0a207e1741c9269dd8294090452beb.png)


#### 加载注册中心中的服务

