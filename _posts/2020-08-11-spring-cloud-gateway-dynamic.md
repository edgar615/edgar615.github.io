---
layout: post
title: Spring Cloud Gateway - 动态路由
date: 2020-08-11
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-discovery.html
---

由于 Spring Cloud Gateway 本身也是一个服务，一旦启动以后路由配置就无法修改了，如果需要修改都需要重新启动服务。

如果回到 Spring Cloud Gateway 最初的定义，我们会发现每个用户的请求都是通过 Route 访问对应的微服务，在 Route 中包括 Predicates 和 Filters 的定义。只要实现 Route 以及其包含的 Predicates 和 Filters 的定义，然后再提供一个 API 接口去更新这个定义就可以动态地修改路由信息了。

RouteDefinition就是Route实体的定义类

```
public class FilterDefinition {
    //Filter Name
    private String name;
    //对应的路由规则
    private Map<String, String> args = new LinkedHashMap<>();
}
public class PredicateDefinition {
    //Predicate Name
    private String name;
    //对应的断言规则
    private Map<String, String> args = new LinkedHashMap<>();
}
public class RouteDefinition {
    //断言集合
private List<PredicateDefinition> predicates = new ArrayList<>();
//路由集合
private List< FilterDefinition > filters= new ArrayList<>();
//uri
private String uri;
//执行次序
private int order = 0;
}
```

**实现路由规则的操作**

```
@Service
public class DynamicRouteServiceImpl implements DynamicRouteService, ApplicationEventPublisherAware {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    private ApplicationEventPublisher publisher;
    
    @Override
    public void addRoute(RouteDefinition routeDefinition) {
        routeDefinitionWriter.save(Mono.just(routeDefinition)).subscribe();
        // EVENT中的对象没有实际用处
        publisher.publishEvent(new RefreshRoutesEvent(new Object()));
    }

    @Override
    public void deleteRoute(String id) {
        routeDefinitionWriter.delete(Mono.just(id)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }
}
```

**对外部提供 API 接口能够让用户或者程序动态修改路由规则**

```
@RestController
@RequestMapping("/route")
public class RouteController {

    @Autowired
    private DynamicRouteService dynamicRouteService;

    @PostMapping
    public String add(@RequestBody RouteDefinition routeDefinition) {
        dynamicRouteService.addRoute(routeDefinition);
        return "success";
    }

    @DeleteMapping("/{id}")
    public String delete(@PathVariable("id") String id) {
        dynamicRouteService.deleteRoute(id);
        return "success";
    }
}
```

**开启actuator，观察路由变化**

```
management:
  endpoint:
    gateway:
      enabled: true # default value
  endpoints:
    web:
      exposure:
        include: gateway
```

可以通过http://localhost:8080/actuator/gateway/routes 访问当前的路由信息，路由信息为空

```
$ curl -si  http://localhost:8080/actuator/gateway/routes
HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json

[]

```

添加路由信息

```
POST http://localhost:8080/route
Content-Type: application/json

{
"id": "9999",
  "uri": "http://localhost:9001",
  "order":0,
  "predicates":[{
    "args":{
      "datetime":"2020-04-20T00:00:00+08:00[Asia/Shanghai]"
    },
    "name":"After"
  }]
}
```

再次访问http://localhost:8080/actuator/gateway/routes

```
$ curl -si  http://localhost:8080/actuator/gateway/routes
HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json

[{"predicate":"After: 2020-04-20T00:00+08:00[Asia/Shanghai]","route_id":"9999","filters":[],"uri":"http://localhost:9001","order":0}]

```

删除路由

```
DELETE http://localhost:8080/route/9999
Content-Type: application/json
```

