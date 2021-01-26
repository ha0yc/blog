## Soul Dubbo初体验

书接上回，我们学习使用了http服务注册和网关调用后端mvc服务的流程和原理，本篇继续讲解如何使用dubbo client接入网关并从网关调用后端dubbo服务的流程和原理。

环境准备：

- 事先准备好zookeeper，监听2181端口
- run起来网关和控制台

## Run Demo

首先，我们把demo run起来，run soul-examples-alibaba-dubbo-service的TestAlibabaDubboApplication即可。

启动日志如下：

![image-20210117015203847](D:\code\blog\images\dubbo-3\dubbo-log.jpeg)

通过日志可以看出，测试应用，监听了8188端口。同时看到有11个服务注册到了控制台。

同时我们打开控制台，打开插件管理页面，将divide的status置为false，将dubbo的status置为true。

打开dubbo插件页面，可以看到注册的selector和rule规则都已经在控制台上显示了。

![image-20210117015538877](D:\code\blog\images\dubbo-3\dubbo-plugin.jpeg)

前一篇中我们已经了解了服务信息是如何注册到控制台并如何下发到网关的流程，dubbo流程相似，只不过需要额外将meta信息同时上送控制台，在此不做赘述。

关于为什么要关闭divide，因为divide是全局插件，请求进来之后divide插件会报ERROR，返回selector not match。此处关于插件的全局性质需要继续深入探究，我们在这插个眼。

插件配置完毕后，我们通过restClient调用网关。

```
GET http://localhost:9195/dubbo/findById?id=sss
Accept: */*
Cache-Control: no-cache
```

返回结果为

```
{"code":200,"message":"Access to success!","data":{"name":"hello world Soul Alibaba Dubbo, findById","id":"sss"}}
```

可以得到结论，dubbo实例运行成功。

## 源码跟踪

以上节请求为例，说明网关调用dubbo服务的流程。

我们请求的url为/dubbo/findById?id=sss。其中selector匹配规则为/dubbo/**，rule匹配规则为/dubbo/findById。在AbstractSoulPlugin中获取到selector和rule信息后，调用dubbo插件的执行方法。

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

通过以上代码可以看出，AlibabaDubboPlugin从上下文中拿到了参数和元数据，使用以上数据通过dubbo泛化调用调起了后端的dubbo服务。

但是参数和元数据又怎么来的呢？

先说参数，通过追溯Constants.DUBBO_PARAMS，可以看到soul-plugin-alibaba-dubbo包中有BodyParamPlugin。BodyParamPlugin完成了从body、parameters两种方式下取参数的行为，并放到了上下文中。

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

soul-plugin-global包里MetaDataAllSubscriber实现了MetaDataSubscriber接口，用来监听元数据变化，元数据被MetaDataCache持有，DefaultSoulContextBuilder从MetaDataCache拿到元数据并构建SoulContext。GlobalPlugin在execute方法中将context放到exchange上下文，从而达到了传递元数据的目的。

```
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

到这里，dubbo的网关调用流程就解析完成了。

## 总结

- 关于divide和dubbo插件的不兼容问题，以及插件的权限问题，需要继续探索
- 继续了解soul中关于apacha-dubbo插件和alibaba-dubbo插件的区别
- 随着对soul的了解越来越深入，插件的加载机制，需要深入研究