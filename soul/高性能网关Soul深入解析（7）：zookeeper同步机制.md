## Soul源码解析之Zookeeper同步机制

经过前边几篇文章的demo运行和机制的初步解析，我们了解到了Soul的强大特性。我们发现，网关内部的插件信息和其他信息都依赖于控制台向网关同步数据，本篇文章将对Soul的Zookeeper同步机制进行深入探讨。

## 如何使用配置Zookeeper同步机制

控制台配置增加zookeeper配置：

```
soul:
sync:
  zookeeper:
      url: localhost:2181
      sessionTimeout: 5000
      connectionTimeout: 2000
```

网关去掉soul-spring-boot-starter-sync-data-websocket的依赖，增加以下依赖：

```
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
  <version>${project.version}</version>
</dependency>
```

同时增加配置：

```
soul:
sync:
  zookeeper:
      url: localhost:2181
      sessionTimeout: 5000
      connectionTimeout: 2000
```

## 同步机制解析

### 网关侧同步机制

从配置可以看出，配置是通过spring-starter的机制进行加载的，因此可以看到soul-spring-boot-starter-sync-data-zookeeper包中只有一个bean配置类ZookeeperSyncDataConfiguration。

```
@Configuration
@ConditionalOnClass(ZookeeperSyncDataService.class)
@ConditionalOnProperty(prefix = "soul.sync.zookeeper", name = "url")
@EnableConfigurationProperties(ZookeeperConfig.class)
@Slf4j
public class ZookeeperSyncDataConfiguration {

  /**
    * Sync data service sync data service.
    *
    * @param zkClient         the zk client
    * @param pluginSubscriber the plugin subscriber
    * @param metaSubscribers   the meta subscribers
    * @param authSubscribers   the auth subscribers
    * @return the sync data service
    */
  @Bean
  public SyncDataService syncDataService(final ObjectProvider<ZkClient> zkClient, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                          final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
      log.info("you use zookeeper sync soul data.......");
      return new ZookeeperSyncDataService(zkClient.getIfAvailable(), pluginSubscriber.getIfAvailable(),
              metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
  }

  /**
    * register zkClient in spring ioc.
    *
    * @param zookeeperConfig the zookeeper configuration
    * @return ZkClient {@linkplain ZkClient}
    */
  @Bean
  public ZkClient zkClient(final ZookeeperConfig zookeeperConfig) {
      return new ZkClient(zookeeperConfig.getUrl(), zookeeperConfig.getSessionTimeout(), zookeeperConfig.getConnectionTimeout());
  }
}
```

从以上代码可以看出，首先通过starter配置声明了一个zkClient实例，然后生成了一个ZookeeperSyncDataService对象。ZookeeperSyncDataService通过bean注入拿到了插件订阅器、元信息订阅器和权限信息订阅器，并通过构造方法实例化。ZookeeperSyncDataService的顶级接口是SyncDataService，是一个标志接口，没有接口方法。我们来看一下ZookeeperSyncDataService的构造方法。

```
public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                  final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
      this.zkClient = zkClient;
      this.pluginDataSubscriber = pluginDataSubscriber;
      this.metaDataSubscribers = metaDataSubscribers;
      this.authDataSubscribers = authDataSubscribers;
      watcherData();
      watchAppAuth();
      watchMetaData();
  }
```

构造方法中，WebsocketSyncDataService拿到了我们声明的zkClient，并完成了watcherData、watchAppAuth和watchMetaData三个行为。

首先看watcherData：

```
private void watcherData() {
      final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
      List<String> pluginZKs = zkClientGetChildren(pluginParent);
      for (String pluginName : pluginZKs) {
          watcherAll(pluginName);
      }
      zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
          if (CollectionUtils.isNotEmpty(currentChildren)) {
              for (String pluginName : currentChildren) {
                  watcherAll(pluginName);
              }
          }
      });
  }
   
  private void watcherAll(final String pluginName) {
      watcherPlugin(pluginName);
      watcherSelector(pluginName);
      watcherRule(pluginName);
  }
```

插件数据的父节点在zk上是/soul/plugin，client拿到了/soul/plugin的每一个子节点，每个子节点对应一个插件，然后遍历调用watcherAll方法。watcherAll里面又分别调用了watcherPlugin、watcherSelector和watcherRule三个方法，分别针对插件获取到了对应的插件、selector和rule数据。

先看watcherPlugin方法，首先得到插件路径/soul/plugin/{pluginName}，通过路径读取到插件数据PluginData，再通过CommonPluginDataSubscriber加载到内存中。

加载完之后调用subscribePluginDataChanges方法，让zkClient监听这一节点的数据变化和删除。

```
private void watcherPlugin(final String pluginName) {
  String pluginPath = ZkPathConstants.buildPluginPath(pluginName);
  if (!zkClient.exists(pluginPath)) {
      zkClient.createPersistent(pluginPath, true);
  }
  cachePluginData(zkClient.readData(pluginPath));
  subscribePluginDataChanges(pluginPath, pluginName);
}
   
  private void cachePluginData(final PluginData pluginData) {
      Optional.ofNullable(pluginData).flatMap(data -> Optional.ofNullable(pluginDataSubscriber)).ifPresent(e -> e.onSubscribe(pluginData));
  }
   
  private void subscribePluginDataChanges(final String pluginPath, final String pluginName) {
      zkClient.subscribeDataChanges(pluginPath, new IZkDataListener() {

          @Override
          public void handleDataChange(final String dataPath, final Object data) {
              Optional.ofNullable(data)
                      .ifPresent(d -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSubscribe((PluginData) d)));
          }

          @Override
          public void handleDataDeleted(final String dataPath) {
              final PluginData data = new PluginData();
              data.setName(pluginName);
              Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.unSubscribe(data));
          }
      });
  }
```

同理watcherSelector和watcherRule也是通过同样的方式进行同步和监听变化的。由于selector和rule在zk上和plugin是解耦的，不属于同一个节点。所以是要分别进行同步的，但是最终都调用了CommonPluginDataSubscriber的subscribeDataHandler方法加载到缓存中。

watcherSelector和watcherRule还调用了subscribeChildChanges监听孩子节点的变化，因为rule和selector是以列表形式存在的。

```
private void subscribeChildChanges(final ConfigGroupEnum groupKey, final String groupParentPath, final List<String> childrenList) {
  switch (groupKey) {
      case SELECTOR:
          zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
              if (CollectionUtils.isNotEmpty(currentChildren)) {
                  List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                  addSubscribePath.stream().map(addPath -> {
                      String realPath = buildRealPath(parentPath, addPath);
                      cacheSelectorData(zkClient.readData(realPath));
                      return realPath;
                  }).forEach(this::subscribeSelectorDataChanges);

              }
          });
          break;
      case RULE:
          zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
              if (CollectionUtils.isNotEmpty(currentChildren)) {
                  List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                  // Get the newly added node data and subscribe to that node
                  addSubscribePath.stream().map(addPath -> {
                      String realPath = buildRealPath(parentPath, addPath);
                      cacheRuleData(zkClient.readData(realPath));
                      return realPath;
                  }).forEach(this::subscribeRuleDataChanges);
              }
          });
          break;
      case APP_AUTH:
          zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
              if (CollectionUtils.isNotEmpty(currentChildren)) {
                  final List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                  addSubscribePath.stream().map(children -> {
                      final String realPath = buildRealPath(parentPath, children);
                      cacheAuthData(zkClient.readData(realPath));
                      return realPath;
                  }).forEach(this::subscribeAppAuthDataChanges);
              }
          });
          break;
      case META_DATA:
          zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
              if (CollectionUtils.isNotEmpty(currentChildren)) {
                  final List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                  addSubscribePath.stream().map(children -> {
                      final String realPath = buildRealPath(parentPath, children);
                      cacheMetaData(zkClient.readData(realPath));
                      return realPath;
                  }).forEach(this::subscribeMetaDataChanges);
              }
          });
          break;
      default:
          throw new IllegalStateException("Unexpected groupKey: " + groupKey);
  }
}
```

看完了插件数据的加载，我们来看watchAppAuth方法。

同理，也是读取了auth节点的数据，通过SignAuthDataSubscriber缓存auth数据，并监听孩子节点的变化。

```
private void watchAppAuth() {
      final String appAuthParent = ZkPathConstants.APP_AUTH_PARENT;
      List<String> childrenList = zkClientGetChildren(appAuthParent);
      if (CollectionUtils.isNotEmpty(childrenList)) {
          childrenList.forEach(children -> {
              String realPath = buildRealPath(appAuthParent, children);
              cacheAuthData(zkClient.readData(realPath));
              subscribeAppAuthDataChanges(realPath);
          });
      }
      subscribeChildChanges(ConfigGroupEnum.APP_AUTH, appAuthParent, childrenList);
  }

private void cacheAuthData(final AppAuthData appAuthData) {
      Optional.ofNullable(appAuthData).ifPresent(data -> authDataSubscribers.forEach(e -> e.onSubscribe(data)));
  }
   
  private void subscribeAppAuthDataChanges(final String realPath) {
      zkClient.subscribeDataChanges(realPath, new IZkDataListener() {
          @Override
          public void handleDataChange(final String dataPath, final Object data) {
              cacheAuthData((AppAuthData) data);
          }

          @Override
          public void handleDataDeleted(final String dataPath) {
              unCacheAuthData(dataPath);
          }
      });
  }
```

接下来是watchMetaData方法。同理，也是读取了meta节点的数据，通过MetaDataSubscriber缓存meta数据，并监听孩子节点的变化。

```
private void watchMetaData() {
  final String metaDataPath = ZkPathConstants.META_DATA;
  List<String> childrenList = zkClientGetChildren(metaDataPath);
  if (CollectionUtils.isNotEmpty(childrenList)) {
      childrenList.forEach(children -> {
          String realPath = buildRealPath(metaDataPath, children);
          cacheMetaData(zkClient.readData(realPath));
          subscribeMetaDataChanges(realPath);
      });
  }
  subscribeChildChanges(ConfigGroupEnum.APP_AUTH, metaDataPath, childrenList);
}
private void cacheMetaData(final MetaData metaData) {
      Optional.ofNullable(metaData).ifPresent(data -> metaDataSubscribers.forEach(e -> e.onSubscribe(metaData)));
  }
private void subscribeMetaDataChanges(final String realPath) {
      zkClient.subscribeDataChanges(realPath, new IZkDataListener() {
          @Override
          public void handleDataChange(final String dataPath, final Object data) {
              cacheMetaData((MetaData) data);
          }

          @SneakyThrows
          @Override
          public void handleDataDeleted(final String dataPath) {
              final String realPath = dataPath.substring(ZkPathConstants.META_DATA.length() + 1);
              MetaData metaData = new MetaData();
              metaData.setPath(URLDecoder.decode(realPath, StandardCharsets.UTF_8.name()));
              unCacheMetaData(metaData);
          }
      });
  }
```

### 控制台侧同步机制

用户在网页端做的数据变更的操作都是通过controller接口同步到后端的，而数据的变化都是通过DataChangedEvent向后传递的。正如上篇所讲，DataChangedEventDispatcher作为监听器对事件进行处理分发，由真正的listenter去完成数据同步的操作。

在websocket的机制下，是WebsocketDataChangedListener通过ws向各网关应用发送数据。而zk同步机制下，则是ZookeeperDataChangedListenerzk写入数据，网关的zkClient监听到zk变化后回调数据处理方法。

```
public class ZookeeperDataChangedListener implements DataChangedListener {

  private final ZkClient zkClient;

  public ZookeeperDataChangedListener(final ZkClient zkClient) {
      this.zkClient = zkClient;
  }

  @Override
  public void onAppAuthChanged(final List<AppAuthData> changed, final DataEventTypeEnum eventType) {
      for (AppAuthData data : changed) {
          final String appAuthPath = ZkPathConstants.buildAppAuthPath(data.getAppKey());
          // delete
          if (eventType == DataEventTypeEnum.DELETE) {
              deleteZkPath(appAuthPath);
              continue;
          }
          // create or update
          upsertZkNode(appAuthPath, data);
      }
  }

  @SneakyThrows
  @Override
  public void onMetaDataChanged(final List<MetaData> changed, final DataEventTypeEnum eventType) {
      for (MetaData data : changed) {
          final String metaDataPath = ZkPathConstants.buildMetaDataPath(URLEncoder.encode(data.getPath(), "UTF-8"));
          // delete
          if (eventType == DataEventTypeEnum.DELETE) {
              deleteZkPath(metaDataPath);
              continue;
          }
          // create or update
          upsertZkNode(metaDataPath, data);
      }
  }

  @Override
  public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
      for (PluginData data : changed) {
          final String pluginPath = ZkPathConstants.buildPluginPath(data.getName());
          // delete
          if (eventType == DataEventTypeEnum.DELETE) {
              deleteZkPathRecursive(pluginPath);
              final String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getName());
              deleteZkPathRecursive(selectorParentPath);
              final String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getName());
              deleteZkPathRecursive(ruleParentPath);
              continue;
          }
          //create or update
          upsertZkNode(pluginPath, data);
      }
  }

  @Override
  public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
      if (eventType == DataEventTypeEnum.REFRESH) {
          final String selectorParentPath = ZkPathConstants.buildSelectorParentPath(changed.get(0).getPluginName());
          deleteZkPathRecursive(selectorParentPath);
      }
      for (SelectorData data : changed) {
          final String selectorRealPath = ZkPathConstants.buildSelectorRealPath(data.getPluginName(), data.getId());
          if (eventType == DataEventTypeEnum.DELETE) {
              deleteZkPath(selectorRealPath);
              continue;
          }
          final String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getPluginName());
          createZkNode(selectorParentPath);
          //create or update
          upsertZkNode(selectorRealPath, data);
      }
  }

  @Override
  public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
      if (eventType == DataEventTypeEnum.REFRESH) {
          final String selectorParentPath = ZkPathConstants.buildRuleParentPath(changed.get(0).getPluginName());
          deleteZkPathRecursive(selectorParentPath);
      }
      for (RuleData data : changed) {
          final String ruleRealPath = ZkPathConstants.buildRulePath(data.getPluginName(), data.getSelectorId(), data.getId());
          if (eventType == DataEventTypeEnum.DELETE) {
              deleteZkPath(ruleRealPath);
              continue;
          }
          final String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getPluginName());
          createZkNode(ruleParentPath);
          //create or update
          upsertZkNode(ruleRealPath, data);
      }
  }
   
  private void createZkNode(final String path) {
      if (!zkClient.exists(path)) {
          zkClient.createPersistent(path, true);
      }
  }
   
  /**
    * create or update zookeeper node.
    * @param path node path
    * @param data node data 
    */
  private void upsertZkNode(final String path, final Object data) {
      if (!zkClient.exists(path)) {
          zkClient.createPersistent(path, true);
      }
      zkClient.writeData(path, data);
  }
   
  private void deleteZkPath(final String path) {
      if (zkClient.exists(path)) {
          zkClient.delete(path);
      }
  }
   
  private void deleteZkPathRecursive(final String path) { 
      if (zkClient.exists(path)) {
          zkClient.deleteRecursive(path);
      }
  }
}
```





## 总结

通过以上对数据同步的流程跟踪，我们可以和上一篇ws数据同步进行比较。区别和相同点在于

- 控制台侧是通过修改zk节点传递的数据信息，而ws是直接和网关进行同步。
- 网关侧则是通过不同的渠道监听数据变化，都是使用回调方法进行同步。