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

这个功能是从旧的用户系统同步用户到新的SSO系统中，方便已有的系统切换到SSO，以及后续用户信息有变更时能自动同步。同步通过mq的方式，当人员信息有变更时会从旧的用户系统发一条消息出来，消息包含基本信息、公司组织、岗位部门等

面对这样的业务，我们通常会采用面向过程的设计方法将流程拆分成N个步骤，每个步骤执行独立的逻辑：

```java
public void doSync() {
    // 处理基本信息
    
    // 公司组织
    
    // 岗位部门    
}
```

如上代码是不符合开闭原则的，修改其中一个步骤仍然可能影响其他步骤(同一个类修改，不符合开闭原则)。当逻辑复杂后，后续的扩展和维护会容易出错，其他人接手也需要更多的时间。
在这种场景下，有一种通过责任链模式，可以将这些子步骤封装成独立的handler，然后通过pipeline将其串联起来

#### 责任链的定义

常见的责任链模式会设计如下：

![markdown](https://ddmcc-1255635056.file.myqcloud.com/d538733f-7367-465d-82c5-44ef4857d42d.png)


这边使用了自动生成责任链模式代码的工具 `foldright/auto-pipeline` ，在需要生成pipeline的接口上加上 `@AutoPipeline`

```java
/**
 * 同步人员处理
 *
 * @author jiangrz
 * @date 2022-09-28 14:26
 */
@AutoPipeline
public interface Sync {

    /**
     * 同步用户
     *
     * @param param param
     * @return boolean
     * @author jiangrz
     * @date 2022/9/28 14:29
     */
    boolean sync(SyncUserDTO param);
}
```

会自己生成handle接口，名称为pipeline接口名+Handler。即：SyncHandler 

```java
public interface SyncHandler {
    boolean sync(SyncUserDTO param, SyncHandlerContext syncHandlerContext);
}
```

然后自定义handler去实现SyncHandler即可，这里分成了处理基本信息、用户组织、岗位部门、用户默认角色等几个处理类：

```java
/**
 * 用户信息处理逻辑
 *
 * @author jiangrz
 * @date 2022-09-28 14:54
 */
@Slf4j
public class UserInfoHandler implements SyncHandler {


    @Override
    public boolean sync(SyncUserDTO param, SyncHandlerContext syncHandlerContext) {
        // 同步用户信息

        // 下一步
        return syncHandlerContext.sync(param);
    }

}

/**
 * 用户组织同步
 *
 * @author jiangrz
 * @date 2022-09-28 14:54
 */
@Slf4j
public class UserOrgHandler implements SyncHandler {

    @Override
    public boolean sync(SyncUserDTO param, SyncHandlerContext syncHandlerContext) {
        // 同步用户组织信息
        
        // 下一步
        return syncHandlerContext.sync(param);
    }
}

/**
 * 用户部门/岗位同步
 *
 * @author jiangrz
 * @date 2022-09-28 14:54
 */
@Slf4j
public class UserDeptPostHandler implements SyncHandler {

    @Override
    public boolean sync(SyncUserDTO param, SyncHandlerContext syncHandlerContext) {
        
        // 下一步
        return syncHandlerContext.sync(param);
    }
}


/**
 * 赋予用户默认角色
 *
 * @author jiangrz
 * @date 2022-09-28 14:54
 */
@Slf4j
public class DefaultRoleHandler implements SyncHandler {

    @Override
    public boolean sync(SyncUserDTO param, SyncHandlerContext syncHandlerContext) {
        // 用户默认角色逻辑
        // 下一步
        return syncHandlerContext.sync(param);
    }
}
```

#### 责任链使用

```java
 // 用户信息
UserInfoHandler userInfoHandler = new UserInfoHandler(params);
// 用户组织
UserOrgHandler userOrgHandler = new UserOrgHandler(params);
// 用户默认角色
DefaultRoleHandler defaultRoleHandler = new DefaultRoleHandler(params);
// 用户部门岗位
UserDeptPostHandler userDeptPostHandler = new UserDeptPostHandler(params);

SyncPipeline syncPipeline = new SyncPipeline()
        .addLast(userInfoHandler)
        .addLast(userOrgHandler)
        .addLast(defaultRoleHandler)
        .addLast(userDeptPostHandler);
// 开始处理
SyncUserDTO syncUser = "";
syncPipeline.sync(syncUser);
```