## Soul：鉴权熔断插件原理分析

Soul网关不仅仅要对后端服务提供代理转发和灰度等基本功能，还要在网关层完成鉴权功能，也就是通过时间戳控制防重放、以及请求及参数加解密。

### Sign插件鉴权原理

通过观察调用链，请求进来会首先经过GlobalPlugin。GlobalPlugin根据请求信息生成SoulContext，生成时DefaultSoulContextBuilder会将AppKey、签名和时间戳放到SoulContext中。

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

在AbstractSoulPlugin中获取到selector和rule信息后，调用Sign插件的执行方法。

```java
public class SignPlugin extends AbstractSoulPlugin {

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        Pair<Boolean, String> result = signService.signVerify(exchange);
        if (!result.getLeft()) {
            Object error = SoulResultWrap.error(SoulResultEnum.SIGN_IS_NOT_PASS.getCode(), result.getRight(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        return chain.execute(exchange);
    }
}
```

可以看到，SignService真正完成了鉴权行为。

```java
@Slf4j
public class DefaultSignService implements SignService {
    
    @Value("${soul.sign.delay:5}")
    private int delay;
    
    @Override
    public Pair<Boolean, String> signVerify(final ServerWebExchange exchange) {
        PluginData signData = BaseDataCache.getInstance().obtainPluginData(PluginEnum.SIGN.getName());
        if (signData != null && signData.getEnabled()) {
            final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
            assert soulContext != null;
            return verify(soulContext, exchange);
        }
        return Pair.of(Boolean.TRUE, "");
    }
    
    private Pair<Boolean, String> verify(final SoulContext soulContext, final ServerWebExchange exchange) {
        if (StringUtils.isBlank(soulContext.getAppKey())
                || StringUtils.isBlank(soulContext.getSign())
                || StringUtils.isBlank(soulContext.getTimestamp())) {
            log.error("sign parameters are incomplete,{}", soulContext);
            return Pair.of(Boolean.FALSE, Constants.SIGN_PARAMS_ERROR);
        }
        final LocalDateTime start = DateUtils.formatLocalDateTimeFromTimestampBySystemTimezone(Long.parseLong(soulContext.getTimestamp()));
        final LocalDateTime now = LocalDateTime.now();
        final long between = DateUtils.acquireMinutesBetween(start, now);
        if (between > delay) {
            return Pair.of(Boolean.FALSE, String.format(SoulResultEnum.SING_TIME_IS_TIMEOUT.getMsg(), delay));
        }
        return sign(soulContext, exchange);
    }
    
    /**
     * verify sign .
     *
     * @param soulContext {@linkplain SoulContext}
     * @return result : True is pass, False is not pass.
     */
    private Pair<Boolean, String> sign(final SoulContext soulContext, final ServerWebExchange exchange) {
        final AppAuthData appAuthData = SignAuthDataCache.getInstance().obtainAuthData(soulContext.getAppKey());
        if (Objects.isNull(appAuthData) || !appAuthData.getEnabled()) {
            log.error("sign APP_kEY does not exist or has been disabled,{}", soulContext.getAppKey());
            return Pair.of(Boolean.FALSE, Constants.SIGN_APP_KEY_IS_NOT_EXIST);
        }
        List<AuthPathData> pathDataList = appAuthData.getPathDataList();
        if (CollectionUtils.isEmpty(pathDataList)) {
            log.error("You have not configured the sign path:{}", soulContext.getAppKey());
            return Pair.of(Boolean.FALSE, Constants.SIGN_PATH_NOT_EXIST);
        }
   
        boolean match = pathDataList.stream().filter(AuthPathData::getEnabled)
                .anyMatch(e -> PathMatchUtils.match(e.getPath(), soulContext.getPath()));
        if (!match) {
            log.error("You have not configured the sign path:{},{}", soulContext.getAppKey(), soulContext.getRealUrl());
            return Pair.of(Boolean.FALSE, Constants.SIGN_PATH_NOT_EXIST);
        }
        String sigKey = SignUtils.generateSign(appAuthData.getAppSecret(), buildParamsMap(soulContext));
        boolean result = Objects.equals(sigKey, soulContext.getSign());
        if (!result) {
            log.error("the SignUtils generated signature value is:{},the accepted value is:{}", sigKey, soulContext.getSign());
            return Pair.of(Boolean.FALSE, Constants.SIGN_VALUE_IS_ERROR);
        } else {
            List<AuthParamData> paramDataList = appAuthData.getParamDataList();
            if (CollectionUtils.isEmpty(paramDataList)) {
                return Pair.of(Boolean.TRUE, "");
            }
            paramDataList.stream().filter(p ->
                    ("/" + p.getAppName()).equals(soulContext.getContextPath()))
                    .map(AuthParamData::getAppParam)
                    .filter(StringUtils::isNoneBlank).findFirst()
                    .ifPresent(param -> exchange.getRequest().mutate().headers(httpHeaders -> httpHeaders.set(Constants.APP_PARAM, param)).build()
            );
        }
        return Pair.of(Boolean.TRUE, "");
    }
    
    private Map<String, String> buildParamsMap(final SoulContext dto) {
        Map<String, String> map = Maps.newHashMapWithExpectedSize(3);
        map.put(Constants.TIMESTAMP, dto.getTimestamp());
        map.put(Constants.PATH, dto.getPath());
        map.put(Constants.VERSION, "1.0.0");
        return map;
    }
}
```

可以看到，signVerify方法中首先从缓存中获取了插件数据，然后调用了verify方法，获取放到上下文中的appKey、签名和时间戳信息，随后计算时间戳是否在当前系统时间的5分钟内。先比较时间戳，可以避免签名计算带来的性能损耗。

时间戳正常则进入到sign方法，计算签名是否一致。首先从缓存中获取AppAuthData，判断当前的上下文路径是否在AppAuthData的路径数据中，在的话就根据appSecret和参数map构建签名，并比较签名是否一致。如果验签成功，则将param放到header向后转发。