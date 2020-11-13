---
layout: post
title: Spring Cloud Gateway - 自定义Filter
date: 2020-08-04
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-custom-filter.html
---

Spring Cloud Gateway 提供了两种Filter：全局Filter(GlobalFilter)和局部Filter(GatewayFilter)

- 全局Filter，对所有的路由都有效，主要实现了GlobalFilter 和 Ordered接口，并将过滤器注册到spring  容器。
- 局部过滤器，需要在配置文件中配置，如果配置，则该过滤器才会生效。主要实现GatewayFilter,  Ordered接口，并通过AbstractGatewayFilterFactory的子类注册到spring容器中，当然也可以直接继承AbstractGatewayFilterFactory，在里面写过滤器逻辑，还可以从配置文件中读取外部数据。

# 1. 自定义全局Filter

```
@Component
public class ElapsedGatewayFilter implements GlobalFilter, Ordered {
    public static final int ELAPSED_ORDER = -100000;

    private static final Logger LOGGER = LoggerFactory.getLogger(ElapsedGatewayFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            Long endTime = System.currentTimeMillis();
            LOGGER.info("{}, elapsed {}ms", exchange.getRequest().getURI().getRawPath(), endTime - startTime);
        }));
    }

    @Override
    public int getOrder() {
        return ELAPSED_ORDER;
    }
}
```

# 2. 自定义局部Filter

GatewayFilter使用工厂模式生成

```
@Component
@Order(-10000)
public class IpBlacklistGatewayFilterFactory
        extends AbstractGatewayFilterFactory<IpBlacklistGatewayFilterFactory.Config> {

    public IpBlacklistGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {

        return (exchange, chain) -> {
            String ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            if (config.getBlacklist().contains(ip)) {
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                exchange.getResponse().setComplete();
            }
            return chain.filter(exchange);
        };
    }

    public static class Config {
        private Set<String> blacklist = new HashSet<>();

        public Set<String> getBlacklist() {
            return blacklist;
        }

        public void setBlacklist(Set<String> blacklist) {
            this.blacklist = blacklist;
        }
    }
}
```

route配置

```
- id: route
  uri: http://httpbin.org:80/get
  predicates:
	- After=2020-04-20T00:00:00+08:00[Asia/Shanghai]
  filters:
	- name: IpBlacklist
	  args:
		blacklist:
		- "192.168.2.100"
		- "192.168.1.100"
```

我们在使用官方提供的Filter的时候，看到有一种简易写法

```
filters:
  - StripPrefix=2
```

这种写法需要实现两个方法

```
@Override
public List<String> shortcutFieldOrder() {
	return Lists.list("blacklist");
}

@Override
public ShortcutType shortcutType() {
	return ShortcutType.DEFAULT;
}
```

`shortcutFieldOrder`用了定义参数名称，可以接受多个参数,`shortcutType`用了定义参数类型，默认实现为DEFAULT

```
  - id: route
	uri: http://httpbin.org:80/get
	predicates:
	  - After=2020-04-20T00:00:00+08:00[Asia/Shanghai]
	filters:
	  - IpBlacklist=192.168.2.100, 192.168.2.101
```

`ShortcutType.DEFAULT`只会解析出一个IP`192.168.2.100`,而`ShortcutType.GATHER_LIST`会解析出两个IP：`192.168.2.100` `192.168.2.101`