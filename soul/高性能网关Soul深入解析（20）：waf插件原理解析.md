## Soul：waf插件原理分析

Soul网关不仅仅要对后端服务提供代理转发和灰度等基本功能，还要在网关层完成防火墙功能。

### waf插件原理

首先在waf插件中配置selector和rule数据，url均为/http/order/findById，rule中请求策略为拒绝，状态码为402。

请求后返回请求结果如下。

```json
{"code":402,"message":" You are forbidden to visit","data":null}
```

在AbstractSoulPlugin中获取到selector和rule信息后，调用Sign插件的执行方法。

```java
@Slf4j
public class WafPlugin extends AbstractSoulPlugin {

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        WafConfig wafConfig = Singleton.INST.get(WafConfig.class);
        if (Objects.isNull(selector) && Objects.isNull(rule)) {
            if (WafModelEnum.BLACK.getName().equals(wafConfig.getModel())) {
                return chain.execute(exchange);
            }
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            Object error = SoulResultWrap.error(403, Constants.REJECT_MSG, null);
            return WebFluxResultUtils.result(exchange, error);
        }
        String handle = rule.getHandle();
        WafHandle wafHandle = GsonUtils.getInstance().fromJson(handle, WafHandle.class);
        if (Objects.isNull(wafHandle) || StringUtils.isBlank(wafHandle.getPermission())) {
            log.error("waf handler can not configuration：{}", handle);
            return chain.execute(exchange);
        }
        if (WafEnum.REJECT.getName().equals(wafHandle.getPermission())) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            Object error = SoulResultWrap.error(Integer.parseInt(wafHandle.getStatusCode()), Constants.REJECT_MSG, null);
            return WebFluxResultUtils.result(exchange, error);
        }
        return chain.execute(exchange);
    }
}
```

可以看到，当请求进入到了waf插件后已经匹配了waf插件的selector和rule规则，进入插件后生成了WafHandle对象，然后查看wafHandle对象的权限和错误码，如果和reject匹配就返回对应错误码的response，反之则继续调用后续流程。