## Soul源码解析之Nacos同步机制

经过前边几篇文章的demo运行和机制的初步解析，我们了解到了Soul的强大特性。我们发现，网关内部的插件信息和其他信息都依赖于控制台向网关同步数据，本篇文章将对Soul的Nacos同步机制进行深入探讨。

## 如何使用配置Nacos同步机制

网关增加soul-spring-boot-starter-sync-data-http的依赖，增加以下依赖：

```
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-sync-data-nacos</artifactId>
  <version>${project.version}</version>
</dependency>
```

同时增加配置：

```
soul :
    sync:
      nacos:
        url: localhost:8848
        namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
        acm:
          enabled: false
          endpoint: acm.aliyun.com
          namespace:
          accessKey:
          secretKey:
```

## 同步机制解析

### 网关侧同步机制

从配置可以看出，配置是通过spring-starter的机制进行加载的，因此可以看到soul-spring-boot-starter-sync-data-nacos包中只有一个bean配置类NacosSyncDataConfiguration。

```
@Configuration
@ConditionalOnClass(NacosSyncDataService.class)
@ConditionalOnProperty(prefix = "soul.sync.nacos", name = "url")
@Slf4j
public class NacosSyncDataConfiguration {

    /**
     * Nacos sync data service.
     *
     * @param configService     the config service
     * @param pluginSubscriber the plugin subscriber
     * @param metaSubscribers   the meta subscribers
     * @param authSubscribers   the auth subscribers
     * @return the sync data service
     */
    @Bean
    public SyncDataService nacosSyncDataService(final ObjectProvider<ConfigService> configService, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use nacos sync soul data.......");
        return new NacosSyncDataService(configService.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    /**
     * Nacos config service config service.
     *
     * @param nacosConfig the nacos config
     * @return the config service
     * @throws Exception the exception
     */
    @Bean
    public ConfigService nacosConfigService(final NacosConfig nacosConfig) throws Exception {
        Properties properties = new Properties();
        if (nacosConfig.getAcm() != null && nacosConfig.getAcm().isEnabled()) {
            properties.put(PropertyKeyConst.ENDPOINT, nacosConfig.getAcm().getEndpoint());
            properties.put(PropertyKeyConst.NAMESPACE, nacosConfig.getAcm().getNamespace());
            properties.put(PropertyKeyConst.ACCESS_KEY, nacosConfig.getAcm().getAccessKey());
            properties.put(PropertyKeyConst.SECRET_KEY, nacosConfig.getAcm().getSecretKey());
        } else {
            properties.put(PropertyKeyConst.SERVER_ADDR, nacosConfig.getUrl());
            properties.put(PropertyKeyConst.NAMESPACE, nacosConfig.getNamespace());
        }
        return NacosFactory.createConfigService(properties);
    }

    /**
     * Http config http config.
     *
     * @return the http config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul.sync.nacos")
    public NacosConfig nacosConfig() {
        return new NacosConfig();
    }
}
```

以上代码声明了ConfigService的实例，即nacos的客户端，其中包含服务器地址和namespace。另外声明了一个NacosSyncDataService对象。NacosSyncDataService通过bean注入拿到了插件订阅器、元信息订阅器和权限信息订阅器，并通过构造方法实例化。NacosSyncDataService的顶级接口是SyncDataService，是一个标志接口，没有接口方法。我们来看一下NacosSyncDataService的构造方法。NacosSyncDataService的父类是NacosCacheHandler，NacosCacheHandler实际上完成了nacos的配置更新的主要流程，而NacosSyncDataService主要起配置作用。

```
	public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        start();
    }
    
    /**
     * Start.
     */
    public void start() {
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }
    
    protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        oc.change(getConfigAndSignListener(dataId, listener));
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }
```

在构造器中，完成环境配置后调用了start方法。start方法中，通过watcherData方法注册了各类数据的实际更新动作。

```
protected void updatePluginMap(final String configInfo) {
        try {
            // Fix bug #656(https://github.com/dromara/soul/issues/656)
            List<PluginData> pluginDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, PluginData.class).values());
            pluginDataList.forEach(pluginData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSubscribe(pluginData);
                subscriber.onSubscribe(pluginData);
            }));
        } catch (JsonParseException e) {
            log.error("sync plugin data have error:", e);
        }
    }

    protected void updateSelectorMap(final String configInfo) {
        try {
            List<SelectorData> selectorDataList = GsonUtils.getInstance().toObjectMapList(configInfo, SelectorData.class).values().stream().flatMap(Collection::stream).collect(Collectors.toList());
            selectorDataList.forEach(selectorData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSelectorSubscribe(selectorData);
                subscriber.onSelectorSubscribe(selectorData);
            }));
        } catch (JsonParseException e) {
            log.error("sync selector data have error:", e);
        }
    }

    protected void updateRuleMap(final String configInfo) {
        try {
            List<RuleData> ruleDataList = GsonUtils.getInstance().toObjectMapList(configInfo, RuleData.class).values()
                    .stream().flatMap(Collection::stream)
                    .collect(Collectors.toList());
            ruleDataList.forEach(ruleData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unRuleSubscribe(ruleData);
                subscriber.onRuleSubscribe(ruleData);
            }));
        } catch (JsonParseException e) {
            log.error("sync rule data have error:", e);
        }
    }

    protected void updateMetaDataMap(final String configInfo) {
        try {
            List<MetaData> metaDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, MetaData.class).values());
            metaDataList.forEach(metaData -> metaDataSubscribers.forEach(subscriber -> {
                subscriber.unSubscribe(metaData);
                subscriber.onSubscribe(metaData);
            }));
        } catch (JsonParseException e) {
            log.error("sync meta data have error:", e);
        }
    }

    protected void updateAuthMap(final String configInfo) {
        try {
            List<AppAuthData> appAuthDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, AppAuthData.class).values());
            appAuthDataList.forEach(appAuthData -> authDataSubscribers.forEach(subscriber -> {
                subscriber.unSubscribe(appAuthData);
                subscriber.onSubscribe(appAuthData);
            }));
        } catch (JsonParseException e) {
            log.error("sync auth data have error:", e);
        }
    }
```

可以看到，各个方法实际上都拿到了DataSubscriber，进入到了之前讲的数据更新流程中。控制台侧同步机制

### 控制台侧数据同步流程

用户在网页端做的数据变更的操作都是通过controller接口同步到后端的，而数据的变化都是通过DataChangedEvent向后传递的。正如上篇所讲，DataChangedEventDispatcher作为监听器对事件进行处理分发，由真正的listenter去完成数据同步的操作。

在Nacos的机制下，是NacosDataChangedListener向nacos写入的数据。NacosDataChangedListener实现了DataChangedListener的方法，拿到了DataChangedEvent整理数据，并通过publishConfig方法写到了nacos中。

```
public class NacosDataChangedListener implements DataChangedListener {

    public NacosDataChangedListener(final ConfigService configService) {
        this.configService = configService;
    }

    private void updateAuthMap(final String configInfo) {
        JsonObject jo = GsonUtils.getInstance().fromJson(configInfo, JsonObject.class);
        Set<String> set = new HashSet<>(AUTH_MAP.keySet());
        for (Entry<String, JsonElement> e : jo.entrySet()) {
            set.remove(e.getKey());
            AUTH_MAP.put(e.getKey(), GsonUtils.getInstance().fromJson(e.getValue(), AppAuthData.class));
        }
        AUTH_MAP.keySet().removeAll(set);
    }

    @SneakyThrows
    private String getConfig(final String dataId) {
        String config = configService.getConfig(dataId, GROUP, 6000);
        return StringUtils.hasLength(config) ? config : EMPTY_CONFIG_DEFAULT_VALUE;
    }

    @SneakyThrows
    private void publishConfig(final String dataId, final Object data) {
        configService.publishConfig(dataId, GROUP, GsonUtils.getInstance().toJson(data));
    }

    @Override
    @SneakyThrows
    public void onAppAuthChanged(final List<AppAuthData> changed, final DataEventTypeEnum eventType) {
        updateAuthMap(getConfig(AUTH_DATA_ID));
        switch (eventType) {
            case DELETE:
                changed.forEach(appAuth -> AUTH_MAP.remove(appAuth.getAppKey()));
                break;
            case REFRESH:
            case MYSELF:
                Set<String> set = new HashSet<>(AUTH_MAP.keySet());
                changed.forEach(appAuth -> {
                    set.remove(appAuth.getAppKey());
                    AUTH_MAP.put(appAuth.getAppKey(), appAuth);
                });
                AUTH_MAP.keySet().removeAll(set);
                break;
            default:
                changed.forEach(appAuth -> AUTH_MAP.put(appAuth.getAppKey(), appAuth));
                break;
        }
        publishConfig(AUTH_DATA_ID, AUTH_MAP);
    }

}
```

以上以auth数据为例，onAppAuthChanged方法拿到数据后，根据操作类型修改本地缓存，然后将全量缓存刷新到了nacos上。