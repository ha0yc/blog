## Soul源码解析之集群同步机制

经过前边几篇文章，我们了解到了Soul是如何通过各种方式对控制台信息进行同步的。但是之前的控制台都是以单机的形式配置的，本篇文章将探究在多控制台情况下，是如何进行数据同步的。我们已经知道了，zk、erureka、nacos都是通过注册中心和远程配置中心进行同步的，所以网关和admin是解耦的，不需要在本地配置中配置多个控制台地址。所以我们只需要看websocket和http长轮询机制下是如何进行多admin同步的。

## websocket下的集群同步机制

老规矩，还是先看一下soul-spring-boot-starter-sync-data-websocket包里的配置类WebsocketSyncDataConfiguration。

```
https://www.ha0yc.top/gao-xing-neng-wang-guan-soulshen-ru-jie-xi-10-adminji-qun-tong-bu-ji-zhi/
```

我们的数据被转换成了pojo类WebsocketConfig。

```
public class WebsocketConfig {
    
    /**
     * if have more soul admin url,please config like this.
     * 127.0.0.1:8888,127.0.0.1:8889
     */
    private String urls;
}
```

可以看到，属性为urls，即支持多个控制台地址。WebsocketConfig拿到后被来完成WebsocketSyncDataService的实例化。

```
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
```

在WebsocketSyncDataService的构造方法中，可以看到每一个url都被声明了一个SoulWebsocketClient，然后被WebsocketSyncDataService以列表形式持有。在前边我们知道了，websocket是通过实现WebSocketClient接口的回调方法实现的连接监听并处理的数据。

## Http长轮询下的集群同步机制

首先来看soul-spring-boot-starter-sync-data-http包下面的配置类HttpSyncDataConfiguration。

```
@Configuration
@ConditionalOnClass(HttpSyncDataService.class)
@ConditionalOnProperty(prefix = "soul.sync.http", name = "url")
@Slf4j
public class HttpSyncDataConfiguration {

    /**
     * Http sync data service.
     *
     * @param httpConfig        the http config
     * @param pluginSubscriber the plugin subscriber
     * @param metaSubscribers   the meta subscribers
     * @param authSubscribers   the auth subscribers
     * @return the sync data service
     */
    @Bean
    public SyncDataService httpSyncDataService(final ObjectProvider<HttpConfig> httpConfig, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use http long pull sync soul data");
        return new HttpSyncDataService(Objects.requireNonNull(httpConfig.getIfAvailable()), Objects.requireNonNull(pluginSubscriber.getIfAvailable()),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    /**
     * Http config http config.
     *
     * @return the http config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul.sync.http")
    public HttpConfig httpConfig() {
        return new HttpConfig();
    }
}
```

配置被转化成了HttpConfig的实例。

```
@Data
public class HttpConfig {
    
    private String url;
    
    private Integer delayTime;
    
    private Integer connectionTimeout;
}
```

HttpConfig实例化后用来完成HttpSyncDataService的实例化，我们看下HttpSyncDataService的构造方法。

```
@Slf4j
public class HttpSyncDataService implements SyncDataService, AutoCloseable {

    private static final AtomicBoolean RUNNING = new AtomicBoolean(false);

    private static final Gson GSON = new Gson();

    /**
     * default: 10s.
     */
    private Duration connectionTimeout = Duration.ofSeconds(10);

    /**
     * only use for http long polling.
     */
    private RestTemplate httpClient;

    private ExecutorService executor;

    private HttpConfig httpConfig;

    private List<String> serverList;

    private DataRefreshFactory factory;

    public HttpSyncDataService(final HttpConfig httpConfig, final PluginDataSubscriber pluginDataSubscriber,
                               final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.factory = new DataRefreshFactory(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        this.httpConfig = httpConfig;
        this.serverList = Lists.newArrayList(Splitter.on(",").split(httpConfig.getUrl()));
        this.httpClient = createRestTemplate();
        this.start();
    }

    private void start() {
        // It could be initialized multiple times, so you need to control that.
        if (RUNNING.compareAndSet(false, true)) {
            // fetch all group configs.
            this.fetchGroupConfig(ConfigGroupEnum.values());
            int threadSize = serverList.size();
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // start long polling, each server creates a thread to listen for changes.
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
        } else {
            log.info("soul http long polling was started, executor=[{}]", executor);
        }
    }

    private void fetchGroupConfig(final ConfigGroupEnum... groups) throws SoulException {
        for (int index = 0; index < this.serverList.size(); index++) {
            String server = serverList.get(index);
            try {
                this.doFetchGroupConfig(server, groups);
                break;
            } catch (SoulException e) {
                // no available server, throw exception.
                if (index >= serverList.size() - 1) {
                    throw e;
                }
                log.warn("fetch config fail, try another one: {}", serverList.get(index + 1));
            }
        }
    }    
}
```

构造方法通过url属性获取了server list，然后调用了start方法，start的方法调用fetchGroupConfig，并遍历serverList启动了轮询控制台线程。fetchGroupConfig完成了从各个server拉取数据的行为。