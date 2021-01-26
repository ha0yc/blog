## Soul源码解析之长轮询同步机制

经过前边几篇文章的demo运行和机制的初步解析，我们了解到了Soul的强大特性。我们发现，网关内部的插件信息和其他信息都依赖于控制台向网关同步数据，本篇文章将对Soul的长轮询同步机制进行深入探讨。

## 如何使用配置长轮询同步机制

网关增加soul-spring-boot-starter-sync-data-http的依赖，增加以下依赖：

```
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
  <version>${project.version}</version>
</dependency>
```

同时增加配置：

```
soul :
    sync:
        http:
             url : http://localhost:9095
```

## 同步机制解析

### 网关侧同步机制

从配置可以看出，配置是通过spring-starter的机制进行加载的，因此可以看到soul-spring-boot-starter-sync-data-http包中只有一个bean配置类HttpSyncDataConfiguration。

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

从以上代码可以看出，生成了一个HttpSyncDataService对象。HttpSyncDataService通过bean注入拿到了插件订阅器、元信息订阅器和权限信息订阅器，并通过构造方法实例化。HttpSyncDataService的顶级接口是SyncDataService，是一个标志接口，没有接口方法。我们来看一下HttpSyncDataService的构造方法。

```
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
```

在构造器中，完成环境配置后调用了start方法。start方法中，将RUNNING状态通过cas设置为true，成功的话就认为是第一次同步，执行fetchGroupConfig方法并为每个admin server启动一个轮询线程HttpLongPollingTask。fetchGroupConfig实际上对每个server执行了doFetchGroupConfig方法。

```
private void doFetchGroupConfig(final String server, final ConfigGroupEnum... groups) {
    StringBuilder params = new StringBuilder();
    for (ConfigGroupEnum groupKey : groups) {
        params.append("groupKeys").append("=").append(groupKey.name()).append("&");
    }
    String url = server + "/configs/fetch?" + StringUtils.removeEnd(params.toString(), "&");
    log.info("request configs: [{}]", url);
    String json = null;
    try {
        json = this.httpClient.getForObject(url, String.class);
    } catch (RestClientException e) {
        String message = String.format("fetch config fail from server[%s], %s", url, e.getMessage());
        log.warn(message);
        throw new SoulException(message, e);
    }
    // update local cache
    boolean updated = this.updateCacheWithJson(json);
    if (updated) {
        log.info("get latest configs: [{}]", json);
        return;
    }
    // not updated. it is likely that the current config server has not been updated yet. wait a moment.
    log.info("The config of the server[{}] has not been updated or is out of date. Wait for 30s to listen for changes again.", server);
    ThreadUtils.sleep(TimeUnit.SECONDS, 30);
}
```

可以看到，doFetchGroupConfig方法中通过/configs/fetch获取到了全量的配置数据，并通过updateCacheWithJson更新到本地缓存中。updateCacheWithJson调用了DataRefreshFactory的executor方法，通过dataRefresh接口将数据写入到缓存中，DataFresh的各个实现类则持有对应的DataSubscriber，这里就进入到了之前讲的数据更新流程中。

```
public boolean executor(final JsonObject data) {
    final boolean[] success = {false};
    ENUM_MAP.values().parallelStream().forEach(dataRefresh -> success[0] = dataRefresh.refresh(data));
    return success[0];
}
```

fetchGroupConfig在网关启动的时候从admin中拉取了全量数据。随后又执行了轮询线程轮询每个服务器的数据变换。线程循环判断RUNNING的值，如果拉取不报错，就循环拉取，报错的话就sleep5分钟再重试。拉取到数据之后，就是走doFetchGroupConfig方法，走上面的流程进行本地缓存更新。

```
class HttpLongPollingTask implements Runnable {

    private String server;

    private final int retryTimes = 3;

    HttpLongPollingTask(final String server) {
        this.server = server;
    }

    @Override
    public void run() {
        while (RUNNING.get()) {
            for (int time = 1; time <= retryTimes; time++) {
                try {
                    doLongPolling(server);
                } catch (Exception e) {
                    // print warnning log.
                    if (time < retryTimes) {
                        log.warn("Long polling failed, tried {} times, {} times left, will be suspended for a while! {}",
                                time, retryTimes - time, e.getMessage());
                        ThreadUtils.sleep(TimeUnit.SECONDS, 5);
                        continue;
                    }
                    // print error, then suspended for a while.
                    log.error("Long polling failed, try again after 5 minutes!", e);
                    ThreadUtils.sleep(TimeUnit.MINUTES, 5);
                }
            }
        }
        log.warn("Stop http long polling.");
    }
}
private void doLongPolling(final String server) {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>(8);
        for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
            ConfigData<?> cacheConfig = factory.cacheConfigData(group);
            String value = String.join(",", cacheConfig.getMd5(), String.valueOf(cacheConfig.getLastModifyTime()));
            params.put(group.name(), Lists.newArrayList(value));
        }
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        HttpEntity httpEntity = new HttpEntity(params, headers);
        String listenerUrl = server + "/configs/listener";
        log.debug("request listener configs: [{}]", listenerUrl);
        JsonArray groupJson = null;
        try {
            String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
            log.debug("listener result: [{}]", json);
            groupJson = GSON.fromJson(json, JsonObject.class).getAsJsonArray("data");
        } catch (RestClientException e) {
            String message = String.format("listener configs fail, server:[%s], %s", server, e.getMessage());
            throw new SoulException(message, e);
        }
        if (groupJson != null) {
            // fetch group configuration async.
            ConfigGroupEnum[] changedGroups = GSON.fromJson(groupJson, ConfigGroupEnum[].class);
            if (ArrayUtils.isNotEmpty(changedGroups)) {
                log.info("Group config changed: {}", Arrays.toString(changedGroups));
                this.doFetchGroupConfig(server, changedGroups);
            }
        }
    }
```

### 控制台侧同步机制





用户在网页端做的数据变更的操作都是通过controller接口同步到后端的，而数据的变化都是通过DataChangedEvent向后传递的。正如上篇所讲，DataChangedEventDispatcher作为监听器对事件进行处理分发，由真正的listenter去完成数据同步的操作。

在长轮询的机制下，是HttpLongPollingDataChangedListener通过异步请求向各网关应用发送数据。

```
public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {

    // compare group md5
    List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
    String clientIp = getRemoteIp(request);

    // response immediately.
    if (CollectionUtils.isNotEmpty(changedGroup)) {
        this.generateResponse(response, changedGroup);
        log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
        return;
    }

    // listen for configuration changed.
    final AsyncContext asyncContext = request.startAsync();

    // AsyncContext.settimeout() does not timeout properly, so you have to control it yourself
    asyncContext.setTimeout(0L);

    // block client's thread.
    scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
}

@Override
protected void afterAppAuthChanged(final List<AppAuthData> changed, final DataEventTypeEnum eventType) {
    scheduler.execute(new DataChangeTask(ConfigGroupEnum.APP_AUTH));
}

@Override
protected void afterMetaDataChanged(final List<MetaData> changed, final DataEventTypeEnum eventType) {
    scheduler.execute(new DataChangeTask(ConfigGroupEnum.META_DATA));
}

@Override
protected void afterPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
    scheduler.execute(new DataChangeTask(ConfigGroupEnum.PLUGIN));
}

@Override
protected void afterRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
    scheduler.execute(new DataChangeTask(ConfigGroupEnum.RULE));
}

@Override
protected void afterSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
    scheduler.execute(new DataChangeTask(ConfigGroupEnum.SELECTOR));
}
```

可以看出，数据变化都是通过DataChangeTask传递的。

```
class DataChangeTask implements Runnable {

    /**
     * The Group where the data has changed.
     */
    private final ConfigGroupEnum groupKey;

    /**
     * The Change time.
     */
    private final long changeTime = System.currentTimeMillis();

    /**
     * Instantiates a new Data change task.
     *
     * @param groupKey the group key
     */
    DataChangeTask(final ConfigGroupEnum groupKey) {
        this.groupKey = groupKey;
    }

    @Override
    public void run() {
        for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
            LongPollingClient client = iter.next();
            iter.remove();
            client.sendResponse(Collections.singletonList(groupKey));
            log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
        }
    }
}

class LongPollingClient implements Runnable {

        /**
         * The Async context.
         */
        private final AsyncContext asyncContext;

        /**
         * The Ip.
         */
        private final String ip;

        /**
         * The Timeout time.
         */
        private final long timeoutTime;

        /**
         * The Async timeout future.
         */
        private Future<?> asyncTimeoutFuture;

        /**
         * Instantiates a new Long polling client.
         *
         * @param ac          the ac
         * @param ip          the ip
         * @param timeoutTime the timeout time
         */
        LongPollingClient(final AsyncContext ac, final String ip, final long timeoutTime) {
            this.asyncContext = ac;
            this.ip = ip;
            this.timeoutTime = timeoutTime;
        }

        @Override
        public void run() {
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                clients.remove(LongPollingClient.this);
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            clients.add(this);
        }

        /**
         * Send response.
         *
         * @param changedGroups the changed groups
         */
        void sendResponse(final List<ConfigGroupEnum> changedGroups) {
            // cancel scheduler
            if (null != asyncTimeoutFuture) {
                asyncTimeoutFuture.cancel(false);
            }
            generateResponse((HttpServletResponse) asyncContext.getResponse(), changedGroups);
            asyncContext.complete();
        }
    }
```

DataChangeTask拿到了当前所有的网关持有的ip，并通过LongPollingClient发送了异步的response供网关轮询时读取，避免了同时请求时进行大量的同步处理。

