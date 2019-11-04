---
title: SpringCache源码| SpringCache源码总览
date: 2019-06-24 22:51:50
tags: 源码
categories: Spring
---

## 1. 概述
Spring提供了一种基于注解（annotation）的缓存技术，他并不是具体的缓存使用方案，而是一个对缓存使用的抽象方案。只要代码使用相应的缓存注解，就能使用各种定义的缓存操作。由此一来缓存的操作非常的便捷，对代码的侵入比较少，让开发者更关注于业务逻辑的实现。
### 1.1 如何使用Spring Cache

## 2. SpringCache的原理
缓存的操作相对固定，可以借助AOP面向切面编程的思想将缓存操作作为通用的逻辑封装起来。SpringCache借助AOP的思想将缓存操作封装成了通用的模块。接下来通过一个流程图来了解一下Spring Cache缓存模块的原理。

### 2.1 SpringCache的基本接口和注解
#### 2.1.1 Cache和CacheManager
1. Cache
Cache接口是缓存操作的基础接口，它提供了关于缓存的增、删、改、查的相关操作。
```java
public interface Cache {

    String getName();

    //返回原始的缓存对象
    Object getNativeCache();
    //返回包装值
    @Nullable
    Cache.ValueWrapper get(Object var1);
    //获取指定类型的缓存
    @Nullable
    <T> T get(Object var1, @Nullable Class<T> var2);

    @Nullable
    <T> T get(Object var1, Callable<T> var2);

    void put(Object var1, @Nullable Object var2);

    //写入缓存
    @Nullable
    Cache.ValueWrapper putIfAbsent(Object var1, @Nullable Object var2);
    //删除指定key的缓存
    void evict(Object var1);
    //清空缓存
    void clear();

}
```
Cache有两个实现类ConcurrentMapCache和NoOpCache

（1）ConcurrentMapCache
Cache接口的默认实现
（2）NoOpCache
测试用

2. CacheManager
CacheManager是一个用于管理Cache对象的集合。SpringCache提供了四种CacheManager的实现。
![CacheManager继承图](/SpringCache源码解析/cacheManager.jpg)
(1)SimpleCacheManager
简单的CacheManager实现。主要用于测试
(2)NoOpCacheManager
测试用的CacherManager
(3)CompositeCacheManager
组合CacheManager, 可以集成多种CacheManager实例，支持多种缓存容器
(4)ConcurentMapCacheManager
SpringCache默认的CacheManager实现。使用ConcurrentMap作为缓存技术。

这里就SpringCache提供了默认的缓存容器实现方案(ConcurentMapCacheManager)做一个源码分析
```java
public class ConcurrentMapCacheManager implements CacheManager, BeanClassLoaderAware {
    //默认创建大小为16的ConcurrentMap来存储Cache对象
    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap(16);
    //是否可以动态创建缓存
    private boolean dynamic = true;
    //是否允许存null值
    private boolean allowNullValues = true;
    private boolean storeByValue = false;
    @Nullable
    private SerializationDelegate serialization;

    public ConcurrentMapCacheManager() {
    }

    public ConcurrentMapCacheManager(String... cacheNames) {
        this.setCacheNames(Arrays.asList(cacheNames));
    }

    //初始化Cache,初始化过后，dynamic设置为false,不会再动态创建Cache.
    public void setCacheNames(@Nullable Collection<String> cacheNames) {
        if (cacheNames != null) {
            Iterator var2 = cacheNames.iterator();

            while(var2.hasNext()) {
                String name = (String)var2.next();
                this.cacheMap.put(name, this.createConcurrentMapCache(name));
            }

            this.dynamic = false;
        } else {
            this.dynamic = true;
        }

    }

    public boolean isAllowNullValues() {
        return this.allowNullValues;
    }

    //根据name获取Cache实例，如果Cache实例不存在，则动态创建一个缓存Cache实例。
    @Nullable
    public Cache getCache(String name) {
        Cache cache = (Cache)this.cacheMap.get(name);
        //如果初始化过Cache,则getCache的时候不会动态创建Cache.
        if (cache == null && this.dynamic) {
            ConcurrentMap var3 = this.cacheMap;
            synchronized(this.cacheMap) {
                cache = (Cache)this.cacheMap.get(name);
                if (cache == null) {
                    cache = this.createConcurrentMapCache(name);
                    this.cacheMap.put(name, cache);
                }
            }
        }

        return cache;
    }

    //创建ConcurrentMapCache对象
    protected Cache createConcurrentMapCache(String name) {
        SerializationDelegate actualSerialization = this.isStoreByValue() ? this.serialization : null;
        return new ConcurrentMapCache(name, new ConcurrentHashMap(256), this.isAllowNullValues(), actualSerialization);
    }
}
```
#### 小结
Cache接口提供了缓存操作的基础操作(增、删、改、查),CacheManager提供了集中管理Cache对象的功能。两个接口一起提供了缓存操作的基本功能。同时这两个接口也是SpringCache对外提供的扩展点。可以根据需要去扩展想要的缓存对象(Cache)和缓存管理器(CacheManager),假如想用Redis作为缓存容器，则可以自定义RedisCache和RedisCacheManager。

### SpringCache核心注解
SpringCache提供了几个缓存操作的核心注解
1. @Cacheable
```java
//可作用于类和方法
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {
    //cache的名称
    @AliasFor("cacheNames")
    String[] value() default {};
    //cache的名称
    @AliasFor("value")
    String[] cacheNames() default {};
    //缓存值的key,使用SPEL手动指定缓存键的组合方式，默认情况是使用所有参数来组合缓存键
    // #root.method：用于获取当前方法的Method实例
    // #root.target：用于获取当前方法的target实例
    // #root.caches：用于获取当前方法关联的缓存
    // #root.methodName：用于获取当前方法的名称
    // #root.targetClass：用于获取目标类类型
    // #root.args[1]：获取当前方法的第二个参数，等同于：#p1和#a1和#argumentName
    String key() default "";
    //缓存键生成器,功能与key()具有排他性。keyGenerator是一个函数接口，可以通过自定义逻辑来实现key的生成策略
    String keyGenerator() default "";
    //设置自定义的缓存管理器，与缓存解析器具有排他性
    String cacheManager() default "";
    //设置自定义的缓存解析器
    String cacheResolver() default "";
    //执行缓存的条件，条件不满足就不会执行缓存操作
    String condition() default "";
    //禁止缓存功能，设置的条件满足的话
    String unless() default "";
    //设置是否对多个针对同一key执行缓存加载的操作的线程进行同步
    boolean sync() default false;
}
```
作用：
+ 直接从缓存中拿数据，如果缓存未命中，则执行方法，将方法返回的结果存入缓存。
+ @Cacheable可以作用在类上，表示类中的所有方法都使用了@Cacheable。
2. @CachePut
@CachePut参数含义与@Cacheable一样
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CachePut {
    @AliasFor("cacheNames")
    String[] value() default {};

    @AliasFor("value")
    String[] cacheNames() default {};

    String key() default "";

    String keyGenerator() default "";

    String cacheManager() default "";

    String cacheResolver() default "";

    String condition() default "";

    String unless() default "";
}
```
作用：
作用与@Cacheable保持一致

3. @CacheEvict
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CacheEvict {
    @AliasFor("cacheNames")
    String[] value() default {};

    @AliasFor("value")
    String[] cacheNames() default {};

    String key() default "";

    String keyGenerator() default "";

    String cacheManager() default "";

    String cacheResolver() default "";

    String condition() default "";
    //指定当前缓存名称下的所有缓存是否全部删除。
    boolean allEntries() default false;
    //指定删除缓存的操作在方法调用前还是调用后执行，默认False,先用调用方法，后执行缓存删除。
    boolean beforeInvocation() default false;
}
```
作用
+ 从缓存中删除数据
+ allEntries来指定是否删除CacheName中的所有缓存数据
+ beforeInvocation表示缓存操作的执行顺序

### 总结
Spring Cache提供了基础且核心的接口和注解，使我们能够更加便捷的操作缓存
但是我们不禁有个疑问?这些缓存接口和注解是如何作用到我们的方法上的。我们是如何获取到注解信息并通过这些信息指导缓存操作的？下一篇就聊一聊缓存注解的解析和应用。