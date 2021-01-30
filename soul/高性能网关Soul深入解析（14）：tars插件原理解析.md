## Soul：Tars插件原理和底层协议分析

在Soul网关中Tars插件用来针对Tars服务将http请求转换为Tars请求，本文将探究Tars插件的运行机制和底层原理.

### Tars插件的请求调用原理

和divide插件一样，Tars插件也处在调用链中。通过AbstractSoulPlugin筛选，拿到了对应的selector和rule数据之后，请求进入到了Tars插件中。

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

在AbstractSoulPlugin中获取到selector和rule信息后，调用Tars插件的执行方法。

```java
	@Override
    @SuppressWarnings("unchecked")
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.TARS_PARAMS);
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
            Object error = SoulResultWrap.error(SoulResultEnum.TARS_HAVE_BODY_PARAM.getCode(), SoulResultEnum.TARS_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        TarsInvokePrxList tarsInvokePrxList = ApplicationConfigCache.getInstance().get(metaData.getPath());
        int index = RANDOM.nextInt(tarsInvokePrxList.getTarsInvokePrxList().size());
        Object prx = tarsInvokePrxList.getTarsInvokePrxList().get(index).getInvokePrx();
        Method method = tarsInvokePrxList.getMethod();
        CompletableFuture future;
        try {
            future = (CompletableFuture) method
                    .invoke(prx, PrxInfoUtil.getParamArray(tarsInvokePrxList.getParamTypes(), tarsInvokePrxList.getParamNames(), body));
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.TARS_INVOKE.getCode(), SoulResultEnum.TARS_INVOKE.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.TARS_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.TARS_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(m -> new SoulException("failed to invoke tars")).then(chain.execute(exchange));
    }
```

通过以上代码可以看出，AlibabaDubboPlugin从上下文中拿到了参数和元数据，使用以上数据调起了后端的Tars服务。

但是参数和元数据又怎么来的呢？

先说参数，通过追溯Constants.TARS_PARAMS，可以看到ssoul-plugin-tars包中有BodyParamPlugin。BodyParamPlugin完成了从body、parameters两种方式下取参数的行为，并放到了上下文中。

```java
	@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        if (Objects.nonNull(soulContext) && RpcTypeEnum.TARS.getName().equals(soulContext.getRpcType())) {
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

拿到参数和元数据信息后，Soul通过ApplicationConfigCache获取到了服务器代理列表，并随机负载均衡到其中一台机器上。然后通过代理方式调用后端的tars服务。

接下来我们可以看一下，如何构造tars远程服务实例。我们可以在tars插件流程中看到，通过ApplicationConfigCache拿到了远程服务的实例。

那么问题来了，ApplicationConfigCache里的远程服务实例是怎么加载的呢？这里重温一遍数据同步流程，其中有一个步骤是元数据信息的同步。我们找到Tars的元数据同步类TarsMetaDataSubscriber。在onSubscribe方法中看到了，加载元信息的时候也同步完成远程服务实例的缓存写入。

```java
@Override
    public void onSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
            MetaData metaExist = META_DATA.get(metaData.getPath());
            List<TarsInvokePrx> prxList = ApplicationConfigCache.getInstance()
                    .get(metaData.getPath()).getTarsInvokePrxList();
            boolean exist = prxList.stream().anyMatch(tarsInvokePrx -> tarsInvokePrx.getHost().equals(metaData.getAppName()));
            if (!exist) {
                ApplicationConfigCache.getInstance().initPrx(metaData);
            }
            if (Objects.isNull(metaExist)) {
                META_DATA.put(metaData.getPath(), metaData);
            }
        }
    }
```

接下来看一下是怎么构造代理对象的。

```java
public void initPrx(final MetaData metaData) {
    for (; ;) {
        Class<?> prxClass = prxClassCache.get(metaData.getServiceName());
        try {
            if (Objects.isNull(prxClass)) {
                assert LOCK != null;
                if (LOCK.tryLock()) {
                    try {
                        if (StringUtils.isEmpty(metaData.getRpcExt())) {
                            throw new SoulException("can't init prx with empty ext string");
                        }
                        String clazzName = PrxInfoUtil.getPrxName(metaData);
                        TarsParamExtInfo tarsParamExtInfo = GsonUtils.getInstance().fromJson(metaData.getRpcExt(), TarsParamExtInfo.class);
                        DynamicType.Builder<?> classDefinition = new ByteBuddy()
                                .makeInterface()
                                .name(clazzName);
                        for (MethodInfo methodInfo : tarsParamExtInfo.getMethodInfo()) {
                            DynamicType.Builder.MethodDefinition.ParameterDefinition<?> definition =
                                    classDefinition.defineMethod(PrxInfoUtil.getMethodName(methodInfo.methodName),
                                            ReturnValueResolver.getCallBackType(Class.forName(methodInfo.getReturnType())),
                                            Visibility.PUBLIC);
                            if (CollectionUtils.isNotEmpty(methodInfo.getParams())) {
                                Class<?>[] paramTypes = new Class[methodInfo.getParams().size()];
                                String[] paramNames = new String[methodInfo.getParams().size()];
                                for (int i = 0; i < methodInfo.getParams().size(); i++) {
                                    Pair<String, String> pair = methodInfo.getParams().get(i);
                                    paramTypes[i] = PrxInfoUtil.getParamClass(pair.getKey());
                                    paramNames[i] = pair.getValue();
                                    definition = definition.withParameter(paramTypes[i], paramNames[i]);
                                    prxParamCache.put(getClassMethodKey(clazzName, methodInfo.getMethodName()), new TarsParamInfo(paramTypes, paramNames));
                                }
                                classDefinition = definition.withoutCode();
                            }
                        }
                        Class<?> prxClazz = classDefinition.annotateType(AnnotationDescription.Builder.ofType(Servant.class).build())
                                .make()
                                .load(Servant.class.getClassLoader(), ClassLoadingStrategy.Default.INJECTION)
                                .getLoaded();
                        assert communicator != null;
                        prxClassCache.put(metaData.getServiceName(), prxClazz);
                    } finally {
                        LOCK.unlock();
                    }
                }
            } else {
                // if object name is same it will return same prx
                Object prx = communicator.stringToProxy(prxClass, PrxInfoUtil.getObjectName(metaData));
                TarsInvokePrxList tarsInvokePrxList = cache.get(metaData.getPath());
                if (tarsInvokePrxList.getMethod() == null) {
                    TarsParamInfo tarsParamInfo = prxParamCache.get(getClassMethodKey(prxClass.getName(), metaData.getMethodName()));
                    Method method = prx.getClass().getDeclaredMethod(
                            PrxInfoUtil.getMethodName(metaData.getMethodName()), tarsParamInfo.getParamTypes());
                    tarsInvokePrxList.setMethod(method);
                    tarsInvokePrxList.setParamTypes(tarsParamInfo.getParamTypes());
                    tarsInvokePrxList.setParamNames(tarsParamInfo.getParamNames());
                }
                tarsInvokePrxList.getTarsInvokePrxList().add(new TarsInvokePrx(prx, metaData.getAppName()));
                break;
            }
        } catch (Exception e) {
            log.error("init tars ref ex:{}", e.getMessage());
            break;
        }
    }
}
```

这里通过ByteBuddy字节码生成了代理类prxClazz，放到了缓存中。然后通过communicator完成了tars远程服务实例的构造。