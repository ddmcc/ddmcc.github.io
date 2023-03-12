---
layout: post
title:  "Spring Cloud Gateway工作原理"
date:   2023-03-11 15:34:00
categories: "Spring Cloud Gateway"
tags:  "Spring Cloud Gateway"
author: ddmcc
---

* content
{:toc}





## 工作流程示意图

下图概括介绍了 `Spring Cloud Gateway` 的工作流程：

![](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/images/spring_cloud_gateway_diagram.png)


客户端向 `Spring Cloud Gateway` 发出请求。如果网关 `HandlerMapping` 确定请求与路由匹配，则将其发送到网关 `WebHandler` 。`Handler` 通过特定的过滤器链来将请求发送到下游服务, 过滤器中间用虚线的原因是过滤器可以在发送代理请求之前和之后运行特定逻辑。顺序是所有“前置”过滤器逻辑都已执行，然后发出代理请求，后将运行“后置”过滤器逻辑


## 处理请求流程

要摸清请求流程，首先要找到请求入口，网关请求入口在 `DispatcherHandler` ，当收到请求后，会执行 `handle` 方法

#### **根据请求获取handle**

当请求到达 `DispatcherHandler` 时，向 `handlerMappings` 获取处理请求的 `handler` ，`handlerMappings` 就是所有路由的映射，包括 `Controller` 接口、一些系统自带的接口及下游服务，如果没找到则直接返回 `404`


遍历handlerMappings获取handle源码：

```java
public Mono<Void> handle(ServerWebExchange exchange) {
    // 没有 handlerMappings ，直接返回 404
    if (this.handlerMappings == null) {
        return createNotFoundError();
    }
    // OPTIONS 请求
    if (CorsUtils.isPreFlightRequest(exchange.getRequest())) {
        return handlePreFlight(exchange);
    }
    // handlerMappings 中获取匹配的handler，没有找到则返回404
    return Flux.fromIterable(this.handlerMappings)
        .concatMap(mapping -> mapping.getHandler(exchange))
        .next()
        .switchIfEmpty(createNotFoundError())
        .flatMap(handler -> invokeHandler(exchange, handler))
        .flatMap(result -> handleResult(exchange, result));
}
```

可以看到 `handlerMappings` 其实是一个列表，其中有保存 `Controller` 接口的 `RequestMappingHandlerMapping` ，还有保存服务路由的 `RoutePredicateHandlerMapping`

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/WeChatf9b248892549c25a8796f72d6204f535.png)

>我们也可以定义自己 `handlerMapping`，通过继承 `AbstractHandlerMapping` ，重写获取handle的方法 `getHandlerInternal`，这里可以根据请求路径等匹配，然后返回handle处理器就是用于处理请求的类


#### **根据请求预设规则匹配路由**

`RoutePredicateHandlerMapping#getHandlerInternal` 方法返回的是具体的处理器 `handle`，这里handle其实是同一个 `webHandler`，并不会根据路径接口或者路由的服务变化， **方法里的逻辑是匹配路由，然后把路由放到请求上下文中**

首先判断是不是访问管理端口的请求，是的话则不处理。接着根据请求查找路由，查找逻辑是判断当前请求是否符合预设的匹配规则。 默认情况下，通过 `DiscoveryClient` 创建的每个网关路由都会有一个默认的匹配规则：`/serviceId/**` ，这个 `serviceId` 是 `DiscoveryClient` 中服务的ID 也就是根据访问服务名来匹配对应的路由。找到路由后返回网关路由特定处理器 `webHandler` ，对应上面图中的 `Gateway Web Handler`


>DiscoveryClient：是用于从注册中心获取服务路由
>
>webHandler： 即 org.springframework.cloud.gateway.handler.FilteringWebHandler


```java
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
    // don't handle requests on management port if set and different than server port
    // 不处理管理端口请求
    if (this.managementPortType == DIFFERENT && this.managementPort != null
            && exchange.getRequest().getURI().getPort() == this.managementPort) {
        return Mono.empty();
    }
    exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());
    // 根据访问路径查找路由
    return lookupRoute(exchange)
            // .log("route-predicate-handler-mapping", Level.FINER) //name this
            .flatMap((Function<Route, Mono<?>>) r -> {
                exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
                if (logger.isDebugEnabled()) {
                    logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
                }
                // 路由存入请求上下文，在具体访问时取出
                exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
                // 返回网关路由特定处理器 org.springframework.cloud.gateway.handler.FilteringWebHandler
                return Mono.just(webHandler);
            }).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
                exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
                if (logger.isTraceEnabled()) {
                    logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
                }
            })));
}


// lookupRoute
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    return this.routeLocator.getRoutes()
        .concatMap(route -> Mono.just(route).filterWhen(r -> {
            exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
            // 根据请求判断所有断言
            return r.getPredicate().apply(exchange);
        })
                .doOnError(e -> logger.error(
                        "Error applying predicate for route: " + route.getId(),
                        e))
                .onErrorResume(e -> Mono.empty()))
        .next()
        .map(route -> {
            if (logger.isDebugEnabled()) {
                logger.debug("Route matched: " + route.getId());
            }
            validateRoute(route, exchange);
            return route;
        });
}

```

#### **webHandler处理路由**

网关的处理步骤和springMVC一样，由 `HandlerAdapter` 适配器处理，适配器也是一个列表，通过重写 `supports` 方法来判断是否支持该处理器。对于路由请求处理器webHandler来说，对应的适配器为 `org.springframework.web.reactive.result.SimpleHandlerAdapter`。

```java
	private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
			    // 判断能否处理该处理器，只有SimpleHandlerAdapter返回true
				if (handlerAdapter.supports(handler)) {
					return handlerAdapter.handle(exchange, handler);
				}
			}
		}
		return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
	
```

经过适配器最终会调用 `FilteringWebHandler#handle` 方法

```java
public Mono<Void> handle(ServerWebExchange exchange) {
    // 从上下文中取路由，在getHandlerInternal中放入的
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
    // 获取网关过滤器，
    List<GatewayFilter> gatewayFilters = route.getFilters();
    // 全局 + 网关过滤器，并按照 @Ordered注解值排序
    List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
    combined.addAll(gatewayFilters);
    AnnotationAwareOrderComparator.sort(combined);

    if (logger.isDebugEnabled()) {
        logger.debug("Sorted gatewayFilterFactories: " + combined);
    }

    return new DefaultGatewayFilterChain(combined).filter(exchange);
}
```

`Handler` 会使用一系列过滤器来处理，最终将请求发送到下游服务，其中过滤器又分全局过滤器和网关过滤器，全局过滤器所有路由都会执行，一般我们的鉴权、限流逻辑就是通过自定义全局过滤器来实现的，通过实现 `GlobalFilter` 接口来定义，也可以通过配置项 `spring.cloud.gateway.default-filters` 来配置，
只针对注册中心来的路由还可以通过配置 `spring.cloud.gateway.discovery.locator.filters`。网关处理器只对这个网关有效，也可以针对每个网关单独配置 `spring.cloud.gateway.routes[0].filters`

**在默认情况下，网关会为通过 `DiscoveryClient` 创建的路由定义一个断言和过滤器：**

- 默认断言是使用 `/serviceId/**` 定义的path断言，其中serviceId是 `DiscoveryClient` 中服务的ID
- 默认的过滤器使用正则表达式 `/serviceId/(?<remaining>.*)` 和替换的 `/${remaining}` 进行重写，用于在请求之前从路径中截取掉serviceId

![](https://ddmcc-1255635056.cos.ap-guangzhou.myqcloud.com/2bada070cf37553d857245a3eed14c2.png)


最终请求会在 `NettyRoutingFilter` 中发出