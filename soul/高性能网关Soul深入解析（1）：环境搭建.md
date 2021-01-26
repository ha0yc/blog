## Soul是什么

Soul（https://github.com/dromara/soul）是一个基于Spring WebFlux的异步、反应式、高性能的API网关。

Soul是一个跨语言的网关，通过Http协议，支持了php、java等多种语言，还支持了dubbo、spring cloud等协议。基于插件化的思想，Soul内置了丰富多样的插件，包括鉴权、限流、熔断、防火墙、负载均衡、多版本和灰度等功能。同时，Soul具有动态配置流量的特性，性能极高。

## 为什么要学习源码

学Soul或者学其他框架，最基本的目的是发生问题的时候能快速定位问题解决问题。对于个人，可以提升成就感和得到学习的正反馈，促进自身继续学习；在职场上，可以提升你的影响力，成为大家眼中解决问题的人和不可或缺的人。更深一层，是增加技术素养，增强技术深度，最终会反馈于工作技能的提升。再深一层，就是将术变成道，一法通则百法通，成为你所在技术领域真正的专家。

## 环境准备

话不多说，先一键三联，star wartch and fork。环境准备如下

- JDK 1.8（不要使用11，lombok在11下无法进行本地编译生成代码）
- Maven 3.3
- Intellij IDEA
- Soul 2.2.1

## Soul初探

Soul源码clone到本地后，切换到2.2.1分支。可以看到模块的基本情况，通过包名和其中类名我们可以猜下每个模块是做什么的，后面的源码解读模块可以求证。

```xml
<modules>
        <module>soul-admin</module><!--管理模块-->
        <module>soul-common</module><!--框架基础工具类包，供其他模块使用-->
        <module>soul-web</module><!--网关核心包，用于网关流程的flux封装-->
        <module>soul-spring-boot-starter</module><!--显而易见，starter和auto configuration-->
        <module>soul-bootstrap</module><!--soul启动-->
        <module>soul-client</module><!--框架基础工具类-->
        <module>soul-plugin</module><!--框架内置插件-->
        <module>soul-sync-data-center</module><!--框架数据同步-->
        <module>soul-metrics</module><!--框架监控-->
        <module>soul-spi</module><!--SPI基础-->
    </modules>
```

## 运行控制台和启动器

### 控制台（soul-admin）

在application-local.yml中配置好数据库信息就可以启动了，soul的控制台启动端口为9095，bootstrap的控制台启动端口为9195。我们既没使用zk的数据同步也没使用narcos，就都不用配置了

```yml
server:
  port: 9095
  address: 0.0.0.0

spring:
  thymeleaf:
    cache: true
    encoding: utf-8
    enabled: true
    prefix: classpath:/static/
    suffix: .html
  datasource:
    url: jdbc:mysql://localhost:3306/soul?useUnicode=true&characterEncoding=utf-8
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver

mybatis:
  config-location: classpath:/mybatis/mybatis-config.xml
  mapper-locations: classpath:/mappers/*.xml


soul:
  sync:
    websocket:
        enabled : true

logging:
  level:
    root: info
    org.springframework.boot: info
    org.apache.ibatis: info
    org.dromara.soul.bonuspoint: info
    org.dromara.soul.lottery: info
    org.dromara.soul: info
```

启动时，Soul会自动建好表的，还会向数据库插入一条控制台管理员信息。

启动成功后界面如下

![1-1](images\soul-1\1-1.png)


可以看到支持的插件都被list出来，同时还有系统管理功能。插件界面如下：

![1-2](images\soul-1\1-2.png)

### 启动器（soul-bootstrap）

从本模块可以看出，启动器配置了NettyReactiveWebServerFactory，和Spring Cloud Gateway一样，都是基于netty的reactor异步的网关。Soul在这里对NettyReactiveWebServerFactory进行了定制，自定义了其中一些网络参数。

```java
@Bean
    public NettyReactiveWebServerFactory nettyReactiveWebServerFactory() {
        NettyReactiveWebServerFactory webServerFactory = new NettyReactiveWebServerFactory();
        webServerFactory.addServerCustomizers(new EventLoopNettyCustomizer());
        return webServerFactory;
    }

    private static class EventLoopNettyCustomizer implements NettyServerCustomizer {

        @Override
        public HttpServer apply(final HttpServer httpServer) {
            return httpServer
                    .tcpConfiguration(tcpServer -> tcpServer
                            .runOn(LoopResources.create("soul-netty", 1, DEFAULT_IO_WORKER_COUNT, true), false)
                            .selectorOption(ChannelOption.SO_REUSEADDR, true)
                            .selectorOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                            .option(ChannelOption.TCP_NODELAY, true)
                            .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT));
        }
    }
```

bootstrap同时还提供了健康检查功能，并通过ws和admin同步数据。启动日志如下

```java
2021-01-14 22:27:15.047  INFO 16836 --- [           main] o.d.s.b.SoulBootstrapApplication         : Starting SoulBootstrapApplication on DESKTOP-HERB92P with PID 16836 (D:\code\soul\soul-bootstrap\target\classes started by haoyc in D:\code\soul)
2021-01-14 22:27:15.049  INFO 16836 --- [           main] o.d.s.b.SoulBootstrapApplication         : The following profiles are active: local
2021-01-14 22:27:18.756  INFO 16836 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2021-01-14 22:27:18.765  INFO 16836 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data Redis repositories in DEFAULT mode.
2021-01-14 22:27:18.884  INFO 16836 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 48ms. Found 0 Redis repository interfaces.
2021-01-14 22:27:20.822  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[global] [org.dromara.soul.plugin.global.GlobalPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[waf] [org.dromara.soul.plugin.waf.WafPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[rate_limiter] [org.dromara.soul.plugin.ratelimiter.RateLimiterPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[hystrix] [org.dromara.soul.plugin.hystrix.HystrixPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[divide] [org.dromara.soul.plugin.divide.DividePlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[webClient] [org.dromara.soul.plugin.httpclient.WebClientPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[divide] [org.dromara.soul.plugin.divide.websocket.WebSocketPlugin]
2021-01-14 22:27:20.823  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[monitor] [org.dromara.soul.plugin.monitor.MonitorPlugin]
2021-01-14 22:27:20.824  INFO 16836 --- [           main] o.d.s.w.configuration.SoulConfiguration  : loader plugin:[response] [org.dromara.soul.plugin.httpclient.response.WebClientResponsePlugin]
2021-01-14 22:27:21.293  INFO 16836 --- [           main] b.s.s.d.w.WebsocketSyncDataConfiguration : you use websocket sync soul data.......
2021-01-14 22:27:23.623 ERROR 16836 --- [           main] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket connection is error.....
2021-01-14 22:27:23.869  INFO 16836 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2021-01-14 22:27:26.140  INFO 16836 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 9195
2021-01-14 22:27:26.147  INFO 16836 --- [           main] o.d.s.b.SoulBootstrapApplication         : Started SoulBootstrapApplication in 12.906 seconds (JVM running for 17.61)
2021-01-14 22:27:35.631 ERROR 16836 --- [ocket-connect-1] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket reconnection is error.....
2021-01-14 22:28:05.843 ERROR 16836 --- [ocket-connect-1] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket reconnection is error.....
2021-01-14 22:28:33.943  INFO 16836 --- [ocket-connect-1] o.d.s.p.s.d.w.WebsocketSyncDataService   : websocket reconnect is successful.....
```

## 展望

看了猫大人的Soul之后，顿感需要学习的地方太多了。正好可以借此机会，多研究一些还没来得及看的内容。

- starter机制和autoConfiguration，深入学习Spring的配置加载原理
- SPI机制和Dubbo中的SPI
- WebFlux响应式框架的原理
- 数据同步机制，以及，如果有机会可以给Soul增加对apollo和disconf的支持
- 设计模式在Soul中的实践
- 吾生也有涯，而知也无涯。以有涯随无涯，殆已！已而为知者，殆而已矣！