## Soul源码解析之Websocket同步机制

经过前边几篇文章的demo运行和机制的初步解析，我们了解到了Soul的强大特性。我们发现，网关内部的插件信息和其他信息都依赖于控制台向网关同步数据，本篇文章将对Soul的WebSocket同步机制进行深入探讨。

## 什么时候会同步？

首先我们要了解，控制台什么时候会向网关同步信息。

第一，从前边文章可以看出，当网关层启动时，会从控制台拉取全量信息。

第二，有新的后端服务注册到控制台时，会向网关层同步增量服务信息。如果是http则同步后端地址等信息，如果是dubbo等rpc服务则同步元信息等数据。

第三，插件配置发生变化，也会向网关层同步增量信息。

## 同步机制解析

基于上边的思路，我们分别来看，网关数据到底是如何同步的。

### 启动时拉取全量信息

网关层配置文件中配置了admin的websocket地址：

```
soul :
    sync:
        websocket :
             urls: ws://localhost:9095/websocket
```

可以看出，这里的配置是通过spring-starter的机制进行加载的，因此可以看到soul-spring-boot-starter-sync-data-websocket包中只有一个配置类WebsocketSyncDataConfiguration。

```
@Configuration
@ConditionalOnClass(WebsocketSyncDataService.class)
@ConditionalOnProperty(prefix = "soul.sync.websocket", name = "urls")
@Slf4j
public class WebsocketSyncDataConfiguration {

    /**
     * Websocket sync data service.
     *
     * @param websocketConfig   the websocket config
     * @param pluginSubscriber the plugin subscriber
     * @param metaSubscribers   the meta subscribers
     * @param authSubscribers   the auth subscribers
     * @return the sync data service
     */
    @Bean
    public SyncDataService websocketSyncDataService(final ObjectProvider<WebsocketConfig> websocketConfig, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use websocket sync soul data.......");
        return new WebsocketSyncDataService(websocketConfig.getIfAvailable(WebsocketConfig::new), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    /**
     * Config websocket config.
     *
     * @return the websocket config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul.sync.websocket")
    public WebsocketConfig websocketConfig() {
        return new WebsocketConfig();
    }
}
```

从配置类可以看出，进行webSocket同步的主要功能都在WebsocketSyncDataService中。WebsocketSyncDataService通过bean注入拿到了插件订阅器、元信息订阅器和权限信息订阅器，并通过构造方法实例化。WebsocketSyncDataService的顶级接口是SyncDataService，是一个标志接口，没有接口方法。我们来看一下WebsocketSyncDataService的构造方法。

```
@Slf4j
public class WebsocketSyncDataService implements SyncDataService, AutoCloseable {

    private final List<WebSocketClient> clients = new ArrayList<>();

    private final ScheduledThreadPoolExecutor executor;
    
    /**
     * Instantiates a new Websocket sync cache.
     *
     * @param websocketConfig      the websocket config
     * @param pluginDataSubscriber the plugin data subscriber
     * @param metaDataSubscribers  the meta data subscribers
     * @param authDataSubscribers  the auth data subscribers
     */
    public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        for (String url : urls) {
            try {
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
            for (WebSocketClient client : clients) {
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
                executor.scheduleAtFixedRate(() -> {
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }
}
```

构造方法中，WebsocketSyncDataService拿到了我们在配置文件配置的url，并为每个地址声明了一个SoulWebsocketClient，并完成连接动作。这里Url支持多个，说明支持多个控制台。

SoulWebsocketClient继承了WebSocketClient类，并重写了onOpen和onMessage等方法。

```
@Slf4j
public final class SoulWebsocketClient extends WebSocketClient {
    
    private volatile boolean alreadySync = Boolean.FALSE;
    
    private final WebsocketDataHandler websocketDataHandler;
    
    /**
     * Instantiates a new Soul websocket client.
     *
     * @param serverUri             the server uri
     * @param pluginDataSubscriber the plugin data subscriber
     * @param metaDataSubscribers   the meta data subscribers
     * @param authDataSubscribers   the auth data subscribers
     */
    public SoulWebsocketClient(final URI serverUri, final PluginDataSubscriber pluginDataSubscriber,
                               final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        super(serverUri);
        this.websocketDataHandler = new WebsocketDataHandler(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
    }
    
    @Override
    public void onOpen(final ServerHandshake serverHandshake) {
        if (!alreadySync) {
            send(DataEventTypeEnum.MYSELF.name());
            alreadySync = true;
        }
    }
    
    @Override
    public void onMessage(final String result) {
        handleResult(result);
    }
}
```

可以看到，在连接打开的时候会发送一条消息给控制台。控制台和客户端交互时会回调onMeesage方法，然后处理全量数据。

我们先看一下服务端是怎么处理连接建立时的请求的。

```
@ServerEndpoint("/websocket")
public class WebsocketCollector {

    private static final Set<Session> SESSION_SET = new CopyOnWriteArraySet<>();

    private static final String SESSION_KEY = "sessionKey";

    /**
     * On message.
     *
     * @param message the message
     * @param session the session
     */
    @OnMessage
    public void onMessage(final String message, final Session session) {
        if (message.equals(DataEventTypeEnum.MYSELF.name())) {
            try {
                ThreadLocalUtil.put(SESSION_KEY, session);
                SpringBeanUtils.getInstance().getBean(SyncDataService.class).syncAll(DataEventTypeEnum.MYSELF);
            } finally {
                ThreadLocalUtil.clear();
            }
        }
    }
}
```

可以看到假如接收到了DataEventTypeEnum.MYSELF.name()就调用SyncDataService的全量同步方法。

```
@Service("syncDataService")
public class SyncDataServiceImpl implements SyncDataService {

    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
        appAuthService.syncData();
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        List<SelectorData> selectorDataList = selectorService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
        List<RuleData> ruleDataList = ruleService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
        metaDataService.syncData();
        return true;
    }
}
```

可以看到服务端从数据库取出了全量数据，并通过DataChangedEvent进行发布。SyncDataServiceImpl就是控制台用来发布数据更新事件的。

DataChangedEventDispatcher作为监听者监听DataChangedEvent事件，通过容器中拿到了所有的listener的bean，但是由于事件相同，需要DataChangedEventDispatcher通过事件类型不同做一次分发。

```
@Component
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {

    private ApplicationContext applicationContext;

    private List<DataChangedListener> listeners;

    public DataChangedEventDispatcher(final ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH:
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN:
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR:
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA:
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }

    @Override
    public void afterPropertiesSet() {
        Collection<DataChangedListener> listenerBeans = applicationContext.getBeansOfType(DataChangedListener.class).values();
        this.listeners = Collections.unmodifiableList(new ArrayList<>(listenerBeans));
    }

}
```

从上面的代码可以看出，是由listener去完成实际的信息同步工作。WebsocketDataChangedListener则完成最终的数据同步工作，调用WebsocketCollector发送数据。

```
public class WebsocketDataChangedListener implements DataChangedListener {

    @Override
    public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
        WebsocketData<PluginData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }

    @Override
    public void onSelectorChanged(final List<SelectorData> selectorDataList, final DataEventTypeEnum eventType) {
        WebsocketData<SelectorData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.SELECTOR.name(), eventType.name(), selectorDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }

    @Override
    public void onRuleChanged(final List<RuleData> ruleDataList, final DataEventTypeEnum eventType) {
        WebsocketData<RuleData> configData =
                new WebsocketData<>(ConfigGroupEnum.RULE.name(), eventType.name(), ruleDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
    }

    @Override
    public void onAppAuthChanged(final List<AppAuthData> appAuthDataList, final DataEventTypeEnum eventType) {
        WebsocketData<AppAuthData> configData =
                new WebsocketData<>(ConfigGroupEnum.APP_AUTH.name(), eventType.name(), appAuthDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
    }

    @Override
    public void onMetaDataChanged(final List<MetaData> metaDataList, final DataEventTypeEnum eventType) {
        WebsocketData<MetaData> configData =
                new WebsocketData<>(ConfigGroupEnum.META_DATA.name(), eventType.name(), metaDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
    }

}
```

这里就走到了前边的SoulWebsocketClient的onMesage回调方法，回调方法通过handleResult处理网关内部信息同步。


    @SuppressWarnings("ALL")
    private void handleResult(final String result) {
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
SoulWebsocketClient调用websocketDataHandler完成缓存加载，

```
public class WebsocketDataHandler {

    private static final EnumMap<ConfigGroupEnum, DataHandler> ENUM_MAP = new EnumMap<>(ConfigGroupEnum.class);

    /**
     * Instantiates a new Websocket data handler.
     *
     * @param pluginDataSubscriber the plugin data subscriber
     * @param metaDataSubscribers  the meta data subscribers
     * @param authDataSubscribers  the auth data subscribers
     */
    public WebsocketDataHandler(final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers,
                                final List<AuthDataSubscriber> authDataSubscribers) {
        ENUM_MAP.put(ConfigGroupEnum.PLUGIN, new PluginDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.SELECTOR, new SelectorDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.RULE, new RuleDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.APP_AUTH, new AuthDataHandler(authDataSubscribers));
        ENUM_MAP.put(ConfigGroupEnum.META_DATA, new MetaDataHandler(metaDataSubscribers));
    }

    /**
     * Executor.
     *
     * @param type      the type
     * @param json      the json
     * @param eventType the event type
     */
    public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
}
```

websocketDataHandler通过分发的模式，从事件类型判断使用对应的DataHandler完成操作

AbstractDataHandler的handle方法调用了模板方法，定义了更新流程。

```
@Override
public void handle(final String json, final String eventType) {
    List<T> dataList = convert(json);
    if (CollectionUtils.isNotEmpty(dataList)) {
        DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
        switch (eventTypeEnum) {
            case REFRESH:
            case MYSELF:
                doRefresh(dataList);
                break;
            case UPDATE:
            case CREATE:
                doUpdate(dataList);
                break;
            case DELETE:
                doDelete(dataList);
                break;
            default:
                break;
        }
    }
}
```

每个DataHandler都持有对应的DataSubsriber，通过对应的订阅器最终更新缓存。订阅器有三个顶级接口，如下所示：

```
AuthDataSubscriber
MetaDataSubscriber
PluginDataSubscriber
```

其中AuthDataSubscriber实现类SignAuthDataSubscriber实现了SignAuthDataCache缓存的操作。PluginDataSubscriber的实现类CommonPluginDataSubscriber则完成了插件数据的同步。

```
private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
    Optional.ofNullable(classData).ifPresent(data -> {
        if (data instanceof PluginData) {
            PluginData pluginData = (PluginData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                BaseDataCache.getInstance().cachePluginData(pluginData);
                Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removePluginData(pluginData);
                Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
            }
        } else if (data instanceof SelectorData) {
            SelectorData selectorData = (SelectorData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                BaseDataCache.getInstance().cacheSelectData(selectorData);
                Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removeSelectData(selectorData);
                Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
            }
        } else if (data instanceof RuleData) {
            RuleData ruleData = (RuleData) data;
            if (dataType == DataEventTypeEnum.UPDATE) {
                BaseDataCache.getInstance().cacheRuleData(ruleData);
                Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
            } else if (dataType == DataEventTypeEnum.DELETE) {
                BaseDataCache.getInstance().removeRuleData(ruleData);
                Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
            }
        }
    });
}
```

MetaDataSubscriber则根据元信息分类的不同，有以下几个实现类。

```
AlibabaDubboMetaDataSubscriber
ApacheDubboMetaDataSubscriber
MetaDataAllSubscriber
SofaMetaDataSubscriber
TarsMetaDataSubscriber
```

这几个实现类都是通过onSubscribe方法完成的元信息的同步。



### 增量同步信息

增量同步信息则是通过controller接口调用事件发布者，后面流程和全量是一致的。



## 总结

通过以上对数据同步的流程跟踪，全量信息同步是从网关层通过ws发起的同步请求，增量信息同步则是通过controller发起的同步请求。二者都通过SyncDataService发布DataChangedEvent事件，监听者拿到事件通过事件类型进行调用listener的方法，listenter实际向网关发送ws请求。网关收到请求之后，通过DataHandler按照事件类型分别进行处理，每个DataHandler都持有相应的DataSubsriber，数据同步的真正操作正是由DataSubsriber完成的。