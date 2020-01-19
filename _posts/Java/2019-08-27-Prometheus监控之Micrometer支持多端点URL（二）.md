---
published: true
layout: post
title: Prometheus监控之Micrometer支持多端点URL（二）
category: Java
tags: Java Prometheus Micrometer
time: 2020.01.19 18:21:00
keywords: 
description: Prometheus监控之Micrometer支持多端点URL（二）

---

接上一篇文章：[Prometheus监控之Micrometer支持多端点URL](https://kute.github.io/2019/08/27/Prometheus%E7%9B%91%E6%8E%A7%E4%B9%8BMicrometer%E6%94%AF%E6%8C%81%E5%A4%9A%E7%AB%AF%E7%82%B9URL.html
)

前面的多端点URL的创建，现在看也是很鸡肋，不说代码问题，起码做不到 运行时配置（runtime），即 如果能够 请求下API就能自动创建一个URL这样才行。

>  现在又找到两种方式，这两种方式前提 都需要 主动创建 端点Bean等

1、创建端点Bean

> 前面的文章中是通过 `BeanDefinitionRegistryPostProcessor`在服务启动时创建一系列bean，现在需要通过API来创建

只写 主要逻辑

		public void doRegistMetricsUrl(String busiCode) throws Exception {
        log.info("doRegistMetricsUrl param for busiCode = {}", busiCode);

        ConfigurableListableBeanFactory beanFactory = ((ConfigurableApplicationContext) applicationContext).getBeanFactory();

        String beanNamePrefix = CaseFormat.UPPER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, busiCode);

        // 1. prometheus properties bean
        PrometheusProperties prometheusProperties = new PrometheusProperties();
        String prometheusPropertiesBeanName = beanNamePrefix + "PrometheusProperties";
        beanFactory.registerSingleton(prometheusPropertiesBeanName, prometheusProperties);

        PrometheusProperties prometheusPropertiesBean = applicationContext.getBean(prometheusPropertiesBeanName, PrometheusProperties.class);

        // 2. prometheus config bean
        PrometheusPropertiesConfigAdapter propertiesConfigAdapter = new PrometheusPropertiesConfigAdapter(prometheusPropertiesBean);
        String propertiesConfigAdapterBeanName = beanNamePrefix + "PrometheusConfig";
        beanFactory.registerSingleton(propertiesConfigAdapterBeanName, propertiesConfigAdapter);

        PrometheusPropertiesConfigAdapter propertiesConfigAdapterBean = applicationContext.getBean(propertiesConfigAdapterBeanName, PrometheusPropertiesConfigAdapter.class);

        // 3. collector registry bean
        CollectorRegistry collectorRegistry = new CollectorRegistry(true);
        String collectorRegistryBeanName = beanNamePrefix + "CollectorRegistry";
        beanFactory.registerSingleton(collectorRegistryBeanName, collectorRegistry);

        CollectorRegistry collectorRegistryBean = applicationContext.getBean(collectorRegistryBeanName, CollectorRegistry.class);

        // 4. prometheus meter registry bean
        Clock clock = applicationContext.getBean(Clock.class);
        PrometheusMeterRegistry prometheusMeterRegistry = new PrometheusMeterRegistry(propertiesConfigAdapterBean, collectorRegistryBean, clock);
        String prometheusMeterRegistryBeanName = beanNamePrefix + "MeterRegistry";
        beanFactory.registerSingleton(prometheusMeterRegistryBeanName, prometheusMeterRegistry);

        PrometheusMeterRegistry prometheusMeterRegistryBean = applicationContext.getBean(prometheusMeterRegistryBeanName, PrometheusMeterRegistry.class);
        // put meter registry
        beanUtil.addMeterRegistry(busiCode, prometheusMeterRegistryBean);

        // 5. register endpoint bean
        CommonScrapeEndPoint endPoint = new CommonScrapeEndPoint(collectorRegistry);
        String endPointBeanName = beanNamePrefix + "ScrapeEndpoint";
        String endPointId = CaseFormat.UPPER_UNDERSCORE.to(CaseFormat.LOWER_HYPHEN, busiCode).replace("-", "");

        // change endpoint id
        WebEndpoint webEndpoint = endPoint.getClass().getAnnotation(WebEndpoint.class);
        InvocationHandler invocationHandler = Proxy.getInvocationHandler(webEndpoint);
        Field field = invocationHandler.getClass().getDeclaredField("memberValues");
        field.setAccessible(true);
        Map<String, Object> memberValues = (Map<String, Object>) field.get(invocationHandler);
        String id = (String) memberValues.get("id");
        memberValues.put("id", endPointId);

        beanFactory.registerSingleton(endPointBeanName, endPoint);

        log.info("doRegistMetricsUrl done for endPointBeanName={}, endPointId={}, prometheusMeterRegistryBeanName={}, propertiesConfigAdapterBeanName={}, " +
                        "prometheusPropertiesBeanName={}, collectorRegistryBeanName={}, busiCode={}",
                endPointBeanName, endPointId, prometheusMeterRegistryBeanName, propertiesConfigAdapterBeanName, prometheusPropertiesBeanName,
                collectorRegistryBeanName, busiCode);
    }


简单说下上述的逻辑：
1. ibc 表示一个单独的业务，新创建的端点URL也以这个为区分，例如 ibc=CALL_GROUP
2. 之后的 bean注册有很多方式，这里只是一种，各种bean的beanName定义都是  `callGroup{BeanClassType}`的方式，例如  `callGroupMeterRegistry`
3. 只有这些bean创建了才可以进行后续的端点URL暴露

下来是 暴露端点URL的方式，先说方式一

2、反射

通过翻看 `actuator`端点的暴露源码，主要入口有两个：
- org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointAutoConfiguration : 用来定义端点的发现组件，例如有  `WebEndpointDiscoverer`用来发现 `@Endpoint`或者`@WebEndpoint`的端点，`ControllerEndpointDiscoverer`用来发现 `@RestControllerEndpoint`或者`@ControllerEndpoint`的，当然可能还会有`ServletEndpointDiscoverer`
- org.springframework.boot.actuate.autoconfigure.endpoint.web.servlet.WebMvcEndpointManagementContextConfiguration：用来将 所有的端点注册并且暴露

截取一段`WebMvcEndpointHandlerMapping`的源码(boot 2.1.6.RELEASE)

		@Bean
	@ConditionalOnMissingBean
	public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(WebEndpointsSupplier webEndpointsSupplier,
			ServletEndpointsSupplier servletEndpointsSupplier, ControllerEndpointsSupplier controllerEndpointsSupplier,
			EndpointMediaTypes endpointMediaTypes, CorsEndpointProperties corsProperties,
			WebEndpointProperties webEndpointProperties) {
		List<ExposableEndpoint<?>> allEndpoints = new ArrayList<>();
		Collection<ExposableWebEndpoint> webEndpoints = webEndpointsSupplier.getEndpoints();
		allEndpoints.addAll(webEndpoints);
		allEndpoints.addAll(servletEndpointsSupplier.getEndpoints());
		allEndpoints.addAll(controllerEndpointsSupplier.getEndpoints());
		EndpointMapping endpointMapping = new EndpointMapping(webEndpointProperties.getBasePath());
		return new WebMvcEndpointHandlerMapping(endpointMapping, webEndpoints, endpointMediaTypes,
				corsProperties.toCorsConfiguration(),
				new EndpointLinksResolver(allEndpoints, webEndpointProperties.getBasePath()));
	}

从源码里可以看到，端点的类型有三种：webendpoint、servletendpoint、controllerendpoint，也就是`WebEndpointAutoConfiguration`里定义的各种组件

其他逻辑就不一一罗列了，从这俩类跟踪下去（这里主要看 webpoint），可以看到 最后是怎么 映射`EndpointMapping`，怎么最后通过 `WebMvcEndpointHandlerMapping`注册到 `springmvc`将端点URL暴露出来，还有一些`endpointfilter`之类的......

大概是这样的流程：

WebEndpointDiscoverer.getEndpoints获取到所有的端点（主要是EndpointId），然后通过 AbstractWebMvcEndpointHandlerMapping.initHandlerMethods注册端点URL到springmvc

		protected void initHandlerMethods() {
		for (ExposableWebEndpoint endpoint : this.endpoints) {
			for (WebOperation operation : endpoint.getOperations()) {
				registerMappingForOperation(endpoint, operation);
			}
		}
		if (StringUtils.hasText(this.endpointMapping.getPath())) {
			registerLinksMapping();
		}
	}
	
	private void registerMappingForOperation(ExposableWebEndpoint endpoint, WebOperation operation) {
		ServletWebOperation servletWebOperation = wrapServletWebOperation(endpoint, operation,
				new ServletWebOperationAdapter(operation));
		registerMapping(createRequestMappingInfo(operation), new OperationHandler(servletWebOperation),
				this.handleMethod);
	}

那么最后 我们一般可以通过 `/actuator`查看到所有注册的端点URL，这个 `/actuator`也是一个端点，在服务启动时注册，其他的端点URL的展现是通过 `EndpointLinkResolver`处理的，这个是在 初始化`WebMvcEndpointHandlerMapping` bean定义的，如下：

		new EndpointLinksResolver(allEndpoints, webEndpointProperties.getBasePath()))
		
		
所以要想完成 主动创建端点URL，除了刚开始的 `生成端点Bean`之外，还需要注册为`springmvc`服务，脑子一抽就想到要：
- 当我新增了一个端点Bean后，用反射 把 `webEndpointDiscoverer`的`endpoints`属性更改下，添加上刚才我们新建的endpoint，然后重新执行下`服务启动时注册为springmvc服务`的流程
- 从上篇文章可以看到，一个 真正的endpoint bean即 `ExposableWebEndpoint`是由 `endpointId`和`operations`组成的，故需要创建一个  `ExposableWebEndpoint`，需要先创建 `endpointId`和`operations`
- 当有了  `ExposableWebEndpoint`，需要把注册为springmvc的url放到`EndpointLinksResolver`中，这样通过 `/actuator`就能看到我们新创建的端点URL了

具体如下：

		HelloEndpoint helloEndpointBean = applicationContext.getBean(beanName, HelloEndpoint.class);

        WebMvcEndpointHandlerMapping webMvcEndpointHandlerMapping = applicationContext
                .getBean(WebMvcEndpointHandlerMapping.class);

        Collection<ExposableWebEndpoint> unmodifiableEndpoints = Reflect.on(webMvcEndpointHandlerMapping).get("endpoints");
        Collection<ExposableWebEndpoint> endpoints = Lists.newArrayList(unmodifiableEndpoints);

        WebEndpointDiscoverer webEndpointDiscoverer = applicationContext
                .getBean(WebEndpointDiscoverer.class);

        EndpointId endpointId = EndpointId.of(name);


        // create operation
        Reflect operationsFactoryReflect = Reflect.on(webEndpointDiscoverer).field("operationsFactory");
        MultiValueMap<WebOperationRequestPredicate, WebOperation> indexed = new LinkedMultiValueMap<>();
        addOperations(indexed, endpointId, helloEndpointBean, true, operationsFactoryReflect);

        List<WebOperation> operations = indexed.values().stream().map(this::getLast).filter(Objects::nonNull)
                .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));

        // create endpoint bean
        ExposableWebEndpoint exposableWebEndpoint = Reflect.on(webEndpointDiscoverer)
                .call("createEndpoint", helloEndpointBean, endpointId, true, operations).get();

        endpoints.add(exposableWebEndpoint);
    //        Reflect.on(webMvcEndpointHandlerMapping).set("endpoints",    Collections.unmodifiableCollection(Lists.newArrayList(exposableWebEndpoint)));
        Reflect.on(webMvcEndpointHandlerMapping).set("endpoints", Collections.unmodifiableCollection(endpoints));

        EndpointMapping endpointMapping = Reflect.on(webMvcEndpointHandlerMapping).get("endpointMapping");
        Reflect.on(endpointMapping).set("path", "");

        Reflect.on(webMvcEndpointHandlerMapping).set("endpointMapping", endpointMapping);

        Reflect linksResolverReflect = Reflect.on(webMvcEndpointHandlerMapping).field("linksResolver");
        linksResolverReflect.set("endpoints", Collections.unmodifiableCollection(endpoints));

        webMvcEndpointHandlerMapping.afterPropertiesSet();


当我测试完后，可以走通，但是有个问题，虽然在 `/actuator`中看到新创建的URL是`/actuator/xxxxx`，但是实际注册到springmvc的url却是`/xxxxx`，原因也是因为 服务启动时 会注册一个`/actuator`的端点，这个url也是 `basePath`，而上述逻辑里重新走 注册的逻辑，即 `webMvcEndpointHandlerMapping.afterPropertiesSet()`时，还会再注册`/actuator`，就会冲突报错，所以在上面讲`Reflect.on(endpointMapping).set("path", "");`设置为空，就不会注册了，具体见源码，所以就会导致在`org.springframework.boot.actuate.endpoint.web.EndpointMapping#createSubPath`里原本应该为`/actuator/xxxxx`的却变成了`/xxxxx`，虽然URL变了，但是 实际功能是没问题的

3、controller endpoint

> 当我完成上述测试后，发现 其实我们是可以通过`controllerEndpoint`完成这样的功能的，上面的代码就当是熟悉源码好了，具体如下

		@Component
	@RestControllerEndpoint(id = "demo")
	@RequiredArgsConstructor
	@Slf4j
	public class DemoControllerEndpoint {

    private final ApplicationContext applicationContext;

    /**
     *
     * @param ibc
     * such as  CALL_GROUP 、DISPATCH_TICKET etc.
     * @return
     */
    @GetMapping(value = "/{ibc}", produces = TextFormat.CONTENT_TYPE_004)
    public String DemoControllerEndpoint(@PathVariable String ibc) {
        log.info("datagridEndpoint param ibc = {}", ibc);

        CollectorRegistry collectorRegistry = getBusiCollectorRegistry(ibc);
        if(null == collectorRegistry) {
            log.error("datagridEndpoint not found and collectorRegistry for ibc={}", ibc);
            return StringUtils.EMPTY;
        }

        try {
            Writer writer = new StringWriter();
            TextFormat.write004(writer, collectorRegistry.metricFamilySamples());
            return writer.toString();
        } catch (IOException ex) {
            // This actually never happens since StringWriter::write() doesn't throw any
            // IOException
            throw new RuntimeException("Writing metrics failed", ex);
        }
    }

    private CollectorRegistry getBusiCollectorRegistry(String ibc) {
        // such as convert CALL_GROUP to callGroup
        String collectorRegistryBeanName = CaseFormat.UPPER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, ibc) + "CollectorRegistry";
        try{
            return applicationContext.getBean(collectorRegistryBeanName, CollectorRegistry.class);
        } catch(NoSuchBeanDefinitionException e){
            log.error("getBusiCollectorRegistry occur NoSuchBeanDefinitionException for ibc={}, collectorRegistryBeanName={}",
                    ibc, collectorRegistryBeanName);
        }
        return null;
    }

	}
		
然后再配置文件里添加暴露：

		management:
			endpoints:
				web:
					exposure:
						include: ['demo']

这样，我们的端点URL就可以是任意的了，格式为  `/actuator/demo/{ibc}`

菜