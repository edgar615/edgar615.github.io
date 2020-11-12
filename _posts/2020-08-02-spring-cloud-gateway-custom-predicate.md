---
layout: post
title: Spring Cloud Gateway-  �Զ������
date: 2020-08-02
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-custom-predicate.html
---

Spring Cloud Gateway��Ȼ������һЩ���ԣ���������Щ����¿��ܲ������á�Spring Cloud Gatewayģ���ṩ���Զ�����Եķ�����

# 1. �ٷ�����

Spring Cloud Gatewayʹ�ù���ģʽʵ�ֶ��ԣ����ǿ��Կ�һ���ٷ�ʵ�֣���ʵ����`RoutePredicateFactory`�ӿڣ�

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

`RoutePredicateFactory<C>`�ӿ���Ҫ����������

```java
default List<String> shortcutFieldOrder() {
	return Collections.emptyList();
}

default ShortcutType shortcutType() {
    return ShortcutType.DEFAULT;
}

Predicate<ServerWebExchange> apply(C config);
```

`shortcutType()`��`shortcutFieldOrder()`������Ҫ���ʹ�ã����ڷ���Config��Ӧ���ֶ�,Ĭ������·���Config�������ֶ�

`apply(C config)`����ִ�о���Ķ����߼�

`AfterRoutePredicateFactory`���췽����������װ���õ��࣬Spring Cloud Gateway���Զ��������е�value����apply������Ρ�

# 2. ʵ���Զ������

����ʵ������ĳ�������һ������ά���APIֻ���ض����û��������ض���IP�����ܷ���

## 2.1. �Զ�����Թ���

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



## 2.2. ע����Թ���

```java
@Bean
DevopsRoutePredicateFactory devopsRoutePredicateFactory() {
    return new DevopsRoutePredicateFactory();
}
```

## 2.3. ʹ��Fluent Api����Route

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

## 2.4. ����

```sh
$ curl -s --cookie "user=Edgar" http://localhost:8080/hello
{"content-length":"0","cookie":"user=Edgar","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","hello":"World","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:53089\"","accept":"*/*","user-agent":"curl/7.59.0"}

```

## 2.5. ʹ��yaml����Route

```yaml
- id: devops_route
  uri: http://localhost:9001
  predicates:
  - name: Devops #����RoutePredicateFactoryʱ��ǰ�沿��
	args:
	  name: user
	  devopsUser:
	  - Edgar
	  - Git
```

`shortcutType()`��`shortcutFieldOrder()`����һ�㲻��Ҫ��д���������Ҫ���Կ�`PathRoutePredicateFactory`��`RemoteAddrRoutePredicateFactory`��ʵ��



