---
layout: post
title:  "设计模式在工作中的应用"
date:   2022-10-12 16:56:23
categories: 设计模式
tags:  设计模式
author: ddmcc
---

* content
{:toc}





## 策略模式

在公司内部的供应链物流系统中，会与第三方物流承运商的系统进行对接。为了减少对对方系统的依赖，交互设计成“通知-反查”的形式，即当物流订单 “状态” 发生变更时，通过统一接口通知到我们系统，这边再根据不同的通知去做相应的处理。
比如：订单确认、拒绝、完成，车辆启运，位置更新等等的通知

在没有使用策略模式下，我们可能会写成这样：

```java
if (noticeType == 订单确认) {
    // 处理订单确认逻辑
} else if (noticeType == 位置更新) {
    // 更新物流位置逻辑
} 
....等等其它逻辑分支
```

每次新增通知都要这边新增一个分支，到最后可能会非常长，后续的扩展和维护也会变得复杂且容易出错。策略模式是解决 `if-else` 代码块过长的方法之一，且后续维护与变更会变得很容易

#### 策略定义

![markdown](https://ddmcc-1255635056.file.myqcloud.com/3e0acfbf-8fa5-475e-b3e0-7942f7d8336a.png)

通知接口包含获取通知类型和处理通知两个方法：

```java
/**
 * 物流通知
 *
 * @author jiangrz
 * @date 2021-12-29 11:11
 */
public interface INotice<V> {

    /**
     * 通知回调
     *
     * @param param param
     * @param merchantNo 商户编号
     * @return V
     * @author jiangrz
     * @date 2021/12/29 11:12 上午
     */
    V notice(CallbackParam param, String merchantNo);


    /**
     * 获取通知类型
     *
     * @return List<NoticeTypeEnum>
     * @author jiangrz
     * @date 2021/12/29 11:16 上午
     */
    List<NoticeTypeEnum> getNoticeType();
```

各个具体的处理类都需要实现上面两个方法：


```java
/**
 * 在途更新、车辆到达通知
 *
 * @author jiangrz
 * @date 2022-01-07 11:39
 */
@Slf4j
@Componet
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class LocationNotice implements INotice<Boolean> {
    
    @Override
    public Boolean notice(CallbackParam param, String merchantNo) {
        // 更新车辆位置
    }

    @Override
    public List<NoticeTypeEnum> getNoticeType() {
        return Lists.newArrayList(NoticeTypeEnum.LOCATION_UPDATE);
    }
}


/**
 * 车辆交付
 *
 * @author jiangrz
 * @date 2022-01-13 16:08
 */
@Slf4j
@Componet
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class OrderCompletedNotice extends INotice<Boolean> {

    @Override
    public Boolean notice(CallbackParam param, String merchantNo) {
        // 处理车辆交付
        return Boolean.TRUE;
    }

    @Override
    public List<NoticeTypeEnum> getNoticeType() {
        return Lists.newArrayList(NoticeTypeEnum.ORDER_CAR_COMPLETE);
    }
}
```


#### 策略的管理与获取

在接收到外部系统请求后，会根据不同的通知类型找到对应的处理类，然后由处理类来执行。这边结合 `Spring` 容器来管理，在构造 `NoticeFactory` 时让容器把所有通知处理类注入进来，然后循环转为map方便获取


```java
@Compoent
public class NoticeFactory {

    /**
     * key：通知类型  value：通知处理器
     * 多个key可能对应同一个处理 如：{101: 询价处理器, 102: 询价处理器, 200: 其它...}
     * 具体有几个取决于 {@link INotice#getNoticeType()} 返回了什么通知类型
     */
    public final Map<Integer, INotice<?>> NOTICE_MAP;

    public NoticeFactory(List<INotice<?>> list) {
        // 初始化处理器map
        NOTICE_MAP = Maps.newHashMap();
        list.forEach(handle -> {
            List<NoticeTypeEnum> noticeType = handle.getNoticeType();
            noticeType.forEach(e -> NOTICE_MAP.put(e.getCode(), handle));
        });
    }
    
    public INotice getNotice(Integer noticeType) {
         return NOTICE_MAP.get(noticeType);
    }
}
```

#### 策略使用

上面已经将所有策略类以key：通知类型，value：处理器对象放在了map中，这边就简单的根据类型来获取即可

```java
CallbackParam param = "外部请求信息";
Boolean result = NoticeFactory.getNotice(param.getNoticeType()).notice(param, merchantNo);
```


## 责任链模式


