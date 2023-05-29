---
layout: post
title:  "websocket服务集群搭建与消息收发实现"
date:   2023-05-29 15:34:00
categories: websocket
tags:  websocket
toc: true
---

本文介绍如何将websocket服务注册到nacos中，并实现负载均衡调用，以及集群模式下消息（单聊，群发）的收发处理

<!-- more -->

## 注册websocket服务到nacos

#### 定义websocket服务

这边使用了 [netty-websocket](https://github.com/YeautyYE/netty-websocket-spring-boot-starter) 来开发搭建websocket服务，其中 `${ws.path}` 和 `${ws.port}` 是配置项，`{userId}` 是前端传入的路径参数，相当于 `@PathVariable`

```java
@Slf4j
@Component
@ServerEndpoint(path = "${ws.path}/{userId}", port = "${ws.port}")
public class WebsocketEndpoint {

}
```

**WebsocketProperties**

```properties
ws:
  name: "websocket-demo"
  path: "/ws"
  port: "19998"
```

#### 将服务注册到nacos中

注册到` nacos` 主要是利用 `NamingService` 提供的接口 `namingService.registerInstance`。首先注入项目的 `nacos` 配置，创建 `NamingService` ，然后调用 `registerInstance` 注册，包含3个参数
`name`,`ip`,`port` 。`name` 是服务名，待会我们配置负载均衡或者前端访问会用到，`ip`和`port` 是上面定义端点的时候用的


```java
@Slf4j
@Component
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class WebsocketServiceRegister implements CommandLineRunner {


    private final WebsocketProperties websocketProperties;
    private final NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public void run(String... args) throws Exception {
        Properties properties = new Properties();
        // 根据项目的nacos配置创建NamingService
        properties.setProperty("serverAddr", nacosDiscoveryProperties.getServerAddr());
        properties.setProperty("namespace", nacosDiscoveryProperties.getNamespace());
        NamingService namingService = NamingFactory.createNamingService(properties);
        // 注册websocket服务节点
        namingService.registerInstance(websocketProperties.getName(), nacosDiscoveryProperties.getIp(), websocketProperties.getPort());
    }
}
```

#### 配置网关负载均衡

现在服务已经起来了，也注册到 `nacos` 上了，下一步就是要配置网关，让网关能够正确路由到 `websocket` 端点，并且在多节点下也能够实现负载均衡。直接上配置：

```yaml
spring:
  cloud:
    gateway:
      enabled: true
      routes:
        - id: wsRouter
          uri: lb:ws://websocket-demo
          predicates:
            - Path=/websocket-demo/**
          filters:
            - StripPrefix=1
```

上面配置的意思就是当访问路径匹配上 `/websocket-demo/**` 时，去访问 `lb:ws://websocket-demo` ，`lb` 代表负载均衡访问的意思，访问 `lb:ws://websocket-demo` 时，会拿着 `websocket-demo` 找服务节点，也就是上面注册nacos时的 `name` ，向nacos获取到节点后，访问路径会被重写为 
`ws://ip:port/**`

这时websocket服务基本就搭建完成了，前端访问 `http://gatewayIp:gatewayPort/websocket-demo/ws/*` 就能连接上websocket服务


## 集群模式下消息收发处理

集群模式下最大的问题是 `session` 共享问题，因为 `session` 不能持久化，就会出现比如：用户1连接请求打在了节点A，这时对应的连接session就保存在了节点A里，然后来了一个给用户1发送消息的请求，此时并不能保证会请求到节点A，如果请求不在A也就无法找到session向A发送消息

这边使用 **redis + 消息队列** 来解决这个问题：

#### 群聊模式

因为 `session` 分散存储在各个节点，群聊相当于 `全量` 的发送，这时可以利用消息队列的 **广播模式** ，将消息广播到每个节点，节点内在去获取 `session` 发送


**下面以rocketMQ为例：**

- 群聊消息发送

```java
private final RocketMQTemplate template;

template.syncSend("GROUP:TAG", JSONObject.toJSONString(message));
```

- 群聊消息消费

主要是 `messageModel = MessageModel.BROADCASTING` ，消息模型声明为广播模式

```java
@Slf4j
@Component
@RocketMQMessageListener(consumerGroup = "group", topic = "GROUP", messageModel = MessageModel.BROADCASTING)
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class GroupChatMessageConsumer implements RocketMQListener<ReceiveMessage> {


    @Override
    public void onMessage(ReceiveMessage dto) {
        log.info("开始消费群聊消息：{}", dto);
        Long to = dto.getTo();
        // 根据群ID获取群人员或者全部发送
        SessionCache.USER_SESSION_MAP.entrySet().forEach(item -> {
            SessionWrapper sessionWrapper = item.getValue();
            if (sessionWrapper != null) {
                // 发送
            //    sessionWrapper.sendMessage(JSONObject.toJSONString(sendMessage))
            }
        });

    }
}
```


#### 单聊模式

单聊模式在集群节点少的情况下用上面的广播消息也可以，但在节点较多的情况下会产生大量冗余消息，造成性能问题，比如说：用户A的session存储在节点1，这时广播消息，那么节点2,3,4,5...也会收到，再去处理的话就显得多余了。这边的解决方案是在 `redis` 中存储的用户与节点的关系，然后将消息直接发送给对应的节点

- 收到连接请求时

将用户与节点关系存到redis中，比如： key -> userId, value -> nodeName

```java

// 节点
private final String discoveryIp;

@Override
public void handle(SessionWrapper session) {
    Long userId = session.getUserId();
    log.info("新用户连接 userId：{}", userId);
    // 用户与服务器ip关系存入redis
    stringRedisTemplate.opsForValue().set(super.buildUserNodeKey(userId), discoveryIp);
    // 用户session存入map
    SessionWrapper sessionWrapper = SessionCache.USER_SESSION_MAP.computeIfPresent(userId, (k, v) -> {
        log.info("用户userId：{} 在其它地方上线了，旧sessionId关闭：{}", k, v.getId());
        v.close();
        return session;
    });
    SessionCache.USER_SESSION_MAP.put(userId, sessionWrapper);
}
```

- 收到websocket消息时

根据消息接收人拿到对应session所在的节点，发送消息到对应节点

```java
String node = stringRedisTemplate.opsForValue().get("对应接收人ID");
template.syncSend("SINGLE:" + node, JSONObject.toJSONString(message));
```

- 消费消息时

demo用的是 `rocketMQ` ， **请注意不要将节点定义成相同的消费组，否则可能会丢消息，也就是如果利用tag来过滤消息，记得将消费组定义成不同的**


**消费者定义：**

```java
@Slf4j
@Component
@RocketMQMessageListener(topic = "SINGLE", consumerGroup = "simple-${node_name}", selectorType = SelectorType.TAG, selectorExpression = "${node_name}")
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class SimpleChatMessageConsumer implements RocketMQListener<ReceiveMessage> {

    private final SessionHandler closeSessionHandler;


    @Override
    public void onMessage(ReceiveMessage dto) {
        log.info("开始消费聊天消息：{}", dto);
        SessionWrapper sessionWrapper = SessionCache.USER_SESSION_MAP.get(dto.getTo());
        if (sessionWrapper == null) {
            log.info("未找到消息接收人：{} ，可能已下线", dto.getTo());
        } else if (sessionWrapper.isClose()) {
            log.info("消息接收人链接已关闭：{}，可能已断开链接", dto.getTo());
            closeSessionHandler.handle(sessionWrapper);
        } else {
            // 发送
            // sessionWrapper.sendMessage(JSONObject.toJSONString(sendMessage));
        }
    }
}
```

当然如果用其它消息队列可能会更直接点，比如说 `rabbitMQ` 可以直接用节点ip绑定队列

## 参考

 [`netty-websocket`](https://github.com/YeautyYE/netty-websocket-spring-boot-starter)
 [demo代码](https://github.com/ddmcc/websocket-demo)