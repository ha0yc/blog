## Soul：sentinel熔断插件原理分析

Soul网关不仅仅要对后端服务提供代理转发和灰度等基本功能，还要在网关层完成限流和熔断功能。

### Sentinel配置

首先开启sentinel插件，之后如图所示配置selector和rule数据，这里url配置的都是/http/order/findById。

![image-20210130004648514](D:\code\blog\soul\images\hystrix-15\14-1.jpeg)

![image-20210203225537042](images\sentinel\sentinel-1.jpeg)

Sentinel规则的参数设置为，开启熔断规则，通过异常比例判断是否熔断。

通过client发送请求，可以看到经过了Sentinel逻辑。经过压测，1000tps，100000次请求可以看到熔断器被开启。

### Sentinel插件熔断原理

通过观察调用链，请求进来会首先经过GlobalPlugin。GlobalPlugin根据请求信息生成SoulContext，生成时DefaultSoulContextBuilder会截短url，生成realUrl和其他http相关信息放到SoulContext中。

```java
public class GlobalPlugin implements SoulPlugin {
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
}
```

```java
public class DefaultSoulContextBuilder implements SoulContextBuilder {
    
    @Override
    public SoulContext build(final ServerWebExchange exchange) {
        final ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        MetaData metaData = MetaDataCache.getInstance().obtain(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            exchange.getAttributes().put(Constants.META_DATA, metaData);
        }
        return transform(request, metaData);
    }
    
    private SoulContext transform(final ServerHttpRequest request, final MetaData metaData) {
        final String appKey = request.getHeaders().getFirst(Constants.APP_KEY);
        final String sign = request.getHeaders().getFirst(Constants.SIGN);
        final String timestamp = request.getHeaders().getFirst(Constants.TIMESTAMP);
        SoulContext soulContext = new SoulContext();
        String path = request.getURI().getPath();
        soulContext.setPath(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            if (RpcTypeEnum.SPRING_CLOUD.getName().equals(metaData.getRpcType())) {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(metaData.getRpcType());
            } else if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
                setSoulContextByDubbo(soulContext, metaData);
            } else if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
                setSoulContextBySofa(soulContext, metaData);
            } else if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
                setSoulContextByTars(soulContext, metaData);
            } else {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
            }
        } else {
            setSoulContextByHttp(soulContext, path);
            soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
        }
        soulContext.setAppKey(appKey);
        soulContext.setSign(sign);
        soulContext.setTimestamp(timestamp);
        soulContext.setStartDateTime(LocalDateTime.now());
        Optional.ofNullable(request.getMethod()).ifPresent(httpMethod -> soulContext.setHttpMethod(httpMethod.name()));
        return soulContext;
    }
    
    private void setSoulContextByHttp(final SoulContext soulContext, final String path) {
        String contextPath = "/";
        String[] splitList = StringUtils.split(path, "/");
        if (splitList.length != 0) {
            contextPath = contextPath.concat(splitList[0]);
        }
        String realUrl = path.substring(contextPath.length());
        soulContext.setContextPath(contextPath);
        soulContext.setModule(contextPath);
        soulContext.setMethod(realUrl);
        soulContext.setRealUrl(realUrl);
    }
}
```

在AbstractSoulPlugin中获取到selector和rule信息后，调用Sentinel插件的执行方法。

```java
	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        String resourceName = SentinelRuleHandle.getResourceName(rule);
        SentinelHandle sentinelHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SentinelHandle.class);
        return chain.execute(exchange).transform(new SentinelReactorTransformer<>(resourceName)).doOnSuccess(v -> {
            if (exchange.getResponse().getStatusCode() != HttpStatus.OK) {
                HttpStatus status = exchange.getResponse().getStatusCode();
                exchange.getResponse().setStatusCode(null);
                throw new SentinelFallbackException(status);
            }
        }).onErrorResume(throwable -> sentinelFallbackHandler.fallback(exchange, UriUtils.createUri(sentinelHandle.getFallbackUri()), throwable));
    }
```

可以看到，SentinelRuleHandle类构造了resourceName。ResourceName对应sentinel规则，是一组熔断规则的唯一标识。

```java
public static String getResourceName(final RuleData ruleData) {
        return ruleData.getSelectorId() + "_" + ruleData.getName();
    }
```

可以看到，代码中构造了SentinelReactorTransformer对象，SentinelReactorTransformer用来完成异步情况下的熔断计数。

```java
@Slf4j
public class HystrixCommand extends HystrixObservableCommand<Void> implements Command {
    
    private final ServerWebExchange exchange;

    private final SoulPluginChain chain;

    private final URI callBackUri;
    
    /**
     * Instantiates a new Http command.
     *
     * @param setter      the setter
     * @param exchange    the exchange
     * @param chain       the chain
     * @param callBackUri the call back uri
     */
    public HystrixCommand(final Setter setter,
                   final ServerWebExchange exchange,
                   final SoulPluginChain chain,
                   final String callBackUri) {
        super(setter);
        this.exchange = exchange;
        this.chain = chain;
        this.callBackUri = UriUtils.createUri(callBackUri);
    }

    @Override
    protected Observable<Void> construct() {
        return RxReactiveStreams.toObservable(chain.execute(exchange));
    }

    @Override
    protected Observable<Void> resumeWithFallback() {
        return RxReactiveStreams.toObservable(doFallback());
    }

    private Mono<Void> doFallback() {
        if (isFailedExecution()) {
            log.error("hystrix execute have error: ", getExecutionException());
        }
        final Throwable exception = getExecutionException();
        return doFallback(exchange, exception);
    }

    @Override
    public Observable<Void> fetchObservable() {
        return this.toObservable();
    }

    @Override
    public URI getCallBackUri() {
        return callBackUri;
    }
}
```

再来看下sentinel的rule是如何同步的。

```java
public class SentinelRuleHandle implements PluginDataHandler {

    @Override
    public void handlerRule(final RuleData ruleData) {
        SentinelHandle sentinelHandle = GsonUtils.getInstance().fromJson(ruleData.getHandle(), SentinelHandle.class);

        List<FlowRule> flowRules = FlowRuleManager.getRules()
                .stream()
                .filter(r -> !r.getResource().equals(getResourceName(ruleData)))
                .collect(Collectors.toList());
        if (sentinelHandle.getFlowRuleEnable() == Constants.SENTINEL_ENABLE_FLOW_RULE) {
            FlowRule rule = new FlowRule(getResourceName(ruleData));
            rule.setCount(sentinelHandle.getFlowRuleCount());
            rule.setGrade(sentinelHandle.getFlowRuleGrade());
            rule.setControlBehavior(sentinelHandle.getFlowRuleControlBehavior());
            flowRules.add(rule);
        }
        FlowRuleManager.loadRules(flowRules);

        List<DegradeRule> degradeRules = DegradeRuleManager.getRules()
                .stream()
                .filter(r -> !r.getResource().equals(getResourceName(ruleData)))
                .collect(Collectors.toList());
        if (sentinelHandle.getDegradeRuleEnable() == Constants.SENTINEL_ENABLE_DEGRADE_RULE) {
            DegradeRule rule = new DegradeRule(getResourceName(ruleData));
            rule.setCount(sentinelHandle.getDegradeRuleCount());
            rule.setGrade(sentinelHandle.getDegradeRuleGrade());
            rule.setTimeWindow(sentinelHandle.getDegradeRuleTimeWindow());
            degradeRules.add(rule);
        }
        DegradeRuleManager.loadRules(degradeRules);
    }

    @Override
    public void removeRule(final RuleData ruleData) {
        FlowRuleManager.loadRules(FlowRuleManager.getRules()
                .stream()
                .filter(r -> !r.getResource().equals(getResourceName(ruleData)))
                .collect(Collectors.toList()));
        DegradeRuleManager.loadRules(DegradeRuleManager.getRules()
                .stream()
                .filter(r -> !r.getResource().equals(getResourceName(ruleData)))
                .collect(Collectors.toList()));
    }

    @Override
    public String pluginNamed() {
        return PluginEnum.SENTINEL.getName();
    }

    /**
     * return sentinel resource name.
     *
     * @param ruleData ruleData
     * @return string string
     */
    public static String getResourceName(final RuleData ruleData) {
        return ruleData.getSelectorId() + "_" + ruleData.getName();
    }

}

```

可以看到，流控规则和熔断规则加载后通过manager加载到内存中。