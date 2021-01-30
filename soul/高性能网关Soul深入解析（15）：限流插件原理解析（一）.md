## Soul：hystrix熔断插件原理分析

Soul网关不仅仅要对后端服务提供代理转发和灰度等基本功能，还要在网关层完成限流和熔断功能。

### Hystrix配置

首先开启hystrix插件，之后如图所示配置selector和rule数据，这里url配置的都是/http/order/findById。

![image-20210130004648514](D:\code\blog\soul\images\hystrix-15\14-1.jpeg)

![image-20210130004741403](D:\code\blog\soul\images\hystrix-15\14-2.jpeg)

Hystrix规则的参数设置为，最小请求数为50 ，熔断异常比例阈值为50%，休眠时间5s，最大并发请求5。

通过client发送请求，可以看到经过了Hystrix逻辑。经过压测，1000tps，100000次请求可以看到熔断器被开启。

```java
2021-01-30 01:46:22.158  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :sss
2021-01-30 01:46:22.158 ERROR 2192 --- [work-threads-14] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-01-30 01:46:22.158  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :sss
2021-01-30 01:46:22.158  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :sss
2021-01-30 01:46:22.159 ERROR 2192 --- [work-threads-14] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-01-30 01:46:22.159  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :sss
2021-01-30 01:46:22.159  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :sss
2021-01-30 01:46:22.159 ERROR 2192 --- [work-threads-14] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-01-30 01:46:22.159  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :sss
2021-01-30 01:46:22.159  INFO 2192 --- [work-threads-14] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :sss
2021-01-30 01:46:22.159 ERROR 2192 --- [work-threads-14] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
```



### Hystrix插件熔断原理

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

在AbstractSoulPlugin中获取到selector和rule信息后，调用Hystrix插件的执行方法。

```java
	@Slf4j
public class HystrixPlugin extends AbstractSoulPlugin {

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final HystrixHandle hystrixHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), HystrixHandle.class);
        if (StringUtils.isBlank(hystrixHandle.getGroupKey())) {
            hystrixHandle.setGroupKey(Objects.requireNonNull(soulContext).getModule());
        }
        if (StringUtils.isBlank(hystrixHandle.getCommandKey())) {
            hystrixHandle.setCommandKey(Objects.requireNonNull(soulContext).getMethod());
        }
        Command command = fetchCommand(hystrixHandle, exchange, chain);
        return Mono.create(s -> {
            Subscription sub = command.fetchObservable().subscribe(s::success,
                    s::error, s::success);
            s.onCancel(sub::unsubscribe);
            if (command.isCircuitBreakerOpen()) {
                log.error("hystrix execute have circuitBreaker is Open! groupKey:{},commandKey:{}", hystrixHandle.getGroupKey(), hystrixHandle.getCommandKey());
            }
        }).doOnError(throwable -> {
            log.error("hystrix execute exception:", throwable);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
            chain.execute(exchange);
        }).then();
    }
}
```

可以看到，通过fetchCommand方法构造了command对象。Command对象是hystrix用来封装远程调用的依赖对象。

```java
private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
    if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
        return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
            exchange, chain, hystrixHandle.getCallBackUri());
    }
    return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
        exchange, chain, hystrixHandle.getCallBackUri());
}

public static HystrixObservableCommand.Setter build(final HystrixHandle hystrixHandle) {
        initHystrixHandleOnRequire(hystrixHandle);
        HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey(hystrixHandle.getGroupKey());
        HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey(hystrixHandle.getCommandKey());
        HystrixCommandProperties.Setter propertiesSetter =
                HystrixCommandProperties.Setter()
                        .withExecutionTimeoutInMilliseconds((int) hystrixHandle.getTimeout())
                        .withCircuitBreakerEnabled(true)
                        .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)
                        .withExecutionIsolationSemaphoreMaxConcurrentRequests(hystrixHandle.getMaxConcurrentRequests())
                        .withCircuitBreakerErrorThresholdPercentage(hystrixHandle.getErrorThresholdPercentage())
                        .withCircuitBreakerRequestVolumeThreshold(hystrixHandle.getRequestVolumeThreshold())
                        .withCircuitBreakerSleepWindowInMilliseconds(hystrixHandle.getSleepWindowInMilliseconds());
        return HystrixObservableCommand.Setter
                .withGroupKey(groupKey)
                .andCommandKey(commandKey)
                .andCommandPropertiesDefaults(propertiesSetter);
    }
```

可以看到，代码中构造了HystrixObservableCommand对象，HystrixObservableCommand 用在依赖服务返回多个操作结果的时候。它有两种实现方式，observe或者toObservable方式。

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

soul对HystrixObservableCommand进行了封装，使用的是toObservable方法。

实际上，soul在以下代码实现了限流逻辑。

```java
return Mono.create(s -> {
    Subscription sub = command.fetchObservable().subscribe(s::success,
            s::error, s::success);
    s.onCancel(sub::unsubscribe);
    if (command.isCircuitBreakerOpen()) {
        log.error("hystrix execute have circuitBreaker is Open! groupKey:{},commandKey:{}", hystrixHandle.getGroupKey(), hystrixHandle.getCommandKey());
    }
}).doOnError(throwable -> {
    log.error("hystrix execute exception:", throwable);
    exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
    chain.execute(exchange);
}).then();
```

command通过fetchObservable方法判断是否限流成功，并注册了成功和失败回调方法。