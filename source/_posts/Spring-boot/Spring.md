---
title: SpringBoot源碼|深入理解@EnbaleXxx
date: 2019-06-16 14:55:37
tags: 源码
categories: SpringBoot 
---

### EnableXXX源码解析

> Spring Boot的核心作用在于他具有强大的自动配置的功能，在Spring框架的基础上利用约定大于配置减少了Spring开发中配置复杂等问题。Spring Boot提供了很多类似于@EnableXXX的注解，这些注解有什么用呢？解决了Spring Boot什么问题？ 接下来我们通过源码来学习一下关@EnableXXX相关注解。

#### 常见的@Enable注解
+ @EnableCaching
+ @EnableAsync
+ @EnbaleJpaRepositories
+ @EnableAutoConfiguration

本文主要以@EnableCaching作为核心来讲解一下SpringBoot关于@EnbaleXXX注解实现

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({CachingConfigurationSelector.class})
public @interface EnableCaching {
    //采用基于JDK的动态代理还是CGLIB
    boolean proxyTargetClass() default false;

    //默认采用动态代理的模式
    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}
```
关于@Import，可以参照博客![@Import注解](http://shawnyou.tech/2019/06/15/Spring-boot/import/#more)

@EnableCaching利用@Import导入了一个CachingConfigurationSelector配置类，

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
    、、、、只保留关键代码

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

CachingConfigurationSelector导入两个关键的动态代理配置类
1. AutoProxyRegistrar 
2. ProxyCachingConfiguration 

#### AutoProxyRegistrar(自动代理构建器)
AutoProxyRegistrar就是一个自动代理注册器，他负责给容器注册了一个InfrastructureAdvisorAutoProxyCreator
#### ProxyCachingConfiguration

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
ProxyCachingConfiguration向容器中注入三个Bean。










