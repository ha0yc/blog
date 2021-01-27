## Soul：Dubbo & Sofa插件

在Soul网关中Dubbo插件用来针对dubbo服务将http请求转换为dubbo请求，本文将探究dubbo的运行机制和底层原理.

Alibaba Dubbo和Apache Dubbo同宗同源，系出同门。Apache Dubbo是阿里将Alibaba Dubbo捐献给apache基金会后毕业的顶级开源项目，在调用流程和框架架构上区别微小。因此在这里我们只看Alibaba Dubbo插件即可。

### Alibaba Dubbo插件的请求调用原理

和divide插件一样，Alibaba Dubbo插件也处在调用链中。通过AbstractSoulPlugin筛选，拿到了对应的selector和rule数据之后，请求进入到了AlibabaDubboPlugin中。

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

以上节请求为例，说明网关调用dubbo服务的流程。

我们请求的url为/dubbo/findById?id=sss。其中selector匹配规则为/dubbo/**，rule匹配规则为/dubbo/findById。在AbstractSoulPlugin中获取到selector和rule信息后，调用dubbo插件的执行方法。

```java
	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        Object result = alibabaDubboProxyService.genericInvoker(body, metaData);
        if (Objects.nonNull(result)) {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, result);
        } else {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, Constants.DUBBO_RPC_RESULT_EMPTY);
        }
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        return chain.execute(exchange);
    }
```

通过以上代码可以看出，AlibabaDubboPlugin从上下文中拿到了参数和元数据，使用以上数据通过dubbo泛化调用调起了后端的dubbo服务。

但是参数和元数据又怎么来的呢？

先说参数，通过追溯Constants.DUBBO_PARAMS，可以看到soul-plugin-alibaba-dubbo包中有BodyParamPlugin。BodyParamPlugin完成了从body、parameters两种方式下取参数的行为，并放到了上下文中。

```java
	@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        if (Objects.nonNull(soulContext) && RpcTypeEnum.DUBBO.getName().equals(soulContext.getRpcType())) {
            MediaType mediaType = request.getHeaders().getContentType();
            ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }
```

我们来看MetaData，从前边知道服务的元数据在服务启动时就上送到了控制台。

soul-plugin-global包里MetaDataAllSubscriber实现了MetaDataSubscriber接口，用来监听元数据变化，元数据被MetaDataCache持有，DefaultSoulContextBuilder从MetaDataCache拿到元数据并构建SoulContext。GlobalPlugin在execute方法中将context放到exchange上下文，从而达到了传递元数据的目的。

```java
public class GlobalPlugin implements SoulPlugin {
    
    private final SoulContextBuilder builder;
    
    /**
     * Instantiates a new Global plugin.
     *
     * @param builder the builder
     */
    public GlobalPlugin(final SoulContextBuilder builder) {
        this.builder = builder;
    }
    
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
    
    @Override
    public int getOrder() {
        return 0;
    }
    
    private SoulContext transformMap(final MultiValueMap<String, String> queryParams) {
        SoulContext soulContext = new SoulContext();
        soulContext.setModule(queryParams.getFirst(Constants.MODULE));
        soulContext.setMethod(queryParams.getFirst(Constants.METHOD));
        soulContext.setRpcType(queryParams.getFirst(Constants.RPC_TYPE));
        return soulContext;
    }
    
    @Override
    public String named() {
        return "global";
    }
}

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
}

public class MetaDataAllSubscriber implements MetaDataSubscriber {
    
    @Override
    public void onSubscribe(final MetaData metaData) {
        MetaDataCache.getInstance().cache(metaData);
    }
    
    @Override
    public void unSubscribe(final MetaData metaData) {
        MetaDataCache.getInstance().remove(metaData);
    }
}
```

接下来，我们要了解下什么叫泛化调用。dubbo文档中如下所述：

```
泛化接口调用方式主要用于客户端没有 API 接口及模型类元的情况，参数及返回值中的所有 POJO 均用 Map 表示，通常用于框架集成，比如：实现一个通用的服务测试框架，可通过 GenericService 调用所有服务实现。
```

作为网关层，是没办法到了服务的存根的，因此需要使用泛化调用的方式来完成后端dubbo服务调用。Soul通过AlibabaDubboProxyService封装了一层代理服务类完成方法调用。

```java
@Slf4j
public class AlibabaDubboProxyService {
    
    private final DubboParamResolveService dubboParamResolveService;
    
    /**
     * Instantiates a new Dubbo proxy service.
     *
     * @param dubboParamResolveService the generic param resolve service
     */
    public AlibabaDubboProxyService(final DubboParamResolveService dubboParamResolveService) {
        this.dubboParamResolveService = dubboParamResolveService;
    }
    
    /**
     * Generic invoker object.
     *
     * @param body     the body
     * @param metaData the meta data
     * @return the object
     * @throws SoulException the soul exception
     */
    public Object genericInvoker(final String body, final MetaData metaData) throws SoulException {
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        GenericService genericService = reference.get();
        try {
            Pair<String[], Object[]> pair;
            if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
                pair = new ImmutablePair<>(new String[]{}, new Object[]{});
            } else {
                pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
            }
            return genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        } catch (GenericException e) {
            log.error("dubbo invoker have exception", e);
            throw new SoulException(e.getExceptionMessage());
        }
    }
}
```

我们可以看到，通过ApplicationConfigCache拿到了远程服务的实例，ReferenceConfig中封装了所有与注册中心及服务提供方连接因此需要缓存存储，避免每次调用都new一个新实例出来。

那么问题来了，ApplicationConfigCache里的远程服务实例是怎么加载的呢？这里重温一遍数据同步流程，其中有一个步骤是元数据信息的同步。我们找到AlibabaDubbo的元数据同步类AlibabaDubboMetaDataSubscriber。在onSubscribe方法中看到了，加载元信息的时候也同步完成远程服务实例的缓存写入。

```java
@Override
public void onSubscribe(final MetaData metaData) {
    if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
        MetaData exist = META_DATA.get(metaData.getPath());
        if (Objects.isNull(META_DATA.get(metaData.getPath())) || Objects.isNull(ApplicationConfigCache.getInstance().get(metaData.getPath()))) {
            // The first initialization
            ApplicationConfigCache.getInstance().initRef(metaData);
        } else {
            // There are updates, which only support the update of four properties of serviceName rpcExt parameterTypes methodName,
            // because these four properties will affect the call of Dubbo;
            if (!Objects.equals(metaData.getServiceName(), exist.getServiceName())
                    || !Objects.equals(metaData.getRpcExt(), exist.getRpcExt())
                    || !Objects.equals(metaData.getParameterTypes(), exist.getParameterTypes())
                    || !Objects.equals(metaData.getMethodName(), exist.getMethodName())) {
                ApplicationConfigCache.getInstance().build(metaData);
            }
        }
        META_DATA.put(metaData.getPath(), metaData);
    }
}
```

回到泛化调用流程中，我们拿到了ReferenceConfig之后获取到泛化服务，就可以按照dubbo文档中对泛化调用的描述完成调用。

### Sofa插件的调用原理

Sofa插件用来针对sofa服务将http请求转换为sofa请求。我们先看以下Sofa插件的执行过程。

```java
@Slf4j
public class SofaPlugin extends AbstractSoulPlugin {

    private final SofaProxyService sofaProxyService;

    /**
     * Instantiates a new Sofa plugin.
     *
     * @param sofaProxyService the sofa proxy service
     */
    public SofaPlugin(final SofaProxyService sofaProxyService) {
        this.sofaProxyService = sofaProxyService;
    }

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.SOFA_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.SOFA_HAVE_BODY_PARAM.getCode(), SoulResultEnum.SOFA_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final Mono<Object> result = sofaProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }

    /**
     * acquire plugin name.
     *
     * @return plugin name.
     */
    @Override
    public String named() {
        return PluginEnum.SOFA.getName();
    }

    /**
     * plugin is execute.
     *
     * @param exchange the current server exchange
     * @return default false.
     */
    @Override
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(soulContext.getRpcType(), RpcTypeEnum.SOFA.getName());
    }

    @Override
    public int getOrder() {
        return PluginEnum.SOFA.getCode();
    }

    private boolean checkMetaData(final MetaData metaData) {
        return null != metaData && !StringUtils.isBlank(metaData.getMethodName()) && !StringUtils.isBlank(metaData.getServiceName());
    }

}
```

可以看到，和前述dubbo的调用流程相似，都是通过泛化服务调用的方式调用的后端服务，但是在泛化调用的过程中略有不同。

```java
@Slf4j
public class SofaProxyService {

    private final SofaParamResolveService sofaParamResolveService;

    /**
     * Instantiates a new Sofa proxy service.
     *
     * @param sofaParamResolveService the generic param resolve service
     */
    public SofaProxyService(final SofaParamResolveService sofaParamResolveService) {
        this.sofaParamResolveService = sofaParamResolveService;
    }
    
    /**
     * Generic invoker object.
     *
     * @param body     the body
     * @param metaData the meta data
     * @param exchange the exchange
     * @return the object
     * @throws SoulException the soul exception
     */
    public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterfaceId())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getServiceName());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        GenericService genericService = reference.refer();
        Pair<String[], Object[]> pair;
        if (null == body || "".equals(body) || "{}".equals(body) || "null".equals(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = sofaParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        CompletableFuture<Object> future = new CompletableFuture<>();
        RpcInvokeContext.getContext().setResponseCallback(new SofaResponseCallback<Object>() {
            @Override
            public void onAppResponse(final Object o, final String s, final RequestBase requestBase) {
                future.complete(o);
            }

            @Override
            public void onAppException(final Throwable throwable, final String s, final RequestBase requestBase) {
                future.completeExceptionally(throwable);
            }

            @Override
            public void onSofaException(final SofaRpcException e, final String s, final RequestBase requestBase) {
                future.completeExceptionally(e);
            }
        });
        genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.SOFA_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.SOFA_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(SoulException::new);
    }
}
```

这里sofa代理服务首先创建了CompletableFuture对象，并通过RpcInvokeContext注册了回调方法，然后才进行后端服务调用。