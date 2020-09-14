--- 
bg: 'spring-code.jpeg'
layout: post
title:  "@Transactional使用注意事项"
crawlertitle: "@Transactional"
summary: ""
date:   2020-09-14 20:00:00 +0800
categories: posts
tags: 'spring'
author: 宋天
---

在业务中使用到了SpringBoot和Mysql，要完成一个事务操作，很容易想到就直接用@Transactional这个注解，非常简单，在想要执行的事务的方法上注解就可以了。


### @Transactional

这个注解使用起来很简单，但是有很多需要注意的点，最常见的应该是两个，传播级别和隔离级别，隔离级别就是MySQL的四种隔离级别，如果没有就使用MySQL默认的隔离级别，传播级别指的是当Java中的带有@Transactional注解的方法相互调用时是以怎么样的方式去处理，默认是Required，就是如果调用方法开启了事务就加入此事务，否则自己开启新的事务

### 注意

**问题**：按照上面对这个注解的理解，我在UserService实现类中写了多个带有@Transactional注解的方法，测试单独的方法，成功了开启了事务，并能成功提交或回滚，而当我测试某个带有@Transactional方法A调用另外一个带有@Transactional方法B时，B并没有加入A, 而是直接提交了事务

**原因**：经过网上的查找答案，发现这个问题还是很多[文章](https://stackoverflow.com/questions/3423972/spring-transaction-method-call-by-the-method-within-the-same-class-does-not-wo)提到过的，在同一个类内，两个带有@Transactinoal的方法相互调用的被调用方的注解不会生效，原因是因为Spring Aop的一些限制，方法内调用的话没有经过Spring代理，自然也不会有Aop做事务的一些操作，并提到了一些解决方法

**解决方法**：
    
1. 使用AspectJ去处理transactions, 将会生效，因为我这边对SpringAop和AspectJ以及内部具体如何实现代理不怎么熟，所以我没有这么做。 
    
2. 注入UserService到UserService, 就是自己注入自己，然后调用时用此属性的方法，而不是直接用this的方法，这样解决了Spring代理不到的问题，这个自己注入自己的功能是Spring4提供的，我用的SpringBoot2应该已经是Spring5以上了，是可以生效，我测试时，任然没有生效（后来测试时可以生效的，没生效的原因是第二个问题），于是我试了第三种方法。
    
3. 最笨的办法，将调用的方法和被调用的方法分离到两个类，但我测试还是没生效



**问题**：如上，两个类带有@Transactional的方法调用时，被调用方的@Transactional不加入事务

**原因**：经过断点调试，在调用栈中发现了一个问题，看到TransactionManager是MongoTransactionManager, 这是一个MySQL事务，怎么是一个MongoDB的事务管理器，我以为@Transactional会用默认事务管理器，后来发现我在common包中曾经给MongoDB配置一个MongoTransactionManager，SpringBoot就直接用它了，而且再单个方法MySQL事务它居然是可以使用的，多个方法就有问题了。

**解决方法**：找到了原因，现在要做的就是将事务管理变为Mysql可以使用的，因为我可能会使用到MongoDB和MySQL, 怎么才能让@Transactional都可以使用，经过查询，@Transactional注解是可以指定事务管理器的，而且可以使用下面的代码配置默认事务管理器, 需要引入jdbc jar包 修改之后，终于生效，然后测试上面问题的第二种方法，自己注入自己，也生效，问题解决。

java代码

```java
@EnableTransactionManagement // 开启注解事务管理，等同于xml配置文件中的 <tx:annotation-driven />
@Configuration
public class TransactionManagerConfig implements TransactionManagementConfigurer {

    @Resource(name="dataSourceTxManager")
    private PlatformTransactionManager dataSourceTxManager;

    // 创建事务管理器
    @Bean(name = "dataSourceTxManager")
    public PlatformTransactionManager txManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    // 实现接口 TransactionManagementConfigurer 方法，其返回值代表在拥有多个事务管理器的情况下默认使用的事务管理器
    @Override
    public PlatformTransactionManager annotationDrivenTransactionManager() {
        return dataSourceTxManager;
    }

}


```

pom文件

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```




## 结论

1. @Transactional同一个类使用this调用内部方法不生效
2. @Transactional需要配置合适的事务管理器

## 追加

后来在阅读别人文档的过程中，看到别人说有方法A没使用@Transactional而且同时方法B声明了@Transactional, A和B在同一个类内，A调用B时B的事务才不会生效，可是我记得我的A方法和B方法都是有@Transactional的，但为什么也没有生效，看到之前的记录想到应该是因为事务管理器配置错误的原因，对此一直理解错了，导致在这一段时间内很多开发都有影响

同时查阅了以下场景会使@Transaction不生效
1. 有一个类Test，它的一个方法A，A再调用本类的方法B（不论方法B是用public还是private修饰），但方法A没有声明注解事务，而B方法有。则外部调用方法A之后，方法B的事务是不会起作用的
2. 异常被catch捕获导致@Transactional失效
3. @Transactional 应用在非 public 修饰的方法上
4. @Transactional 注解属性 propagation 设置错误
5. @Transactional 注解属性 rollbackFor 设置错误
6. 数据库引擎不支持事务

第一种和第三种应该是最容易被忽略的，第六种的像我们同时使用Mongo和mysql可能会出现这种情况，第二种如果理解的@Transaction回滚的机制就不会犯错，其他对于自己设置的属性应该还是自己理解才会去设置

## 追加2
在之前的开发中一直是知道`throw new RuntimeException()`是可以让@Transactional回滚的，但@Transactional默认只在收到Error和RuntimeException才会回滚，其他的一些受检异常是不错回滚的，例如IOException, 当需要这些异常也触发回滚时可以设置rollbackFor参数

## 参考

[1] [SpringBoot中@Transactional注解的使用总结
](https://blog.csdn.net/WP081600/article/details/93073727)

[2] [同一个类的@Transactional注解不起作用](https://stackoverflow.com/questions/3423972/spring-transaction-method-call-by-the-method-within-the-same-class-does-not-wo)