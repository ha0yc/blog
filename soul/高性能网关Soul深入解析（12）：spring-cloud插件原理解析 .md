## Soul：Springcloud插件

在Soul网关中spring-cloud插件用来针对spring-cloud服务将http请求转换为spring-cloud请求，本文将探究spring-cloud插件的运行机制和底层原理

### spring-cloud插件的请求调用原理

和divide插件一样，spring-cloud插件也处在调用链中。通过AbstractSoulPlugin筛选，拿到了对应的selector和rule数据之后，请求进入到了SpringCloudPlugin中。

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

首先取出SoulContext，并将selector和rule的handle属性转换为对象。

由于selector和rule属于通用pojo类，这里的SpringCloudRuleHandle和SpringCloudSelectorHandle是存储的真正的关于路由规则的信息，这里使用gson将string序列化为对象。

拿到路由数据之后就是最重要的一步，通过负载均衡器获取服务实例。

在SpringCloudPluginConfiguration类里面可以看到，我们通过构造器注入的方式拿到了LoadBalancerClient。

```
@ConditionalOnClass({LoadBalancerClient.class, RibbonAutoConfiguration.class, DispatcherHandler.class})
@AutoConfigureAfter(RibbonAutoConfiguration.class)
@ConditionalOnBean(LoadBalancerClient.class)
@Configuration
public class SpringCloudPluginConfiguration {

    /**
     * init springCloud plugin.
     *
     * @param loadBalancerClient the load balancer client
     * @return {@linkplain SpringCloudPlugin}
     */
    @Bean
    public SoulPlugin springCloudPlugin(final ObjectProvider<LoadBalancerClient> loadBalancerClient) {
        return new SpringCloudPlugin(loadBalancerClient.getIfAvailable());
    }
}
```

可以看到，配置类使用了条件注册bean的方式，依赖于RibbonAutoConfiguration的存在。

我们配置文件中配置了ribbon的配置，说明是通过ribbon的随机规则进行匹配服务实例的。

```
springCloud-test:
  ribbon.NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

因此我们在SpringCoud插件拿到的loadBalancer即是RibbonLoadBalancerClient。

我们来看一下RibbonLoadBalancerClient是如何选择服务实例的。

```
/**
 * New: Select a server using a 'key'.
 * @param serviceId of the service to choose an instance for
 * @param hint to specify the service instance
 * @return the selected {@link ServiceInstance}
 */
public ServiceInstance choose(String serviceId, Object hint) {
   Server server = getServer(getLoadBalancer(serviceId), hint);
   if (server == null) {
      return null;
   }
   return new RibbonServer(serviceId, server, isSecure(server, serviceId),
         serverIntrospector(serviceId).getMetadata(server));
}

protected ILoadBalancer getLoadBalancer(String serviceId) {
		return this.clientFactory.getLoadBalancer(serviceId);
	}
	
	protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
		if (loadBalancer == null) {
			return null;
		}
		// Use 'default' on a null hint, or just pass it on?
		return loadBalancer.chooseServer(hint != null ? hint : "default");
	}
```

可以看到，首先通过服务id获取负载均衡器，然后获取服务实例，并包装返回。

这里持有了SpringClientFactory，这个bean提供了获取实例的方法，从而拿到了负载均衡器。

```
@Override
public <C> C getInstance(String name, Class<C> type) {
   C instance = super.getInstance(name, type);
   if (instance != null) {
      return instance;
   }
   IClientConfig config = getInstance(name, IClientConfig.class);
   return instantiateWithConfig(getContext(name), type, config);
}
```

然后通过BaseLoadBalancer的chooseServer方法获取到了server实例。

```
public Server chooseServer(Object key) {
    if (this.counter == null) {
        this.counter = this.createCounter();
    }

    this.counter.increment();
    if (this.rule == null) {
        return null;
    } else {
        try {
            return this.rule.choose(key);
        } catch (Exception var3) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", new Object[]{this.name, key, var3});
            return null;
        }
    }
}
```

这里支持多种策略选择最终的实例，但是我们在配置中选择了随机规则，也就是RandomRule。

```
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    } else {
        Server server = null;

        while(server == null) {
            if (Thread.interrupted()) {
                return null;
            }

            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();
            int serverCount = allList.size();
            if (serverCount == 0) {
                return null;
            }

            int index = this.chooseRandomInt(serverCount);
            server = (Server)upList.get(index);
            if (server == null) {
                Thread.yield();
            } else {
                if (server.isAlive()) {
                    return server;
                }

                server = null;
                Thread.yield();
            }
        }

        return server;
    }
}
```

可以看到，负载均衡器本身持有了所有的server，通过随机数获取到了server索引，然后进行server探活。如果正常则返回server，如果不正常就进行循环，直到找到server为止。

拿到了server之后，回过头来继续看SpringCloud插件的调用流程。通过负载均衡器构建URI

```
@Override
public URI reconstructURI(ServiceInstance instance, URI original) {
   Assert.notNull(instance, "instance can not be null");
   String serviceId = instance.getServiceId();
   RibbonLoadBalancerContext context = this.clientFactory
         .getLoadBalancerContext(serviceId);

   URI uri;
   Server server;
   if (instance instanceof RibbonServer) {
      RibbonServer ribbonServer = (RibbonServer) instance;
      server = ribbonServer.getServer();
      uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
   }
   else {
      server = new Server(instance.getScheme(), instance.getHost(),
            instance.getPort());
      IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
      ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
      uri = updateToSecureConnectionIfNeeded(original, clientConfig,
            serverIntrospector, server);
   }
   return context.reconstructURIWithServer(server, uri);
}
```

这里根据server配置构建出了URI，并通过buildRealURL构建出了带参数的GET请求。

```
private String buildRealURL(final String url, final String httpMethod, final String query) {
    if (httpMethod.equals(HttpMethod.GET.name()) && StringUtils.isNotBlank(query)) {
        return url + "?" + query;
    }
    return url;
}
```

最终放到上下文中供webClientPlugin调用后端服务。