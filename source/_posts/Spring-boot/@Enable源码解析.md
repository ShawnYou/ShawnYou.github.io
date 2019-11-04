---
title: SpringBoot源碼|@EnbaleXxx的实现原理
date: 2019-06-16 14:55:37
tags: 源码
categories: SpringBoot 
---

### 1.EnableXxx实现原理解析

> Spring Boot的核心作用在于他具有强大的自动配置的功能，在Spring框架的基础上利用约定大于配置减少了Spring开发中配置复杂等问题。Spring Boot提供了很多类似于@EnableXXX的注解，这些注解有什么用呢？解决了Spring Boot什么问题？ 接下来我们通过源码来学习一下关@EnableXXX相关注解。

#### 1.1常见的@Enable注解
+ @EnableCaching
+ @EnableAsync
+ @EnbaleJpaRepositories
+ @EnableAutoConfiguration


#### 1.2 @EnableXxxx源码入口
本文主要以@EnableCaching作为核心来讲解一下SpringBoot关于@EnbaleXXX注解实现

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({CachingConfigurationSelector.class})
public @interface EnableCaching {
    //动态代理的两种实现方式（JDK动态代理或者CGLIB）,默认使用JDK动态代理
    boolean proxyTargetClass() default false;

    //代理实现的两种方式（AspectJ或者动态代理），默认使用动态代理
    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}
```
@EnableCaching是Spring Cache的入口，开启了注解式的缓存支持。
1. 设置动态代理的实现方式，默认采用JDK动态代理。
2. 默认采用动态代理的方式实现代理
3. 通过@Import注解导入了一个CachingConfigurationSelector配置类。

关于@Import，可以参照博客[@Import注解](http://shawnyou.tech/2019/06/15/Spring-boot/@Import%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)
接下来根据@Import导入的CachingConfigurationSelector类进行分析

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
    、、、、只保留关键代码

    //根据@EnableCaching注解的信息确定需要导入容器的Bean的类名
    public String[] selectImports(AdviceMode adviceMode) {
        switch(null.$SwitchMap$org$springframework$context$annotation$AdviceMode[adviceMode.ordinal()]) {
        case 1:
            return this.getProxyImports();
        case 2:
            return this.getAspectJImports();
        default:
            return null;
        }
    }

    //获取动态代理的配置Bean.
    private String[] getProxyImports() {
        List<String> result = new ArrayList(3);
        result.add(AutoProxyRegistrar.class.getName());
        result.add(ProxyCachingConfiguration.class.getName());
        if(jsr107Present && jcacheImplPresent) {
            result.add("org.springframework.cache.jcache.config.ProxyJCacheConfiguration");
        }

        return StringUtils.toStringArray(result);
    }

    //对于AspectJ代理模式，不作分析
    private String[] getAspectJImports() {
        List<String> result = new ArrayList(2);
        result.add("org.springframework.cache.aspectj.AspectJCachingConfiguration");
        if(jsr107Present && jcacheImplPresent) {
            result.add("org.springframework.cache.aspectj.AspectJJCacheConfiguration");
        }

        return StringUtils.toStringArray(result);
    }
    、、、只保留关键代码、
}
```

如上，CachingConfigurationSelector的父类AdviceModeImportSelector实现了ImportSelector接口，导入两个关键的动态代理配置类。
1. AutoProxyRegistrar 
AutoProxyRegistrar就是一个自动代理注册器，他负责给容器注册了一个InfrastructureAdvisorAutoProxyCreator，他就是一个后置增强处理器，负责在Bean初始化后通过动态代理生成代理对象，关于AutoProxyRegistrar的解析，这里有专门的博客对AutoProxyRegistrar创建代理的过程进行阐述。
2. ProxyCachingConfiguration 
```java
@Configuration
@Role(2)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {
    public ProxyCachingConfiguration() {
    }

    @Bean(
        name = {"org.springframework.cache.config.internalCacheAdvisor"}
    )
    @Role(2)
    public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
        BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
        advisor.setCacheOperationSource(this.cacheOperationSource());
        advisor.setAdvice(this.cacheInterceptor());
        if(this.enableCaching != null) {
            advisor.setOrder(((Integer)this.enableCaching.getNumber("order")).intValue());
        }

        return advisor;
    }

    @Bean
    @Role(2)
    public CacheOperationSource cacheOperationSource() {
        return new AnnotationCacheOperationSource();
    }

    @Bean
    @Role(2)
    public CacheInterceptor cacheInterceptor() {
        CacheInterceptor interceptor = new CacheInterceptor();
        interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
        interceptor.setCacheOperationSource(this.cacheOperationSource());
        return interceptor;
    }
}
```
ProxyCachingConfiguration向容器中注入了三个Bean，一起组装了一个BeanFactoryCacheOperationSourceAdvisor，它是一个PointAdvisor(切面),整合了PointCut(切点)和Advice(通知)两个模块，确定了在什么方法上(PointCut)执行什么样的缓存操作(Advice)。

#### 小结
@Enable通过@Import注解能够将指定的配置文件或者Bean装配到Spring容器中，完成自动装配的功能。










