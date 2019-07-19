---
layout:     post
title:      "模板方法模式简要介绍"
subtitle:   "CommonJS，RequireJS，SeaJS 归纳笔记"
date:       2019-07-18
author:     "Ice Silkworm"
header-img: "img/home-bg.jpg"
tags:
    - 服务端开发
    - Java
---



## 模板方法模式简要介绍

> 模板方法模式：定义一个操作中算法的框架，而将一些步骤延迟到子类中。模板方法模式使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。
>
> Template Method Pattern: Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.
>
> 模板方法模式是一种基于继承的代码复用技术，它是一种类行为型模式。
>
> 模板方法模式是结构最简单的行为型设计模式，在其结构中只存在父类与子类之间的继承关系。通过使用模板方法模式，可以将一些复杂流程的实现步骤封装在一系列基本方法中，在抽象父类中提供一个称之为模板方法的方法来定义这些基本方法的执行次序，而通过其子类来覆盖某些步骤，从而使得相同的算法框架可以有不同的执行结果。模板方法模式提供了一个模板方法来定义算法框架，而某些具体步骤的实现可以在其子类中完成。

以上引用自《设计模式Java版》一书的内容，更多内容可参考`GitBook`版本：[设计模式Java版](https://gof.quanke.name/)

## 前序

最近有新的需求过来，要求扩展会员产品体系，而且这次**”又“**和以前的添加规则不同，为什么说是**”又“**呢，因为以前也是这样的情况，而且很多。

按照之前的代码处理方式，无非就是增加 `if` 分支，copy 之前的业务代码（因基本上的整体流程是相同的，只有核心的添加算法逻辑不同），稍微改动添加算法逻辑，这样就添加新业务成功了。但业务是添加成功了，只是代码看起来就很恐怖了，一长串的 `if` 分支，滚动两屏都看不到头。。。

在我接手此次的需求后，我看到的代码就是这样的。大量重复代码，大量分支，梳理逻辑及其困难，遂决定进行重构。

## 需求梳理

三方支付渠道支付成功后，统一进入会员添加入口，进行会员添加逻辑。根据订单中的会员类型信息，进入各自的添加逻辑分支。

会员添加逻辑的具体算法是：计算出会员开始时间、添加或者更新会员信息、记录日志

## 接口抽象

抽象算法模板父类：

```java
/**
 * 抽象会员添加处理方式
 *
 * @author fanshen
 * @date 2019-03-12 11:43
 */
public abstract class AbstractMemberAddHandler {

    /**
     * 添加会员前的业务处理方法，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @author fanshen
     * @date 2019-03-12 16:50
     */
    protected void beforeAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
    }

    /**
     * 添加金牌会员业务方法，由具体子类实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @return com.zhl.rmi.pay.member.pojo.AddMemberTypeResultEntity
     * @author fanshen
     * @date 2019-01-29 17:52
     */
    protected abstract AddMemberTypeResultEntity addMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity);

    /**
     * 添加会员之后，记录会员添加日志，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity  - 添加会员信息请求参数实体
     * @param addMemberTypeResultEntity - 添加会员处理结果信息实体
     * @author fanshen
     * @date 2019-03-12 16:51
     */
    protected abstract void processAddMemberTypeLog(AddMemberTypeParamEntity addMemberTypeParamEntity, AddMemberTypeResultEntity addMemberTypeResultEntity);

    /**
     * 添加会员之后的回调方法，可以处理相关业务，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity  - 添加会员信息请求参数实体
     * @param addMemberTypeResultEntity - 添加会员处理结果信息实体
     * @author fanshen
     * @date 2019-03-12 16:51
     */
    protected void afterAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity, AddMemberTypeResultEntity addMemberTypeResultEntity) {
    }

    /**
     * 处理具体的添加会员信息的业务方法，客户端调用该方法添加会员
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @return com.zhl.rmi.pay.member.pojo.AddMemberTypeResultEntity
     * @author fanshen
     * @date 2019-03-12 16:54
     */
    final public AddMemberTypeResultEntity processAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        this.beforeAddMemberType(addMemberTypeParamEntity);
        AddMemberTypeResultEntity addMemberTypeResultEntity = this.addMemberType(addMemberTypeParamEntity);
        this.processAddMemberTypeLog(addMemberTypeParamEntity, addMemberTypeResultEntity);
        this.afterAddMemberType(addMemberTypeParamEntity, addMemberTypeResultEntity);
        return addMemberTypeResultEntity;
    }
}
```

抽象父类定义了整个添加会员的关键算法顺序：`beforeAddMemberType` ---> `addMemberType` ---> `processAddMemberTypeLog` ---> `afterAddMemberType` 。

父类只对外暴露 `processAddMemberType` 方法，提供添加会员的统一入口；

各个关键算法使用 `protected` 关键字修饰，保证这些方法只能被子类实现，而不能被业务调用方访问和篡改；

---

会员添加辅助支撑类：

```java
/**
 * 添加金牌会员辅助服务类
 *
 * @author fanshen
 * @date 2019-03-12 11:43
 */
@Service
@Slf4j
public class AddMemberTypeSupportHandler extends AbstractMemberAddHandler {

    @Resource
    private MemberInfoDao memberInfoDao;

    /**
     * 添加会员之后，记录会员添加日志，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity  - 添加会员信息请求参数实体
     * @param addMemberTypeResultEntity - 添加会员处理结果信息实体
     * @author fanshen
     * @date 2019-03-12 16:51
     */
    @Override
    protected void processAddMemberTypeLog(AddMemberTypeParamEntity addMemberTypeParamEntity, AddMemberTypeResultEntity addMemberTypeResultEntity) {
        // 添加会员日志
        UserMemberLogEntity memberLogEntity = new UserMemberLogEntity();
        // ...
        // 根据请求参数和添加结果参数信息，设置会员添加日志记录信息，保存
        memberInfoDao.addMemberLog(memberLogEntity);
    }

    /**
     * 判断当前会员是否是已过期
     *
     * @param modelMember - 查询到的会员信息
     * @param currentTime - 当前时间
     * @return boolean
     * @author fanshen
     * @date 2019-03-12 17:50
     */
    boolean hasMemberButExpired(MemberInfoEntity modelMember, int currentTime) {
        // 根据参数判断会员是否过期
        return true;
    }

    /**
     * 判断是否需要基于孩有为有效会员的到期时间作为起点时间
     *
     * @param addMemberTypeParamEntity - 添加金牌会员的请求参数实体
     * @author fanshen
     * @date 2019-03-13 11:33
     */
    void processStartCalculateTime(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        // 根据请求参数，计算会员开始时间
        addMemberTypeParamEntity.setStartTime(startTime);
    }

    /**
     * 在处理完成孩有为的购买逻辑之后，同步处理小学学习APP的会员信息
     *
     * @param addMemberTypeParamEntity - 添加会员请求参数实体
     * @param memberAddHandler         - 会员添加处理器
     * @author fanshen
     * @date 2019-03-13 11:49
     */
    void syncProcessAfterPay(AddMemberTypeParamEntity addMemberTypeParamEntity, AbstractMemberAddHandler memberAddHandler) {
        // do somethings ...
        // 这里主要用于处理一些支付完成之后的通用逻辑
    }

    /**
     * 添加金牌会员业务方法，由具体子类实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @return com.zhl.rmi.pay.member.pojo.AddMemberTypeResultEntity
     * @author fanshen
     * @date 2019-01-29 17:52
     */
    @Override
    protected AddMemberTypeResultEntity addMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        return null;
    }
}
```

会员添加辅助支撑类主要是起到抽取一些共用业务的抽象和封装，还有就是一些小工具方法的声明及实现，供子类去调用。

---

具体会员类型添加逻辑实现子类：

```java
/**
 * 语文会员添加处理方式
 *
 * @author fanshen
 * @date 2019-03-12 11:43
 */
@Service
public class ChineseMemberAddHandler extends AddMemberTypeSupportHandler {

    @Resource
    private GeneralMemberAddHandler generalMemberAddHandler;

    /**
     * 添加会员前的业务处理方法，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @author fanshen
     * @date 2019-03-12 16:50
     */
    @Override
    protected void beforeAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        processStartCalculateTime(addMemberTypeParamEntity);
    }

    /**
     * 添加会员业务方法，由具体子类实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @return com.zhl.rmi.pay.member.pojo.AddMemberTypeResultEntity
     * @author fanshen
     * @date 2019-01-29 17:52
     */
    @Override
    public AddMemberTypeResultEntity addMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        // 这里有一些通用的会员添加逻辑，进行了算法抽取，共用
        return generalMemberAddHandler.addMemberType(addMemberTypeParamEntity);
    }

    /**
     * 添加会员之后的回调方法，可以处理相关业务，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity  - 添加会员信息请求参数实体
     * @param addMemberTypeResultEntity - 添加会员处理结果信息实体
     * @author fanshen
     * @date 2019-03-12 16:51
     */
    @Override
    protected void afterAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity, AddMemberTypeResultEntity addMemberTypeResultEntity) {
        syncProcessMemberAfterPay(addMemberTypeParamEntity, generalMemberAddHandler);
    }
}
```

```java
/**
 * 英语单科金牌会员添加处理方式
 *
 * @author fanshen
 * @date 2019-03-12 11:43
 */
@Service
public class EnglishMemberAddHandler extends AddMemberTypeSupportHandler {

    @Resource
    private MemberInfoDao memberInfoDao;

    /**
     * 添加会员前的业务处理方法，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @author fanshen
     * @date 2019-03-12 16:50
     */
    @Override
    protected void beforeAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        processStartCalculateTimeOnXxxxMemberPay(addMemberTypeParamEntity);
    }

    /**
     * 添加金牌会员业务方法，由具体子类实现
     *
     * @param addMemberTypeParamEntity - 添加会员信息请求参数实体
     * @return com.zhl.rmi.pay.member.pojo.AddMemberTypeResultEntity
     * @author fanshen
     * @date 2019-01-29 17:52
     */
    @Override
    protected AddMemberTypeResultEntity addMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity) {
        // 比较个性化的添加逻辑，需自己实现添加逻辑
        AddMemberTypeResultEntity resultEntity = new AddMemberTypeResultEntity();
        // 设置添加结果信息，返回
        return resultEntity;
    }

    /**
     * 添加会员之后的回调方法，可以处理相关业务，具体实现由子类去实现
     *
     * @param addMemberTypeParamEntity  - 添加会员信息请求参数实体
     * @param addMemberTypeResultEntity - 添加会员处理结果信息实体
     * @author fanshen
     * @date 2019-03-12 16:51
     */
    @Override
    protected void afterAddMemberType(AddMemberTypeParamEntity addMemberTypeParamEntity, AddMemberTypeResultEntity addMemberTypeResultEntity) {
        syncProcessMemberAfterPay(addMemberTypeParamEntity, this);
    }
```

以及更多类型子类实现，这里不再一一列出。

## 业务调用

面向接口编程，第三方调用的时候，就只需要使用接口类进行声明和调用即可。

```java
AddMemberTypeParamEntity addMemberTypeParamEntity = new AddMemberTypeParamEntity();
// do somethings ...
// 设置会员添加请求信息
// 根据会员类型，通过会员添加处理类工厂生成会员添加处理实体对象，处理会员添加逻辑
AddMemberHandlerFactory.buildInstance(modelProduct.getMemberType()).processAddMemberType(addMemberTypeParamEntity);
```

会员添加处理类工厂：

```java
/**
 * 添加金牌会员逻辑处理器工厂
 *
 * @author fanshen
 * @date 2019-03-13 11:56
 */
public class AddMemberHandlerFactory {

    /**
     * 根据不同的会员类型返回不同的会员添加处理器实体
     *
     * @param memberType - 会员类型
     * @return com.zhl.services.strategy.AbstractMemberAddHandler
     * @author fanshen
     * @date 2019-03-13 12:02
     */
    public static AbstractMemberAddHandler buildInstance(int memberType) {
        AbstractMemberAddHandler memberAddHandler = null;
        switch (memberType) {
            case 3:
                // 英语会员
                memberAddHandler = BeanProvider.getBean(EnglishMemberAddHandler.class);
                break;
            case 4:
                // 语文会员
                memberAddHandler = BeanProvider.getBean(ChineseMemberAddHandler.class);
                break;
            default:
                break;
        }
        return memberAddHandler;
    }
}
```

这样就统一了会员添加逻辑的入口，业务调用方只需要简简单单的调用添加会员的方法即可，不用再管其后的实现逻辑了，现在一切都好起来了，^_^