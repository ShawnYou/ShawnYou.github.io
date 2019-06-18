### EnableXXX源码解析

Spring Boot的核心作用在于他具有强大的自动配置的功能，在Spring框架的基础上利用约定大于配置减少了Spring开发中配置复杂等问题。

Spring Boot提供了很多类似于@EnableXXX的注解，这些注解有什么用呢？解决了Spring Boot什么问题？ 接下来我们通过源码来学习一下关@EnableXXX相关注解。

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
    boolean proxyTargetClass() default false;

    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}
```
关于@Import，可以参照博客（）

@EnableCaching利用@Import导入了一个配置类，也就是

