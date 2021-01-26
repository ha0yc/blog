## Soul Sofa初体验

书接上回，我们学习使用了dubbo client接入网关并从网关调用后端dubbo服务的流程和原理，本篇继续针对sofa客户端接入和sofa插件的调用流程和原理进行解析。

环境准备：

- 事先准备好zookeeper，监听2181端口
- run起来网关和控制台

## Run Demo

首先，我们把demo run起来，run soul-examples-sofa的TestSofaApplication即可。

启动日志如下：

![image-20210118215720249](D:\code\blog\images\sofa-4\sofa-log-1.jpeg)

![image-20210118215504450](D:\code\blog\images\sofa-4\sofa-log.jpeg)

通过日志可以看出，有3个服务注册到了控制台。

同时我们打开控制台，打开插件管理页面，sofa的status置为true。

打开sofa插件页面，可以看到注册的selector和rule规则都已经在控制台上显示了。

![image-20210118215858223](D:\code\blog\images\sofa-4\sofa-admin.jpeg)

前一篇中我们已经了解了服务信息是如何注册到控制台并如何下发到网关的流程，sofa流程相似，只不过需要额外将meta信息同时上送控制台，在此不做赘述。

插件配置完毕后，我们通过restClient调用网关。

```
GET http://localhost:9195/sofa/findById?id=sss
Accept: */*
Cache-Control: no-cache
```

返回结果为

```
{"code":200,"message":"Access to success!","data":{"name":"hello world Soul Sofa, findById","id":"sss"}}
```

可以得到结论，sofa实例运行成功。

## 源码跟踪

以上节请求为例，说明网关调用sofa服务的流程。

我们请求的url为/sofa/findById?id=sss。其中selector匹配规则为/sofa/**，rule匹配规则为/sofa/findById。在AbstractSoulPlugin中获取到selector和rule信息后，调用sofa插件的执行方法。

```
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

通过以上代码可以看出，SoulPlugin从上下文中拿到了参数和元数据，使用以上数据通过sofa泛化调用调起了后端的dubbo服务。

但是参数和元数据又怎么来的呢？

先说参数，通过追溯Constants.SOFA_PARAMS，可以看到ssoul-plugin-sofa包中有BodyParamPlugin。BodyParamPlugin完成了从body、parameters两种方式下取参数的行为，并放到了上下文中。

```
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

SofaMetaDataSubscriber实现了MetaDataSubscriber接口，用来监听元数据变化，元数据被MetaDataCache持有，DefaultSoulContextBuilder从MetaDataCache拿到元数据并构建SoulContext。GlobalPlugin在execute方法中将context放到exchange上下文，从而达到了传递元数据的目的。

```
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

public class SofaMetaDataSubscriber implements MetaDataSubscriber {

    private static final ConcurrentMap<String, MetaData> META_DATA = Maps.newConcurrentMap();

    @Override
    public void onSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
            MetaData exist = META_DATA.get(metaData.getPath());
            if (Objects.isNull(exist) || Objects.isNull(ApplicationConfigCache.getInstance().get(exist.getPath()).refer())) {
                // The first initialization
                ApplicationConfigCache.getInstance().initRef(metaData);
            } else {
                if (!exist.getServiceName().equals(metaData.getServiceName()) || !exist.getRpcExt().equals(metaData.getRpcExt())) {
                    // update
                    ApplicationConfigCache.getInstance().build(metaData);
                }
            }
            META_DATA.put(metaData.getPath(), metaData);
        }
    }
}
```

在SofaPluginConfiguration中，我们还看到了其他几个插件。在此我们进行一一分析。

除了SofaPlugin之外，还有SofaResponsePlugin。

```
public class SofaResponsePlugin implements SoulPlugin {
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            final Object result = exchange.getAttribute(Constants.SOFA_RPC_RESULT);
            if (Objects.isNull(result)) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
            return WebFluxResultUtils.result(exchange, success);
        }));
    }
}
```

可以看出，SofaResponsePlugin是对响应进行包装的插件，包装成soul的通用格式返回给前端。

## 总结

在Soul中，插件是有自己的套路的。已经学习了三种插件，从被调用的过程、参数读取的方式、数据同步的方式我们可以看到流程非常相近，只是因为后端服务不同导致其中的细微差别。