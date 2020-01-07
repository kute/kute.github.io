---
published: true
layout: post
title: JavaTips之spring使用构造函数注入依赖配合Lombok无需添加@Autowired注解
category: Java
tags: Java Spring Inject Autowired
time: 2019.08.27 18:21:00
keywords: 
description: JavaTips之spring使用构造函数注入依赖无需添加@Autowired注解

---

在spring bean中注入依赖，当使用 @Autowired时，之前是这样的

```java
@Service
public class MyService {

    @Autowired
    private YourService yourService;
}
```

或者

```java
@Service
public class MyService {

    
    private YourService yourService;
 
    @Autowired
    MyService(YourService yourService) {
        this.yourService = yourService;
    }
}
```


根据[spirng 官方文档描述](https://docs.spring.io/spring/docs/4.3.25.RELEASE/spring-framework-reference/htmlsingle/#beans-autowired-annotation)，从4.3之后使用构造函数时，@Autowired 注解不是必须的

> As of Spring Framework 4.3, an @Autowired annotation on such a constructor is no longer necessary if the target bean only defines one constructor to begin with. However, if several constructors are available, at least one must be annotated to teach the container which one to use.

也就是上述例子可以更改为如下

```java
@Service
public class MyService {
    
    private YourService yourService;
    
    MyService(YourService yourService) {
        this.yourService = yourService;
    }
}
```

配合 Lombok使用更方便

```java
@Service
@RequiredArgsConstructor
public class MyService {
    
    private final YourService yourService;
    
}
```

摘一段sharding jdbc 3.1版本的样例

```java
@Configuration
@EnableConfigurationProperties({
        SpringBootShardingRuleConfigurationProperties.class, SpringBootMasterSlaveRuleConfigurationProperties.class, 
        SpringBootConfigMapConfigurationProperties.class, SpringBootPropertiesConfigurationProperties.class
})
@RequiredArgsConstructor
public class SpringBootConfiguration implements EnvironmentAware {
    
    private final SpringBootShardingRuleConfigurationProperties shardingProperties;
    
    private final SpringBootMasterSlaveRuleConfigurationProperties masterSlaveProperties;
    
    private final SpringBootConfigMapConfigurationProperties configMapProperties;
    
    private final SpringBootPropertiesConfigurationProperties propMapProperties;
    
    private final Map<String, DataSource> dataSourceMap = new LinkedHashMap<>();

    // ...
}
```


