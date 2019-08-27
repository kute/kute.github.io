---
published: true
layout: post
title: Prometheus监控之Micrometer支持多端点URL
category: Java
tags: Java Prometheus Micrometer
time: 2019.08.27 18:21:00
keywords: 
description: Prometheus监控之Micrometer支持多端点URL

---

1、背景

> 使用prometheus做监控系统时（java），一般的做法就是系统暴露端点URL给 prometheus，诸如 /metrics ，然后prometheus拉取这个url中的指标数据，
> 主要用到的东西有（spring-boot-starter-actuator, micrometer-registry-prometheus）
> 但是 问题是，默认暴露的端点是  /prometheus，全路径为 /actuator/prometheus，只有一个URL，那么如果有这样的场景该如何：

- prometheus 限制单个URL中的指标数据不能超过 1W
- 若用来做业务指标监控，每个业务方都想用不同的URL暴露指标（应该不会想在一个URL里放所有业务的指标）

那么就需要添加多个URL，该如何做呢？

2、实践

查看micrometer源码（主要是`PrometheusMetricsExportAutoConfiguration`此类)，可以知道默认端点`prometheus`的初始化流程，
也没查到其他公开的API可以方便的添加暴露多个端点URL，那么就可以仿照他的流程自己再写一套这个配置，经实践自己写的配置不会太多，
例如 我想暴露一个 `apple`的端点，即 `/actuator/apple`的URL，那么代码如下：

首先是定义端点

    @WebEndpoint(id = "apple")
    public class AppleScrapeEndPoint {
    
        private final CollectorRegistry collectorRegistry;
    
        public AppleScrapeEndPoint(CollectorRegistry collectorRegistry) {
            this.collectorRegistry = collectorRegistry;
        }
    
        @ReadOperation(produces = TextFormat.CONTENT_TYPE_004)
        public String scrape() {
            try {
                Writer writer = new StringWriter();
                TextFormat.write004(writer, this.collectorRegistry.metricFamilySamples());
                return writer.toString();
            } catch (IOException ex) {
                // This actually never happens since StringWriter::write() doesn't throw any
                // IOException
                throw new RuntimeException("Writing metrics failed", ex);
            }
        }
    }
    
然后是定义`ApplePropertiesConfigAdapter`

    public class ApplePropertiesConfigAdapter extends PropertiesConfigAdapter<PrometheusProperties>
            implements PrometheusConfig {
    
        ApplePropertiesConfigAdapter(PrometheusProperties properties) {
            super(properties);
        }
    
        @Override
        public String get(String key) {
            return null;
        }
    
        @Override
        public boolean descriptions() {
            return get(PrometheusProperties::isDescriptions, PrometheusConfig.super::descriptions);
        }
    
        @Override
        public Duration step() {
            return get(PrometheusProperties::getStep, PrometheusConfig.super::step);
        }
    
    }
    
最后是配置初始化

    @Configuration
    @AutoConfigureAfter(value = {PrometheusMetricsExportAutoConfiguration.class})
    @ConditionalOnClass(value = {PrometheusMeterRegistry.class})
    @ConditionalOnProperty(prefix = "management.metrics.export.apple", name = "enabled", havingValue = "true",
            matchIfMissing = true)
    public class ApplePrometheusAutoConfiguration {
    
        @Bean(name = "applePrometheusProperties")
        @ConfigurationProperties(prefix = "management.metrics.export.apple")
        public PrometheusProperties applePrometheusProperties() {
            return new PrometheusProperties();
        }
    
        @Bean(name = "applePrometheusConfig")
        public PrometheusConfig applePrometheusConfig() {
            return new ApplePropertiesConfigAdapter(applePrometheusProperties());
        }
    
        @Bean(name = "appleMeterRegistry")
        public PrometheusMeterRegistry appleMeterRegistry(Clock clock) {
            return new PrometheusMeterRegistry(applePrometheusConfig(), appleCollectorRegistry(), clock);
        }
    
        @Bean(name = "appleCollectorRegistry")
        public CollectorRegistry appleCollectorRegistry() {
            System.out.println("=======appleCollectorRegistry");
            return new CollectorRegistry(true);
        }
    
        @Configuration
        @ConditionalOnEnabledEndpoint(endpoint = AppleScrapeEndPoint.class)
        public static class TicketScrapeEndpointConfiguration {
    
            @Resource
            private CollectorRegistry appleCollectorRegistry;
    
            @Bean(name = "appleEndpoint")
            @ConditionalOnMissingBean
            public AppleScrapeEndPoint appleEndpoint() {
                return new AppleScrapeEndPoint(appleCollectorRegistry);
            }
    
        }
    
    }
    
然后再配置文件中配置新添加的端点

    management:
      endpoint:
        prometheus:
          # 关闭默认的prometheus端点，新建自己的
          enabled: false
        health:
          show-details: always
      endpoints:
        web:
          exposure:
            include: ['health', 'apple']
            
这样就完成了，就可以在 `/actuator`这个URL中看到自己新添加的URL。
如果想添加多个，那么照着上述copy一份代码，改改名称啥的就可以了，别忘了在配置文件中`include`里添加。

这样基本能解决问题了，但是看着不太舒服，我有多个URL就需要COPY多份这样的代码，而且基本还差不多一样，所以我们可以考虑 主动配置，
减少重复代码的创建，具体如下：

例如我想再添加一个`a`的端点，即 `/actuator/a`


首先是定义端点：

    @Component
    @DatagridEndpoint
    @WebEndpoint(id = "a")
    public class AEndpoint {
    
        private CollectorRegistry collectorRegistry;
        public AEndpoint(){
        }
    
        @ReadOperation(produces = TextFormat.CONTENT_TYPE_004)
        public String scrape() {
            try {
                Writer writer = new StringWriter();
                TextFormat.write004(writer, this.collectorRegistry.metricFamilySamples());
                return writer.toString();
            } catch (IOException ex) {
                // This actually never happens since StringWriter::write() doesn't throw any
                // IOException
                throw new RuntimeException("Writing metrics failed", ex);
            }
        }
    }
    
然后是配置流程

    @Slf4j
    @Component
    @AutoConfigureAfter(value = {PrometheusMetricsExportAutoConfiguration.class})
    @ConditionalOnClass(value = {PrometheusMeterRegistry.class})
    public class MetricsExportAutoConfiguration implements BeanDefinitionRegistryPostProcessor,
            ApplicationContextAware {
    
        private ApplicationContext applicationContext;
    
        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            this.applicationContext = applicationContext;
        }
    
        public class AutoPropertiesConfigAdapter extends PropertiesConfigAdapter<PrometheusProperties>
                implements io.micrometer.prometheus.PrometheusConfig {
    
            AutoPropertiesConfigAdapter(PrometheusProperties properties) {
                super(properties);
            }
    
            @Override
            public String get(String key) {
                return null;
            }
    
            @Override
            public boolean descriptions() {
                return get(PrometheusProperties::isDescriptions, io.micrometer.prometheus.PrometheusConfig.super::descriptions);
            }
    
            @Override
            public Duration step() {
                return get(PrometheusProperties::getStep, PrometheusConfig.super::step);
            }
    
        }
    
        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        }
    
        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) throws BeansException {
    
            Map<String, Object> beansMap = applicationContext.getBeansWithAnnotation(DatagridEndpoint.class);
            if (CollectionUtils.isEmpty(beansMap)) {
                return;
            }
    
            Clock clock = applicationContext.getBean(Clock.class);
            Preconditions.checkNotNull(clock);
    
            for (Map.Entry<String, Object> entry : beansMap.entrySet()) {
                Object bean = entry.getValue();
                WebEndpoint webEndpoint = bean.getClass().getAnnotation(WebEndpoint.class);
                if (null == webEndpoint) {
                    continue;
                }
                String endPointName = webEndpoint.id();
                if (Strings.isNullOrEmpty(endPointName)) {
                    continue;
                }
                // prometheus properties bean
                BeanDefinitionBuilder prometheusPropertiesBeanDefinitionBuilder = BeanDefinitionBuilder
                        .genericBeanDefinition(PrometheusProperties.class);
                BeanDefinition prometheusPropertiesBeanDefinition = prometheusPropertiesBeanDefinitionBuilder.getRawBeanDefinition();
                ((DefaultListableBeanFactory) factory).registerBeanDefinition(endPointName + "PrometheusProperties", prometheusPropertiesBeanDefinition);
    
                PrometheusProperties prometheusProperties = applicationContext.getBean(endPointName + "PrometheusProperties", PrometheusProperties.class);
    
                // prometheus config bean
                BeanDefinitionBuilder prometheusConfigBeanDefinitionBuilder = BeanDefinitionBuilder
                        .genericBeanDefinition(AutoPropertiesConfigAdapter.class, () -> new AutoPropertiesConfigAdapter(prometheusProperties));
                BeanDefinition prometheusConfigBeanDefinition = prometheusConfigBeanDefinitionBuilder.getRawBeanDefinition();
                ((DefaultListableBeanFactory) factory).registerBeanDefinition(endPointName + "PrometheusConfig", prometheusConfigBeanDefinition);
    
                // collector registry bean
                BeanDefinitionBuilder collectorRegistryBeanDefinitionBuilder = BeanDefinitionBuilder
                        .genericBeanDefinition(CollectorRegistry.class, () -> new CollectorRegistry(true));
                BeanDefinition collectorRegistryBeanDefinition = collectorRegistryBeanDefinitionBuilder.getRawBeanDefinition();
                ((DefaultListableBeanFactory) factory).registerBeanDefinition(endPointName + "CollectorRegistry", collectorRegistryBeanDefinition);
    
                PrometheusConfig prometheusConfig = applicationContext.getBean(endPointName + "PrometheusConfig", AutoPropertiesConfigAdapter.class);
                CollectorRegistry collectorRegistry = applicationContext.getBean(endPointName + "CollectorRegistry", CollectorRegistry.class);
    
                // prometheus meter registry bean
                BeanDefinitionBuilder meterRegistryBeanDefinitionBuilder = BeanDefinitionBuilder
                        .genericBeanDefinition(PrometheusMeterRegistry.class, () -> new PrometheusMeterRegistry(prometheusConfig, collectorRegistry, clock));
                BeanDefinition meterRegistryBeanDefinition = meterRegistryBeanDefinitionBuilder.getRawBeanDefinition();
                ((DefaultListableBeanFactory) factory).registerBeanDefinition(endPointName + "MeterRegistry", meterRegistryBeanDefinition);
    
                Reflect.on(bean).set("collectorRegistry", collectorRegistry);
    
            }
        }
    }
    
最后也是在配置文件里`include`添加暴露的端点


    management:
      endpoint:
        prometheus:
          # 关闭默认的prometheus端点，新建自己的
          enabled: false
        health:
          show-details: always
      endpoints:
        web:
          exposure:
            include: ['health', 'apple', 'a']
            
ok，下次如果想添加额外的，那么只需要创建和端点`a`一样的类，改下`id`值，然后再配置文件里`include`里暴露下就可以了，
`MetricsExportAutoConfiguration`这个类会自动创建其他的配置，就不需要重复代码了

哦，对，关于`DatagridEndpoint`这个注解，就只是个简单的注解而已，如下

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface DatagridEndpoint {
    }
    
这样就完成了。
当然，上述只是很潦草的代码，各位可以看着自己改改，更适合自己的项目！

源码见：https://github.com/kute/prometheus-demo/

有问题及时联系