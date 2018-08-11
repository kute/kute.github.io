---
published: true
layout: post
title: Dubbo-Retry超时重试防止数据重复小记
category: Java
tags: Java Dubbo
time: 2018.08.06 18:21:00
keywords: 
description: Dubbo-Retry超时重试防止数据重复小记

---

- <a href="#background">一、背景</a>
- <a href="#schema">二、方案</a>
  - <a href="#config_retry">1、在配置了重试机制的基础上，修改 单个方法的  “重试” 的配置 + （调用方异常捕获 或者 配置降级策略）</a>
    - <a href="#config_retry_1">1.1、服务调用方 添加 parameters 参数，设置 重试次数</a>
    - <a href="#config_retry_2">1.2、设置 集群容错模式</a>
    - <a href="#config_retry_3">1.3、服务降级（consumer)</a>
      - <a href="#config_retry_3_1">a. 为整个Service 配置降级策略</a>
      - <a href="#config_retry_3_2">b. 单独设置某个方法的降级策略</a>
    - <a href="#catch_ex">1.4、调用方（consumer)异常捕获</a>
  - <a href="#flag">2、在服务调用前后添加唯一标识进行判断</a>
    - <a href="#flag_1">2.1、通过redis实现</a>
  - <a href="#other">三、其他</a>
    - <a href="#other_1">1、provider（服务提供方）设置tps以及tps.interval ：控制请求频率</a>
    - <a href="#other_2">2、dubbo异常处理（ExceptionFilter) 顺序</a>
    - <a href="#other_3">3、数据库唯一索引</a>
- <a href="#link">三、参考</a>
    


<a name="background">一、背景</a>

> dubbo 服务调用过程中配置了重试，对于 非幂等性接口 ，由于 网络 或者 服务端处理速度较慢，发生超时，重试导致 接口被多次调用进行业务逻辑处理，发生脏数据等问题

<a name="schema">二、方案</a>

<a name="config_retry">1、在配置了重试机制的基础上，修改 单个方法的  “重试” 的配置 + （调用方异常捕获 或者 配置降级策略）</a>

> dubbo默认未提供 方法 级别 的注解（xml配置是有的），只有 @Service @Reference，重试次数是对 整个service 中的所有方法生效，通过修改某些对于 幂等性 要求较高的“方法”级别的重试配置（如 取消重试，减少重试），避免因重试带来的脏数据问题


问题 1:

对于某些关键服务调用若配置超时不重试，可能引起 数据丢失问题，需要添加 降级

处理：
  如：扔到队列，异步消费，消费时进行数据一致性校验，如 数据 是否真正入库配置 降级措施
  
  
<a name="config_retry_1">1.1、服务调用方 添加 parameters 参数，设置 重试次数</a>

> 如下 ICityService接口服务的方法 findCity 的重试次数 设置为 2，而 此服务的重试次数默认为5。
（类似的对于超时时间 都可以 这么设置：parameters = {"#myMethod#.#property#", "#propertyValue#"}）


    @Reference(interfaceClass = ICityService.class, retries =5, timeout = 5000, parameters = {"findCity.retries", "2"})
    private ICityService cityService;

    public interface ICityService {
        String findCity(String code, long timeOutMillis);
    }

<a name="config_retry_2">1.2、设置 集群容错模式</a>

> 单独设置 非幂等方法的容错模式为：failfast（快速失败），只调用一次，调用超时则立即失败，然后调用方（consumer) 进行异常捕获，提供降级逻辑。

    /**
     * 设置 ICityService 接口 容错模式为 failover，默认重试次数为5，而单独设置 buildCity方法（非幂等）的 容错模式为快速失败
     */
    @Reference(interfaceClass = ICityService.class, cluster = "failover", retries = 5, parameters = {"buildCity.cluster", "failfast"})
    private ICityService cityService;
    
    
<a name="config_retry_3">1.3、服务降级（consumer)</a> 

- 服务降级 是在 业务调用方 实际调用失败后（或者 强制直接走降级）执行 降级策略，以保障服务可用性。
- 服务降级 可以 配置 在整个 Service 上，也可以单独为 Service 的 某个方法 配置降级策略

`基本方法：配置  <dubbo:reference /> 或者 @Reference 的mock 属性， 示例如下`

<a name="config_retry_3_1">a. 为整个Service 配置降级策略</a>

    /**
     * 设置ICityService 服务降级:
     * @see com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker
     * mock属性值：
     *    false: 不降级
     *    true: 服务调用失败后，调用mock服务接口进行降级
     *    default: 服务调用失败后，调用mock服务接口进行降级
     *    forece: 强制 调用mock服务接口进行降级，无论 接口调用是否成功
     * mock服务接口类定义规则：接口+Mock，如 ICityServiceMock，注意 此类的package路径要和接口一致，如果不一致则需要直接在mock参数里指明 此类
     * 
     * 注意这里的配置是在 失败后，再重复调用5次后还失败的情况下，执行 降级策略
     */
    @Reference(interfaceClass = ICityService.class, retries = 5, check = false, mock = "com.kute.service.mock.ICityServiceMock")
    private ICityService cityService;
    
其中降级处理类如下：

    public class ICityServiceMock implements ICityService {
        @Override
        public String findCity(String code, long timeOutMillis) {
            // 自定义 降级策略实现
            return "mock_findCity_" + code;
        }
    
        // other method
    }
    
<a name="config_retry_3_2">b. 单独设置某个方法的降级策略</a>

    /**
     *  如下为 单独 为 ICityService 的 findCity 方法配置降级
     *
     *  对于 想单独为 某个方法设置 降级mock，可以在 parameters 中设置
     *  如下 设置了 findCity 重试次数（不重试，针对 非幂等接口 就这么设置），然后如果失败了就 调用降级mock
     */
    @Reference(interfaceClass = ICityService.class, retries = 5, parameters = {"findCity.mock", "com.kute.service.mock.method.ICityServiceMock", "findCity.retries", "0"})
    private ICityService cityService;
    
这里 对于 findCity 方法的降级处理类 直接用了 ICityServiceMock ，如果我们只是对部分方法 有降级的需求，那么可以提供一个 模板类（适配器），降级处理类继承模板类，然后只实现必要的方法

    @Reference(interfaceClass = ICityService.class, retries = 5, parameters = {"findCity.mock", "com.kute.service.mock.method.ICityServiceFindCityMock", "findCity.retries", "0"})
    private ICityService cityService;
     
    // 模板类
    public class ICityServiceAdapter implements ICityService {
        @Override
        public String findCity(String code, long timeOutMillis) {
            return null;
        }
    
        // other method
    }
     
    // findCity 的降级处理类
    public class ICityServiceFindCityMock extends ICityServiceAdapter {
    
        @Override
        public String findCity(String code, long timeOutMillis) {
            return "mock_findCity_adaptor_" + code;
        }
    }
    
<a name="catch_ex">1.4、调用方（consumer)异常捕获</a>
        
> try{...}catch(..){...}  或者 自定义 切面 或者 filter 处理异常
 对于 异常，参见：其他

<a name="flag">2、在服务调用前后添加唯一标识进行判断</a>


- 客户端：每次进行rpc调用前，生成唯一ID（UUID），传递到服务端
- 服务端：首先判断 以 UUID 为key在redis中是否存在，不存在 则可以执行正常逻辑；若存在，则认为是重试（重复调用）

<a name="flag_1">2.1、通过redis实现</a>

- 客户端：对有调用duubo rpc的方法添加切面，以注解声明的接口类以及调用的方法名 为 key，值为UUID.randomUUID()，存于 RpcContext 发送到 服务端。
- 服务端：通过在对dubbo方法添加切面，判断 redis中是否 存在 以 此UUID为key的缓存，若存在则判定为 重复调用，直接返回，否则 存于redis并设置过期时间

`伪码如下：`

客户端：
     
    String uuid = UUID.randomUUID().toString();
    // key：类名+方法名+uuid
    // value: uuid
    RpcContext.getContext().setAttachment("com.rpc.myDubboService.myMethodInvoke.uuid", uuid)
    // rpc调用
    com.rpc.myDubboService.myMethodInvoke()
    
服务端：

    String uuid = RpcContext.getContext().getAttachment("com.rpc.myDubboService.myMethodInvoke.uuid");
    // 当key不存在时 设置成功
    if(redis.exists(uuid)) {
        // 若 key已存在，即 重试，所以直接返回
        return
    }
    redis.set(uuid, uuid, expire)
    // 执行正常业务逻辑
    myMethodInvoke
    
`代码简单实现：`

为客户端提供的注解：
    
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD})
    @Inherited
    public @interface DubboConsumerBeforeInvoke {

        // 接口类
        Class[] serviceClazz();

        // rpc方法名
        String[] method();

        boolean enabled() default true;
    }
    
切面：

    @Pointcut("@annotation(com.lianjia.sinan.qc.annotation.DubboConsumerBeforeInvoke)")
    public void pointcut() {

    }

    @Before(value = "pointcut() && @annotation(invoke)")
    public void consumerBeforeInvoke(JoinPoint joinPoint, DubboConsumerBeforeInvoke invoke) {

        if (!invoke.enabled()) {
            logger.warn("Dubbo consumerBeforeInvoke annotation[com.lianjia.sinan.qc.annotation.DubboConsumerBeforeInvoke] is not enabled");
            return;
        }

        Class[] serviceClassArray = invoke.serviceClazz();
        String[] methodArray = invoke.method();

        if (serviceClassArray.length == 0 || serviceClassArray.length != methodArray.length) {
            throw new IllegalArgumentException("Dubbo annotation[com.lianjia.sinan.qc.annotation.DubboConsumerBeforeInvoke] need parameter declare");
        }

        int size = serviceClassArray.length;

        // 对 要调用的每个 dubbo 方法 生成 唯一ID
        for (int i = 0; i < size; i++) {
            Class<?> serviceClass = serviceClassArray[i];
            String dubboMethod = methodArray[i];
            String methodKey = KeyUtil.getMethodKey(serviceClass, dubboMethod);

            String uuid = UUID.randomUUID().toString();
            logger.info("Dubbo consumerBeforeInvoke method[{}] setAttachment in context:{}", methodKey, uuid);
            RpcContext.getContext().setAttachment(methodKey, uuid);

        }
    }
     
    private String getMethodKey(Class<?> serviceClass, String methodName) {
        return Joiner.on(".").useForNull("").join(serviceClass.getName(), methodName, ".uuid");
    }
    
客户端使用注解：

    @DubboConsumerBeforeInvoke(serviceClazz = {ICityService.class}, method = {"liveCity"})
    
服务端提供注解：

    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD})
    @Inherited
    public @interface DubboProviderRetryCheck {
        // 根据业务评估 接口完成调用所需的时间
        long expiredMillis() default 3000;
    }
    
切面：

    @Pointcut("@annotation(com.lianjia.sinan.qc.annotation.DubboProviderRetryCheck)")
    public void pointcut() {

    }

    @Before(value = "pointcut() && @annotation(check)")
    public void providerRetryCheck(JoinPoint joinPoint, DubboProviderRetryCheck check) {

        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();

        // dubbo service class (interface)
        Class<?> serviceClass = method.getDeclaringClass();
        long expiredMillis = check.expiredMillis();
        String methodName = method.getName();

        String methodKey = KeyUtil.getMethodKey(serviceClass, methodName);

        logger.info("Dubbo providerRetryCheck parameters, method={}, expiredMillis={}", methodKey, expiredMillis);
        try {
            // uuid
            String uuid = RpcContext.getContext().getAttachment(methodKey);

            if (null != uuid) {

                String redisKey = KeyUtil.getRedisKey(uuid);

                if (cacher.exists(redisKey)) {
                    logger.info("Dubbo providerRetryCheck method[{}] repeat call then return", methodKey);
                    // 若 key 存在，那么认为是 重试（重复调用）
                    throw new DubboRetryException("method[" + methodKey + "] repeat execute");
                }
                // 否则，设置key值并设定过期时间
                cacher.set(redisKey, uuid, Long.valueOf(expiredMillis).intValue());
            }

        } catch (Exception e) {
            logger.error("Dubbo providerRetryCheck exception in [{}]", methodKey, e);
            if ("DubboRetryException".equalsIgnoreCase(e.getClass().getSimpleName())) {
                throw e;
            }
        }

    }

    
服务端使用：

    @DubboProviderRetryCheck(timeOutMillis=6000)
    
<a name="other">三、其他</a>

<a name="other_1">1、provider（服务提供方）设置tps以及tps.interval ：控制请求频率</a>
     
     dubbo: TpsLimitFilter（滑动窗口）

<a name="other_2">2、dubbo异常处理（ExceptionFilter) 顺序</a>

- 如果是checked异常则直接抛出
- 如果是unchecked异常但是在接口上有声明，也会直接抛出
- 如果异常类和接口类在同一jar包里，直接抛出
- 如果是JDK自带的异常，直接抛出
- 如果是Dubbo的异常，直接抛出
- 其余的都包装成RuntimeException然后抛出（避免异常在Client出不能反序列化问题）

<a name="other_3">3、数据库唯一索引</a>

    对于 非幂等 接口，如果可以 借助 数据库唯一索引 保证接口幂等，但是 还是存在 接口调用资源浪费。
 

<a name="link">三、参考</a>

1、[改造Dubbo，使其可以对接口方法进行注解配置](https://my.oschina.net/roccn/blog/871032)

2、[https://my.oschina.net/roccn/blog/871032](https://blog.csdn.net/joeyon1985/article/details/51046605)
