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


gateway支持多种配置路由的方式，如通过 `properties` 文件配置、通过接口添加、从注册中心获取等等。不管是在yml文件配置的 `routes` 节点，还是通过 `DiscoveryClient` 加载的注册中心的服务，亦或是通过 `/gateway/routes/{id}` 接口增加的，在网关中都会生成对应一个个 `RouteDefinition` ，再通过 
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

`DiscoveryClientRouteDefinitionLocator` 作为加载注册中心服务的路由定义定位器，其通过访问 `DiscoveryClient` 接口来获取注册中心的服务信息，再将其生成 `RouteDefinition`。所以第三方注册中心要想接入gateway，
则创建一个bean来实现 `DiscoveryClient` 接口即可

在这里 `DiscoveryClient` 也是采用了组合模式来实现，其结构如下图：

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/3661679383159_.pic.jpg)

gateway在配置类 `CompositeDiscoveryClientAutoConfiguration` 中，手动创建了组合器 `CompositeDiscoveryClient` ，将所有 `DiscoveryClient` 的bean注入并赋值到 `discoveryClients`，
在调用获取服务ID或者根据服务ID获取服务列表时会循环请求 `discoveryClients` 。 **这边要注意的是，getServices方法返回的是所有client的服务ID，getInstances返回的是第一个client的结果（根据这个id获取），至于访问client的顺序可以用注解@Ordered，所以serviceId最好不要与其他的重复**

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

**下面以nacos为例，是如何接入gateway并将服务作为路由的：**

`NacosDiscoveryClient` 实现了 `DiscoveryClient` 接口，并实现了 `getServices` 方法获取所有服务ID、`getInstances(String serviceId)` 方法获取服务对象信息，
`ServiceInstance` 对象主要有服务id、IP、端口、是否https、额外信息等字段，利用这些就可以生成相应的路由了

```java
public class NacosDiscoveryClient implements DiscoveryClient {

	private static final Logger log = LoggerFactory.getLogger(NacosDiscoveryClient.class);

	/**
	 * Nacos Discovery Client Description.
	 */
	public static final String DESCRIPTION = "Spring Cloud Nacos Discovery Client";

	private NacosServiceDiscovery serviceDiscovery;

	public NacosDiscoveryClient(NacosServiceDiscovery nacosServiceDiscovery) {
		this.serviceDiscovery = nacosServiceDiscovery;
	}

	@Override
	public String description() {
		return DESCRIPTION;
	}

    // 根据服务ID获取服务对象列表
	@Override
	public List<ServiceInstance> getInstances(String serviceId) {
		try {
			return serviceDiscovery.getInstances(serviceId);
		}
		catch (Exception e) {
			throw new RuntimeException(
					"Can not get hosts from nacos server. serviceId: " + serviceId, e);
		}
	}

    // 获取所有服务ID
	@Override
	public List<String> getServices() {
		try {
			return serviceDiscovery.getServices();
		}
		catch (Exception e) {
			log.error("get service name from nacos server fail,", e);
			return Collections.emptyList();
		}
	}

}
```

可以看到 `getInstances` 返回的是一个列表，而不是单个对象，这是因为返回的列表是这个服务的所有节点，用于负载均衡访问

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/3671679386390_.pic.jpg)



#### 生成 **Route**

现在路由定义信息已经有了，下面就可以通过定义信息来生成真正的 `Route` 。`Route` 通过 `RouteLocator` 生成，它的实现方式和前面一样，使用了组合的设计模式，`RouteDefinitionRouteLocator` 只是作为其获取路由的方式之一

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/3691679388909_.pic.jpg)

在创建 `routeDefinitionRouteLocator` 时会将 `RouteDefinitionLocator` 注入进来，因为在创建 `CompositeRouteDefinitionLocator` 时被声明为@Primary，所以这里注入的对象就是组合类

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/3681679388267_.pic.jpg)

生成 `route` 的时候通过 `CompositeRouteDefinitionLocator` 获取所有的 `RouteDefinition` ，然后转为route

```java
@Override
public Flux<Route> getRoutes() {
    Flux<Route> routes = this.routeDefinitionLocator.getRouteDefinitions()
            .map(this::convertToRoute);
    // 省略一部分逻辑
    return routes.map(route -> {
        if (logger.isDebugEnabled()) {
            logger.debug("RouteDefinition matched: " + route.getId());
        }
        return route;
    });
}

private Route convertToRoute(RouteDefinition routeDefinition) {
    AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
    List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);
    return Route.async(routeDefinition).asyncPredicate(predicate).replaceFilters(gatewayFilters).build();
}
```

得益于组合模式，我们也可以很方便的自定义 `RouteLocator` 去注册路由：定义MyRouteLocator，并实现RouteLocator的getRoutes方法

```java
@Component
public class MyRouteLocator implements RouteLocator {
    
    @Override
    public Flux<Route> getRoutes() {
        return null;
    }
}
```

可以看到 `MyRouteLocator` 已被注入到 `CompositeRouteLocator`

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/3701679389911_.pic.jpg)


## 路由动态管理

