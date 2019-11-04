
---
title: SpringBoot源碼|深入理解@Import注解
date: 2019-06-15 14:55:37
tags: 源码
categories: SpringBoot 
---

### 深入理解@Import注解


> 查看SpringBoot相关组件的源码，会发现有很多地方运用了@Import注解，这个注解有什么用呢，工作原理是什么，我们可以利用这个@Import来做什么？

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    Class<?>[] value();
}
```

### @Import的使用方式

```java
public class Client {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        ServiceImpl service = applicationContext.getBean(ServiceImpl.class);
        Assert.notNull(service,"service has inject successful");
    }
}
```
#### Configuration
通过@Import将Class加载到AppConfig类中

```java
@Configuration
@Import(value = ServiceImpl.class)
public class AppConfig {
}
```

#### ImportBeanDefinitionRegistrar

```java
public interface ImportBeanDefinitionRegistrar {
    void registerBeanDefinitions(AnnotationMetadata var1, BeanDefinitionRegistry var2);
}
```

自定义BeanDefinitionRegistrar的实现
```java
public class ServiceBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar{
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(ServiceImpl.class);
        //BeanDefinitionRegistry将生成的BeanDefinition注册到容器中去
        beanDefinitionRegistry.registerBeanDefinition("service",builder.getBeanDefinition());
    }
}
```


```java
@Configuration
@Import(value = ServiceBeanDefinitionRegistrar.class)
public class AppConfig {
}
```
#### 通过ImportBeanDefinitionRegistrar实现的Import可以针对Bean注册容器的过程进行自定义

### ImportSelector
告诉容器需要注入哪些类的类名就可以实现Bean的注入

```java
public interface ImportSelector {
    String[] selectImports(AnnotationMetadata var1);
}
```

```java
public class ServiceImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        //annotationMetadata是关于注解的元数据信息,可以根据需要获取注解的信息
        return new String[]{ServiceImpl.class.getName()};
    }
}
```

```java
@Configuration
@Import(value = ServiceImportSelector.class)
public class AppConfig {
}
```

* 总结
获取注解元数据信息，返回一组需要注入类的类名，有框架来生成对应的实体类从而注入到容器中去。
