---
published: true
layout: post
title: 利用Dubbo Reference实现HTTP调用执行任何方法
category: Java
tags: Java Dubbo
time: 2020.01.03 18:21:00
keywords: 
description: 利用Dubbo Reference实现HTTP调用执行任何方法

---

- <a href="#background">一、背景</a>
- <a href="#schema">二、方案</a>
  - <a href="#plan_1">1、利用Dubbo Reference实现HTTP2DUBBO</a>
    - <a href="#plan_1_1">主要流程</a>
  - <a href="#plan_2">2、利用Socket通信</a>

<hr />

<a name="background">一、背景</a>

> 首先，我们有个主要使用 Dubbo 实现的RPC服务，以java为主要语言的客户端 调用该服务那很轻松，但是 如果 是其他语言的客户端呢？例如 PHP

<a href="#schema">二、方案</a>

几个方案：

1. 如果 我们的服务端 能够提供HTTP调用，那么 其他语言的客户端 调用也是比较轻松的，也就是 `HTTP2DUBBO`
2. 可以 使用 已经实现的各个语言的客户端库，如 [dubbo2.js ](https://gitee.com/mirrors/dubbo2-js)


这里主要阐述下 HTTP 的方案，主要有下面几种：

1. 如果服务本身能够提供HTTP调用，那么也就无所谓Dubbo了，所以不用考虑 跨语言的问题了
2. 当服务本身 无法或者没有 提供HTTP调用，但是 一般的服务 都会有 `网关` 这一层，网关一般是 提供了HTTP调用的，那么如何 借助 网关来调用我们的DUBBO服务呢？

方案也有几种：

<a href="#plan_1">1、利用Dubbo Reference实现HTTP2DUBBO</a>

<a href="#plan_1_1">1.1、主要流程</a>

1、首先构建一个 服务端服务引用的 library，在这个library中我们新建一个普通的`spring bean service`，名称比如是 `MiddlewareService`

```java
@Service("middlewareService")
public class MiddlewareService implements InitializingBean {

    @Resource("dubboApplication")
    private ApplicationConfig application;
    @Resource("dubboRegistry")
    private RegistryConfig registry;

    /**
         * 动态获取dubbo service
         * @param module
         * @param tClass
         * @param referenceParamMap
         * @param <T>
         * @return
         */
	public <T> T getDubboService(String module, Class<T> tClass, Map<String, String> referenceParamMap) {
            ReferenceConfig<T> reference = new ReferenceConfig<T>();
            reference.setApplication(application);
            reference.setRegistry(registry);
            reference.setInterface(tClass);
            reference.setGroup(module);
            if (null != referenceParamMap && referenceParamMap.size() > 0) {
                reference.setParameters(referenceParamMap);
            }
            ReferenceConfigCache cache = ReferenceConfigCache.getCache();
            return cache.get(reference);
        }
    
    @Override
    public void afterPropertiesSet() throws Exception {
	    
    }
}
```
然后我们再提供一个 我们定制的`dubbo service`，用来作为桥梁来调用真正的服务：

```java
public interface IDispatchService {

    Object dispatch(String beanName, String methodName, Map<String, Object> paramMap) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException;

}
```
实现类
```java

import com.alibaba.dubbo.config.annotation.Service;

/**
 * 注意 这里的 deliverygroup，是需要 在 每个 dubbo 服务的服务端配置的
 */
@Component
@Service(interfaceClass = IDispatchService.class, group = "${deliverygroup}")
public class DispatchService implements IDispatchService {
    
/**
     * @param beanName  我们要执行的beanName
     * @param methodName 我们要执行的method
     * @param paramMap  参数
     * @return
     */
    @Override
    public Object dispatch(String beanName, String methodName, Map<String, Object> paramMap) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {

        Object beanService = ApplicationContextHolder.getBean(beanName);
        if(null == beanService) {
            throw new MiddlewareException("Bean[" + beanName + "] is not found.");
        }

        Method method = ReflectionUtils.findMethod(beanService.getClass(), methodName);
        if(null == method) {
            throw new MiddlewareException("Method[" + methodName + "] of Bean[" + beanName + "] is not exists. ");
        }

        Method sourceMethod = AopUtils.getTargetClass(beanService).getMethod(method.getName(), method.getParameterTypes());

        Object[] params = ParamUtil.resolveArguments(paramMap, sourceMethod);

        Object result = method.invoke(beanService, params);

        return result;
    }
}
```

然后是网关层的http接口

```java
@Component
public class BaseController {
    
    @Autowired
    private MiddlewareService middlewareService;

    public Object feRemoteInvoke(String module, String bean, String method, String custom, String resourceCode, Map<String, String> param) throws Exception {
            IDispatchService dispatchService = null;
            try {
                // 动态获取到我们定制的IDispatchService，module即group
                dispatchService = middlewareService.getDubboService(module, IDispatchService.class);
            } catch (Exception e) {
                // 暂无可用服务
                return null;
            }
            // 通过反射 执行目标方法，当然不仅仅是dubbo方法
            Object result = dispatchService.doDispatch(bean, method, MapUtils.filterEmptyEntry(param), attachMap(custom, resourceCode));
            return result;
        }
}
```

至此 基本就完成了，有几点注意事项：
1. 网关和各个提供dubbo接口的服务端需要 使用 同一个zk地址（如果你用的是zk）
2. 各个服务端都需要配置 deliverygroup 这个配置，且不允许相同，这样同一个 `dubbo service`才会以不同`group`的形式在同一个zk中存在
3. 定制dubbo service即 IDispatchService的引用是在 网关层的
4. 真正通过反射执行目标接口方法是在 各个服务自己内部的，即invoke是自己服务内的

最后以一张图来总结下：

![图片链接](/public/img/posts/java/dubbo-reference.png)

<a href="#plan_2">2、利用Socket通信</a>

1. 可以 加一个任意的注解，在类或者方法上，来标注 想要 暴露的接口方法，这个范围可以自己制定，当然也可以全部暴露
2. 在服务启动的时候，扫描注解获取要暴露的接口方法，并随机生成 `socket port`用来通信并监听，并将 暴露的所有方法签名以及所在类，方法参数，还有socket端口
按一定数据结构存储到 zk上，这个结构可以分成不同层级节点，比如 一个类作为 一个节点，这个节点下 有诸多子节点，乃是 该类下的方法还有参数，也就是方法签名
3. 客户端调用指定 接口方法时，服务端可以根据要调用的接口参数签名等去zk查询节点，当然也可以查到socket端口，
然后 通过socket通信来发送接口参数，来达到 接口调用的效果
4. 当然也可以起一个admin模块，负责把所有暴露的接口从zk上查找到，并输出在 一个友好的页面上，什么类，什么方法以及参数，我们可以手动执行该接口

这部分就不贴代码了~

缺点：随机生成socket port不太好管理

<hr />

上面两种方式可以达到的效果：

> 可以通过HTTP调用执行任意方法，不仅仅是Dubbo接口，适用场景有很多，
> 比如 我们写了一些`CacheService`，里面有一些清缓存的方法，我们可以在需要时手动执行 该接口清除。
> 而且 `HTTP2DUBBO`在某些场景下是 非常有用的，比如我们的前端页面 就可以直接通过 `网关的HTTP2DUBBO`能力，
> 达到 前端直接调用dubbo接口的目的，哦，不对，是达到直接调用 任意接口方法的目的

<hr />

语言组织不太好，但是思路都是切实可行的~