---
layout: post
title: Spring Cloud Gateway -  自定义断言
date: 2020-08-02
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-custom-predicate.html
---

Spring Cloud Gateway虽然内置了一些断言，但是在有些情况下可能并不够用。Spring Cloud Gateway模块提供了自定义断言的方法。

# 1. 官方断言

Spring Cloud Gateway使用工厂模式实现断言，我们可以看一个官方实现，它实现了`RoutePredicateFactory`接口，

```java
public class AfterRoutePredicateFactory
		extends AbstractRoutePredicateFactory<AfterRoutePredicateFactory.Config> {

	/**
	 * DateTime key.
	 */
	public static final String DATETIME_KEY = "datetime";

	public AfterRoutePredicateFactory() {
		super(Config.class);
	}

	@Override
	public List<String> shortcutFieldOrder() {
		return Collections.singletonList(DATETIME_KEY);
	}

	@Override
	public Predicate<ServerWebExchange> apply(Config config) {
		ZonedDateTime datetime = config.getDatetime();
		return exchange -> {
			final ZonedDateTime now = ZonedDateTime.now();
			return now.isAfter(datetime);
		};
	}

	public static class Config {

		@NotNull
		private ZonedDateTime datetime;

		public ZonedDateTime getDatetime() {
			return datetime;
		}

		public void setDatetime(ZonedDateTime datetime) {
			this.datetime = datetime;
		}

	}

}
```

`RoutePredicateFactory<C>`接口主要有三个方法

```java
default List<String> shortcutFieldOrder() {
	return Collections.emptyList();
}

default ShortcutType shortcutType() {
    return ShortcutType.DEFAULT;
}

Predicate<ServerWebExchange> apply(C config);
```

`shortcutType()`和`shortcutFieldOrder()`方法需要配合使用，用于返回Config对应的字段,默认情况下返回Config的所有字段

`apply(C config)`方法执行具体的断言逻辑

`AfterRoutePredicateFactory`构造方法传入用来装配置的类，Spring Cloud Gateway会自动把配置中的value传入apply中做入参。

# 2. 实现自定义断言

我们实现下面的场景：有一部分运维类的API只有特定的用户（或者特定的IP）才能访问

## 2.1. 自定义断言工厂

```java
public class DevopsRoutePredicateFactory extends AbstractRoutePredicateFactory<DevopsRoutePredicateFactory.Config> {
    public DevopsRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return serverWebExchange -> {
            List<HttpCookie> cookies = serverWebExchange.getRequest()
                    .getCookies()
                    .get(config.getName());
            if (cookies == null || cookies.isEmpty()) {
                return false;
            } else {
                String userId = cookies.get(0).getValue();
                return config.getDevopsUser().contains(userId);
            }
        };
    }

    @Validated
    public static class Config {

        @NotEmpty
        private String name;

        @NotEmpty
        private List<String> devopsUser;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public List<String> getDevopsUser() {
            return devopsUser;
        }

        public void setDevopsUser(List<String> devopsUser) {
            this.devopsUser = devopsUser;
        }
    }
}
```



## 2.2. 注册断言工厂

```java
@Bean
DevopsRoutePredicateFactory devopsRoutePredicateFactory() {
    return new DevopsRoutePredicateFactory();
}
```

## 2.3. 使用Fluent Api定义Route

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder, DevopsRoutePredicateFactory devopsRoutePredicateFactory) {
    return builder.routes()
        .route(p -> p.path("/hello")
               .filters(f -> f.addRequestHeader("Hello", "World"))
               .uri("http://localhost:9001")
               .asyncPredicate(devopsRoutePredicateFactory.applyAsync(config -> {
                   config.setName("user");
                   config.setDevopsUser(Collections.singletonList("Edgar"));
               }))
              )
        .build();
}
```

## 2.4. 测试

```sh
$ curl -s --cookie "user=Edgar" http://localhost:8080/hello
{"content-length":"0","cookie":"user=Edgar","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","hello":"World","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:53089\"","accept":"*/*","user-agent":"curl/7.59.0"}

```

## 2.5. 使用yaml定义Route

```yaml
- id: devops_route
  uri: http://localhost:9001
  predicates:
  - name: Devops #命名RoutePredicateFactory时的前面部分
	args:
	  name: user
	  devopsUser:
	  - Edgar
	  - Git
```

`shortcutType()`和`shortcutFieldOrder()`方法一般不需要重写，如果有需要可以看`PathRoutePredicateFactory`和`RemoteAddrRoutePredicateFactory`的实现



