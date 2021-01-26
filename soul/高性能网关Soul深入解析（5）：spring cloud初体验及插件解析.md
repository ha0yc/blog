## Soul SpringCloud初体验

书接上回，我们学习使用了sofa client接入网关并从网关调用后端sofa服务的流程和原理，本篇继续针对springCloud客户端接入和插件的调用流程和原理进行解析。

环境准备：

- 事先准备好eureka，监听8761端口
- run起来网关和控制台

## Run Demo

首先，我们把demo run起来。我们看到在soul-examples包下面，有soul-examples-eureka包，就是eureka的启动包。先看下eureka的配置。

```yml
server:
  port: 8761

spring:
  application:
    name: springcloud-eureka

eureka:
  instance:
    hostname: ${hostname:localhost}
    preferIpAddress: true
    lease-renewal-interval-in-seconds: 2
    lease-expiration-duration-in-seconds: 6
  server:
    peerEurekaNodesUpdateIntervalMs: 60000
    enableSelfPreservation: false
    evictionIntervalTimerInMs: 5000
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
    healthcheck:
      enabled: true
    eurekaServiceUrlPollIntervalSeconds: 60
```

我们可以看到，eureka监听8761端口。启动eureka，启动日志如下：

![image-20210119234911642](D:\code\blog\images\spring-cloud-5\eureka-log.jpeg)

之后启动，spring-cloud的example，配置上eureka的地址。

```
server:
  port: 8884
  address: 0.0.0.0

spring:
  application:
    name: springCloud-test

springCloud-test:
  ribbon.NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule

eureka:
  client:
    serviceUrl:
      defaultZone:  http://localhost:8761/eureka/

soul:
  springcloud:
    admin-url: http://localhost:9095
    context-path: /springcloud

logging:
  level:
    root: info
    org.dromara.soul: debug
  path: "./logs"
```

启动日志如下：

![image-20210119235111765](D:\code\blog\images\spring-cloud-5\spring-cloud-log.jpeg)

通过日志可以看出，有5个服务注册到了控制台。同时，本服务作为eureka的client注册到了eureka上。

同时我们打开控制台，打开插件管理页面，将springCloud插件的status置为true。

打开springCloud插件页面，可以看到注册的selector和rule规则都已经在控制台上显示了。

![image-20210119235306211](D:\code\blog\images\spring-cloud-5\soul-admin.jpeg)

之前我们已经了解了服务信息是如何注册到控制台并如何下发到网关的流程，springCloud流程相似，在此不做赘述。

插件配置完毕后，我们通过restClient调用网关。

```
GET http://localhost:9195/springcloud/order/findById?id=sss
Accept: */*
Cache-Control: no-cache
```

返回结果提示找不到service，我们跟踪SpringCloudPlugin的执行流程。

```java
final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
```

发现在这一行中没有获取到server信息，继续向下debug。发现在PredicateBasedRule中的choose方法里，没有拿到serverInstance。

```
public Server choose(Object key) {
    ILoadBalancer lb = this.getLoadBalancer();
    Optional<Server> server = this.getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    return server.isPresent() ? (Server)server.get() : null;
}
```

此时我们有两种猜测，一是网关侧没获取到注册中心上的实例信息；二是SpringCloud应用没有注册到eureka上去。通过前面SpringCloud示例启动的日志来看，后者是不成立的。因此我们重新过了一遍网关的配置，发现确实是没有获取到注册中心的实例。因此我们在pom中加入了eureka的client依赖，并在配置中增加了eureka的配置。重新启动网关再次发送请求，可以看到请求成功，问题已经fix。

```
{"id":"sss","name":"hello world spring cloud findById"}
```

## 源码跟踪

以上节请求为例，说明网关调用sofa服务的流程。

通过观察调用链，请求进来会首先经过GlobalPlugin。GlobalPlugin根据请求信息生成SoulContext，生成时DefaultSoulContextBuilder会截短url，生成realUrl和其他http相关信息放到SoulContext中。

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
```

```
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

我们请求的url为/springcloud/order/findById?id=sss。其中selector匹配规则为/springcloud/**，rule匹配规则为/springcloud/order/findById。在AbstractSoulPlugin中获取到selector和rule信息后，调用SpringCoud插件的执行方法。

```
	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        if (Objects.isNull(rule)) {
            return Mono.empty();
        }
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final SpringCloudRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SpringCloudRuleHandle.class);
        final SpringCloudSelectorHandle selectorHandle = GsonUtils.getInstance().fromJson(selector.getHandle(), SpringCloudSelectorHandle.class);
        if (StringUtils.isBlank(selectorHandle.getServiceId()) || StringUtils.isBlank(ruleHandle.getPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getCode(), SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }

        final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final URI uri = loadBalancer.reconstructURI(serviceInstance, URI.create(soulContext.getRealUrl()));

        String realURL = buildRealURL(uri.toASCIIString(), soulContext.getHttpMethod(), exchange.getRequest().getURI().getQuery());

        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        //set time out.
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        return chain.execute(exchange);
    }
```

首先取出SoulContext，并将selector和rule的handle属性转换为对象。使用serviceId通过负载均衡器获取服务实例，拿到后根据实例信息和url等构建真正的后端信url，并放到exchange向后传递，并通过webClient插件请求后端服务拿到response。
