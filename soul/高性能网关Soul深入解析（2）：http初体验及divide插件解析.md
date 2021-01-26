## Soul Http初体验

书接上回，我们run起来了Soul的控制台和网关，把基础设施搞起来了，也了解了Soul的模块，有了对Soul最初的认识。

秦老师讲过学习大型开源项目代码可以分为几个步骤，个人觉得非常值得借鉴，同时结合自己看源码的一些经验总结如下：

- 看介绍，跑demo
- 看入门的guide，自己写一个demo
- 将功能集成到自己的应用中，学会如何使用基本功能并学习一些高级特性
- 总结项目的优点与不足，加入自己的思考
- 捋出主线，沿着主线看源码，边看边debug；大胆假设，小心求证
- 不要纠结细节，先放眼全局
- 修改源码，自己实现，我可以了

今天我们就使用这个思路，学习Soul的http-demo和其中涉及的相关知识点。

## Run Demo

首先，我们把demo run起来，run soul-examples-http的SoulTestHttpApplication即可。

启动日志如下：

![image-20210116013311583](D:\code\blog\images\soul-2\http-log.jpeg)

通过日志可以看出，测试应用基于netty框架，监听了8188端口。同时看到有五条http client注册成功的日志，上下文根分别是/http/test和/http/order。

通过观察目录结构，controller包下面有两个Controller，一个是HttpTestController，一个是OrderController。进入代码后发现，两个controller分别和两个上下文根相对应。

同时我们打开控制台，打开divide插件页面

![image-20210116013859870](D:\code\blog\images\soul-2\admin-divide.jpeg)

可以发现，日志里的注册成功的信息静静的躺在控制台里。我们就不禁有了第一个疑问，注册的行为是怎么发生的呢？

观察这个页面，我们发现分为两部分，一部分是selectorList，第二部分是rulesList。我理解selector是断言加路由的基本信息，用来判断请求是否走当前selector，同时还支持配置多个host用来进行负载均衡。而rulesList里面的规则则代表，在当前selector下面，什么样的url可以匹配对应接口。在请求流程中，selector可以选出host，而rule可以匹配出url，两者相加就得到了最终的调用url。

通过RestClient执行请求

```java
GET http://localhost:9195/findById?id=sss
Accept: */*
Cache-Control: no-cache
```

在网关日志报错

```java
2021-01-16 00:36:16.934 ERROR 9632 --- [-work-threads-1] o.d.soul.plugin.base.utils.CheckUtils    : can not match selector data: divide
```

可以看出，没有匹配到selector。因为selector定义的断言是url要匹配/http/**

我们将请求更换为以下形式

```java
GET http://localhost:9195/http/order/findById?id=sss
Accept: */*
Cache-Control: no-cache
```

得到以下结果

```json
{"id":"sss","name":"hello world findById"}
```

说明调用成功。

在selector界面，可以添加多条configuration。我们另外添加了一条记录（ip不存在），权重均为50.

![image-20210116021108823](D:\code\blog\images\soul-2\selector-configuration.jpeg)

通过多次测试，发现接近一半的几率请求调用不成功，说明负载均衡机制成功了。

观察整个页面，还有几个配置值得注意。一个是type，有两个取值，custom和full，含义为自定义或全局的意思。MatchType支持and和or两种形式，即代表多个conditions如何匹配。这个设计就比SCG要好很多，SCG就是简单粗暴的AND形式，不是很灵活，但是应用可以通过自己扩展GatewayFilter完成需求。order代表当前selector在当前plugin的所有selector中的排序，通过order可以得到一个OrderedSelectorList，保证selector按照业务需求顺序执行。

rule界面如下：

![image-20210116022014539](D:\code\blog\images\soul-2\rules-configuration.jpeg)

可以看到，当我们修改selector和rule数据的时候，网关可以快速拿到数据并更新到内存，这里的实现机制是我们的第二个疑问。

还有，请求在网关中的调用流程中，divide插件是如何完成匹配、拼接url并调用后端服务的，这是第三个疑问。

## 源码跟踪

### 后端服务启动，如何动态注册当前服务信息？

通过对配置文件的观察，我们可以看到配置了控制台的信息和selector的数据

```yml
soul:
  http:
    adminUrl: http://localhost:9095
    port: 8188
    contextPath: /http
    appName: http
    full: false
```

通过我们对starter机制的了解，可以知道以上配置对应着一个Config类。找到soul-spring-boot-starter-client-springmvc模块中的SoulSpringMvcClientConfiguration，其中有三个方法

```java
/**
     * Spring http client bean post processor spring http client bean post processor.
     *
     * @param soulSpringMvcConfig the soul http config
     * @return the spring http client bean post processor
     */
    @Bean
    public SpringMvcClientBeanPostProcessor springHttpClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        return new SpringMvcClientBeanPostProcessor(soulSpringMvcConfig);
    }
    
    /**
     * Context register listener context register listener.
     *
     * @param soulSpringMvcConfig the soul spring mvc config
     * @return the context register listener
     */
    @Bean
    public ContextRegisterListener contextRegisterListener(final SoulSpringMvcConfig soulSpringMvcConfig) {
        return new ContextRegisterListener(soulSpringMvcConfig);
    }
    
    /**
     * Soul http config soul http config.
     *
     * @return the soul http config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul.http")
    public SoulSpringMvcConfig soulHttpConfig() {
        return new SoulSpringMvcConfig();
    }
```

soulHttpConfig方法读取soul.http配置，并将之填到new出来的SoulSpringMvcConfig。SoulSpringMvcConfig对象里面放的就是控制台配置。

springHttpClientBeanPostProcessor方法使用注入的SoulSpringMvcConfig生成了一个SpringMvcClientBeanPostProcessor。

```
/**
     * Instantiates a new Soul client bean post processor.
     *
     * @param soulSpringMvcConfig the soul spring mvc config
     */
    public SpringMvcClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        ValidateUtils.validate(soulSpringMvcConfig);
        this.soulSpringMvcConfig = soulSpringMvcConfig;
        url = soulSpringMvcConfig.getAdminUrl() + "/soul-client/springmvc-register";
        executorService = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
    }

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }
```

构造方法里，SpringMvcClientBeanPostProcessor拿到了config定义的admin地址，拼起来得到了控制台上springmvc的注册接口。SpringMvcClientBeanPostProcessor实现了BeanPostProcessor接口的postProcessAfterInitialization方法，这个方法会在所有bean的初始化之后被调用。方法中执行的是真正的将当前信息注册到控制台的行为，本方法找到了本应用中定义的所有接口中被SoulSpringMvcClient注解标注的接口，并注册到控制台。这样就完成了接口注册的行为。

这里我们只看到了注册rule的行为，没有看到注册selector的行为。通过对RegisterUtil方法的debug发现按，确实没有注册过selector。因此我们去admin的接口中搜索下是否有答案

```java
@PostMapping("/springmvc-register")
    public String registerSpringMvc(@RequestBody final SpringMvcRegisterDTO springMvcRegisterDTO) {
        return soulClientRegisterService.registerSpringMvc(springMvcRegisterDTO);
    }
```

```java
@Override
    @Transactional
    public String registerSpringMvc(final SpringMvcRegisterDTO dto) {
        if (dto.isRegisterMetaData()) {
            MetaDataDO exist = metaDataMapper.findByPath(dto.getPath());
            if (Objects.isNull(exist)) {
                saveSpringMvcMetaData(dto);
            }
        }
        String selectorId = handlerSpringMvcSelector(dto);
        handlerSpringMvcRule(selectorId, dto);
        return SoulResultMessage.SUCCESS;
    }
```

在service方法中我们看到，在注册rule之前，admin通过对rule信息处理注册了selector。在handlerSpringMvcSelector方法里，我们可以看到还对多server注册做了处理，这样可以在server维度上完成注册。

### admin上的数据同步如何同步到网关上

通过对admin页面操作的更新请求的监控，selector和rule数据的更新走了selector和rule走了两个接口。

先看selector

```
@PutMapping("/{id}")
    public SoulAdminResult updateSelector(@PathVariable("id") final String id, @RequestBody final SelectorDTO selectorDTO) {
        Objects.requireNonNull(selectorDTO);
        selectorDTO.setId(id);
        Integer updateCount = selectorService.createOrUpdate(selectorDTO);
        return SoulAdminResult.success(SoulResultMessage.UPDATE_SUCCESS, updateCount);
    }
```

selector接口中有更新操作的方法。可以看到调用了service的createOrUpdate方法，可以看到同时支持创建和更新两种操作。

```
	@Override
    @Transactional(rollbackFor = RuntimeException.class)
    public int createOrUpdate(final SelectorDTO selectorDTO) {
        int selectorCount;
        SelectorDO selectorDO = SelectorDO.buildSelectorDO(selectorDTO);
        List<SelectorConditionDTO> selectorConditionDTOs = selectorDTO.getSelectorConditions();
        if (StringUtils.isEmpty(selectorDTO.getId())) {
            // create
        } else {
            // update
        }
        publishEvent(selectorDO, selectorConditionDTOs);
        return selectorCount;
    }
    
    private void publishEvent(final SelectorDO selectorDO, final List<SelectorConditionDTO> selectorConditionDTOs) {
        PluginDO pluginDO = pluginMapper.selectById(selectorDO.getPluginId());
        List<ConditionData> conditionDataList =
                selectorConditionDTOs.stream().map(ConditionTransfer.INSTANCE::mapToSelectorDTO).collect(Collectors.toList());
        // publish change event.
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE,
                Collections.singletonList(SelectorDO.transFrom(selectorDO, pluginDO.getName(), conditionDataList))));
    }
```

数据库持久化selector数据之后，通过spring的eventPublisher发布了selector的事件，事件类型是DataChangedEvent。

然后我们就要去看，监听DataChangedEvent的listener在哪里。非常巧，我们在admin下面看到了listener包，其中有个抽象类AbstractDataChangedListener，它的实现类只有HttpLongPollingDataChangedListener。从名字上可以看出，我们找的就是它。

DataChangedListener是所有监听器的最上层接口，抽象定义了所有的数据更新方法。AbstractDataChangedListener实现了DataChangedListener的onSelectorChanged方法，定义了更新流程，这里是模板模式。HttpLongPollingDataChangedListener只需要实现afterSelectorChanged方法即可。

```
	@Override
    protected void afterSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.SELECTOR));
    }
    
    class DataChangeTask implements Runnable {
        @Override
        public void run() {
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
                log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
            }
        }
    }
```

HttpLongPollingDataChangedListener实际上调用了DataChangeTask线程，将数据更新结果通过ws发送给了监听本应用的所有网关。

至此，我们理清了网关http数据同步的流程。撒花！

### divide插件的请求调用原理

通过上面的接口数据注册和数据同步两步操作，我们看到数据是如何持久化并同步到网关中的。网关的基础数据此时都已准备妥当，我们现在可以看下请求进入网关之后是如何调用divide插件并分发的。

网关自身的框架包都放到了soul-web包下边，先看SoulConfiguration类

```java
@Configuration
@ComponentScan("org.dromara.soul")
@Import(value = {ErrorHandlerConfiguration.class, SoulExtConfiguration.class, SpringExtConfiguration.class})
@Slf4j
public class SoulConfiguration {
    
    /**
     * Init SoulWebHandler.
     *
     * @param plugins this plugins is All impl SoulPlugin.
     * @return {@linkplain SoulWebHandler}
     */
    @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
    
    /**
     * init dispatch handler.
     *
     * @return {@link DispatcherHandler}.
     */
    @Bean("dispatcherHandler")
    public DispatcherHandler dispatcherHandler() {
        return new DispatcherHandler();
    }
}
```

SoulConfiguration是定义webflux框架下基础组件的配置类，这里我们选取了SoulWebHandler和DispatcherHandler的配置。

其中DispatcherHandler用来请求分发，类似Spring MVC里面的dispatchServlet。SoulWebHandler的注入参数是plugin列表，可见soulWebHandler是用来完成plugin执行的。

我们看下SoulWebHandler：

```java
	@Override
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler)
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }

    private static class DefaultSoulPluginChain implements SoulPluginChain {

        private int index;

        private final List<SoulPlugin> plugins;

        /**
         * Instantiates a new Default soul plugin chain.
         *
         * @param plugins the plugins
         */
        DefaultSoulPluginChain(final List<SoulPlugin> plugins) {
            this.plugins = plugins;
        }

        /**
         * Delegate to the next {@code WebFilter} in the chain.
         *
         * @param exchange the current server exchange
         * @return {@code Mono<Void>} to indicate when request handling is complete
         */
        @Override
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
    }
```

这里可以看出，Soul也是采用调用链模式完成上下文信息传递和plugin调用的。这里的设计和Spring Cloud Gateway相同。在这里说两句题外话，个人觉得Soul和Spring Cloud Gateway是部分同源的。Soul完全没有route的概念，而是将route进行拆分。同时，网关本身的业务配置全都放到了admin上，而SCG是通过java代码或config文件配置的。这里立个flag，Soul源码解析结束后，写一篇结合实际应用场景比较soul和SCG的文章，应该会非常有意思。

我们现在直接跳到divide插件。所有插件都继承了抽象类AbstractSoulPlugin。

```
@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```

在父类中，先通过BaseDataCache获取插件数据，通过插件id去缓存里查selector数据和rule数据。

根据admin中定义的众多selector数据查询是否有满足条件的selector：

```
private SelectorData matchSelector(final ServerWebExchange exchange, final Collection<SelectorData> selectors) {
        return selectors.stream()
                .filter(selector -> selector.getEnabled() && filterSelector(selector, exchange))
                .findFirst().orElse(null);
    }

    private Boolean filterSelector(final SelectorData selector, final ServerWebExchange exchange) {
        if (selector.getType() == SelectorTypeEnum.CUSTOM_FLOW.getCode()) {
            if (CollectionUtils.isEmpty(selector.getConditionList())) {
                return false;
            }
            return MatchStrategyUtils.match(selector.getMatchMode(), selector.getConditionList(), exchange);
        }
        return true;
    }
```

rule也通过类似的形式获取，当通过规则获取到对应的selector和data之后传给divide plugin去做操作。

```
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        //获取当前上下文中后端服务地址ip
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
        //通过负载均衡机制获取selector定义的ip，负载均衡器通过SPI引入
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url
        String domain = buildDomain(divideUpstream);
        //构建真正的目标url并存到上下文中
        String realURL = buildRealURL(domain, soulContext, exchange);
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        // set the http timeout
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }
```

这里可以看出divide插件是如何将已有的selector和rule数据拼成真正的url的，详见注释。这里的代码比较简单，不做赘述。divide插件后面还有webclient等插件去完成调用等流程。That's all.



## 总结

本篇虽然是关于http-demo和divide插件的解析，但是有很多题外话，请见谅

- http服务启动的注册流程
- 控制台数据是如何通过http方式同步给网关的
- divide插件如何进行请求匹配和url构建的
- flag，写一篇关于soul和spring cloud gateway的比较

