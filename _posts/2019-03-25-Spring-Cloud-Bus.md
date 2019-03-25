---
layout: post
title: Spring Cloud Bus
date: 2019-03-25
categories:
    - Spring Boot
    - Spring Cloud
comments: true
permalink: Spring-Cloud-Bus.html
---

在微服务中，我们将使用轻量级消息代理，通过一个共用的消息主题，让系统中所有微服务都连上来，主题中的消息会被所有监听者消费，所以称为消息总线。

消息代理(Message Broker)是一种消息验证、传输、路由的架构模式。它在应用程序之间起到通信调度并最小化应用之间的依赖的作用，使得应用程序可以高效地解耦通信过程。消息代理是一个中间件产品，它的核心是一个消息的路由程序，用来实现接收和分发消息， 并根据设定好的消息处理流来转发给正确的应用。 它包括独立的通信和消息传递协
议，能够实现组织内部和组织间的网络通信。设计代理的目的就是为了能够从应用程序中传入消息，并执行一些特别的操作，下面这些是在企业应用中，我们经常需要使用消息代理的场景：

- 将消息路由到一个或多个目的地。
- 消息转化为其他的表现方式。
- 执行消息的聚集、消息的分解，并将结果发送到它们的目的地，然后重新组合响应返回给消息用户。
- 调用Web服务来检索数据。
- 响应事件或错误。
- 使用发布－订阅模式来提供内容或基千主题的消息路由。


# get started
增加依赖
```
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
```

增加application.yml配置

```
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: 192.168.1.212:9092
          minPartitionCount: 1
          autoCreateTopics: true
          autoAddPartitions: true
          
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env
```

增加启动类

```
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
      SpringApplication.run(Application.class, args);
    }
}
```

启动应用后，通过观察日志我们发现应用暴露了两个端点`bus-refresh`和`bus-env`

```
Mapped "{[/actuator/bus-env],methods=[POST],consumes=[application/vnd.spring-boot.actuator.v2+json || application/json]}" 
Mapped "{[/actuator/bus-env/{destination}],methods=[POST],consumes=[application/vnd.spring-boot.actuator.v2+json || application/json]}" 
Mapped "{[/actuator/bus-refresh/{destination}],methods=[POST]}" 
Mapped "{[/actuator/bus-refresh],methods=[POST]}
```

使用`springCloudBus`主题作为总线地址

```
Using kafka topic for outbound: springCloudBus
```

同时可以看到应用随机生成了一个groupId

```
[Consumer clientId=consumer-2, groupId=anonymous.dcaa77f8-cdeb-402a-8a60-e004d89594ac] Discovered group coordinator 192.168.1.212:9092 (id: 2147483647 rack: null)
[Consumer clientId=consumer-2, groupId=anonymous.dcaa77f8-cdeb-402a-8a60-e004d89594ac] Revoking previously assigned partitions []
[container-0-C-1] o.s.c.s.b.k.KafkaMessageChannelBinder$1  : partitions revoked: []
```



触发一次`/bus-refresh`

```
$ curl -s -X POST -H "Content-Type: application/json" http://localhost:8080/actuator/bus-refresh
```

可以看到另一个应用收到了对应的refresh

```
Received remote refresh request. Keys refreshed []
```
`springCloudBus`中出现了三个消息，一个`RefreshRemoteApplicationEvent`，两个`AckRemoteApplicationEvent`（我总共启动了两个应用）
```
{"type":"RefreshRemoteApplicationEvent","timestamp":1553496711901,"originService":"application:0:48fcd10b6e8981d213b01250be5e292e","destinationService":"**","id":"5571a1db-ef1d-4791-ad42-ad36d60b43df"}
{"type":"AckRemoteApplicationEvent","timestamp":1553496711904,"originService":"application:0:48fcd10b6e8981d213b01250be5e292e","destinationService":"**","id":"cbdda733-8dbd-403a-a8e3-f76016e0d19b","ackId":"5571a1db-ef1d-4791-ad42-ad36d60b43df","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"}
{"type":"AckRemoteApplicationEvent","timestamp":1553496712363,"originService":"application:9001:b06bd84ba7be1d2e0cf7d101b55b6f2a","destinationService":"**","id":"028ac8a9-a23c-4386-ae86-581facfd3326","ackId":"5571a1db-ef1d-4791-ad42-ad36d60b43df","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"}
```
触发一次的`/bus-env

```
$ curl -s -X POST -d '{"name": "key1", "value": "value1"}' -H "Content-Type: application/json" http://localhost:8080/actuator/bus-env
```

```
Received remote environment change request. Keys/values to update {key1=value1}
```
`springCloudBus`中出现了三个消息，一个`EnvironmentChangeRemoteApplicationEvent`，两个`AckRemoteApplicationEvent`
```
{"type":"EnvironmentChangeRemoteApplicationEvent","timestamp":1553496803529,"originService":"application:0:48fcd10b6e8981d213b01250be5e292e","destinationService":"**","id":"f7634064-10fa-47ff-8e2e-acc965c8b74c","values":{"key1":"value1"}}
{"type":"AckRemoteApplicationEvent","timestamp":1553496803538,"originService":"application:0:48fcd10b6e8981d213b01250be5e292e","destinationService":"**","id":"89285fe4-c4de-4226-9089-1b6c3a46bc64","ackId":"f7634064-10fa-47ff-8e2e-acc965c8b74c","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.EnvironmentChangeRemoteApplicationEvent"}
{"type":"AckRemoteApplicationEvent","timestamp":1553496803590,"originService":"application:9001:b06bd84ba7be1d2e0cf7d101b55b6f2a","destinationService":"**","id":"99e10919-410a-464a-951e-c1f64248f440","ackId":"f7634064-10fa-47ff-8e2e-acc965c8b74c","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.EnvironmentChangeRemoteApplicationEvent"}
```
# 实例地址

应用的每个实例都有个实例ID，它可以通过`spring.cloud.bus.id`指定，而且它的值应该是使用`:`分隔的字符串，从左到右依次确定实例的ID，默认的实例ID为`{app}:{index}:{id}`

- `app`  通过 `vcap.application.name`设置，如果不存在使用 `spring.application.name`
- `index` 通过 `vcap.application.instance_index`设置,如果不存在，按照下面的顺序设置, `spring.application.index`, `local.server.port`, `server.port`, or `0`
- `id` 通过 `vcap.application.instance_id`设置, 如果不存在，使用随机数


我们按照上述规则配置后重新观察`springCloudBus`主题，发现服务的ID已经变化了

```
{"type":"AckRemoteApplicationEvent","timestamp":1553496976970,"originService":"cloud-bus-kafka2:9001:15b4c214558697788aa4a47d01f1e2c7","destinationService":"**","id":"3c4d0e87-f3de-40b8-9169-3bc33d8eacd3","ackId":"4f7be4b5-009b-482c-9888-4b63dce6bb88","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.EnvironmentChangeRemoteApplicationEvent"}
```
我们可以在端点指定具体的应用，例如`/bus-refresh/cloud-bus-kafka2:9001`，当应用收到消息后首先判断是否是它的消息，如果是处理这个消息，其他消息将被忽略

测试
```
$ curl -s -X POST -d '{"name": "key1", "value": "value1"}' -H "Content-Type: application/json" http://localhost:8080/actuator/bus-env/cloud-bus-kafka2:9001
```
我们可以看到，这时候kafka中只有两个消息
```
{"type":"EnvironmentChangeRemoteApplicationEvent","timestamp":1553497132777,"originService":"application:0:48fcd10b6e8981d213b01250be5e292e","destinationService":"cloud-bus-kafka2:9001:**","id":"23461004-65c5-4b5b-b1a4-91e57717d4d4","values":{"key1":"value1"}}
{"type":"AckRemoteApplicationEvent","timestamp":1553497132805,"originService":"cloud-bus-kafka2:9001:15b4c214558697788aa4a47d01f1e2c7","destinationService":"**","id":"796a174b-3402-4c77-bd34-2fca57ea322c","ackId":"23461004-65c5-4b5b-b1a4-91e57717d4d4","ackDestinationService":"cloud-bus-kafka2:9001:**","event":"org.springframework.cloud.bus.event.EnvironmentChangeRemoteApplicationEvent"}
```
`EnvironmentChangeRemoteApplicationEvent`消息中指明了`destinationService`
如果想匹配某个服务的所有实例，可以使用`cloud-bus-kafka2:**`

# 跟踪总线的事件
前面看到的`EnvironmentChangeRemoteApplicationEvent`、`AckRemoteApplicationEvent`这些都是`RemoteApplicationEvent`的子类，我们可以通过`spring.cloud.bus.trace.enabled=true`来跟踪它

如果我们想自己处理ack事件，可以通过`@EventListener`注解监听`AckRemoteApplicationEvent`和`SentApplicationEvent`来实现，也可以通过`tracerepository`来挖掘
`tracerepository`这我找了好多资料也不知道该怎么做

# 自定义事件
1 继承`RemoteApplicationEvent`
<pre class="line-numbers "><code class="language-java">
public class MyCustomRemoteEvent extends RemoteApplicationEvent {
    private String message;

    public MyCustomRemoteEvent() {
    }

    public MyCustomRemoteEvent(Object source, String originService, String message) {
        // source is the object that is publishing the event
        // originService is the unique context ID of the publisher
        super(source, originService);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public MyCustomRemoteEvent setMessage(String message) {
        this.message = message;
        return this;
    }
    
}
</code></pre>
注意：构造函数的第一个参数是发布事件的对象，第二个参数是发布事件的实例ID，第三个对象才是我们的消息
我们可以将自定义的事件放在`org.springframework.cloud.bus.event`的子包下，但是这种方式不灵活
2 定义事件的包扫描，支持`value`, `basePackages` 和`basePackageClasses`三种方式，默认BusConfiguration的路径
<pre class="line-numbers "><code class="language-java">
@Configuration
//@RemoteApplicationEventScan({"com.acme", "foo.bar"})
//@RemoteApplicationEventScan(basePackages = {"com.acme", "foo.bar", "fizz.buzz"})
@RemoteApplicationEventScan(basePackageClasses = BusConfiguration.class)
public class BusConfiguration {
}
</code></pre>

3 增加端点
<pre class="line-numbers "><code class="language-java">
@Endpoint(id = "event")
public class EventEndpoint extends AbstractBusEndpoint {

	public EventEndpoint(ApplicationEventPublisher context, String id) {
		super(context, id);
	}

	@WriteOperation
	public String event() {
    String data = UUID.randomUUID().toString();
    publish(new MyCustomRemoteEvent(data, getInstanceId(), data));
    return data;
	}

}
</code></pre>
4 配置端点
<pre class="line-numbers "><code class="language-java">
  @Bean
  public EventEndpoint eventEndpoint(ApplicationContext context, BusProperties bus) {
    return new EventEndpoint(context, bus.getId());
  }
</code></pre>
测试
```
$ curl -s -X POST -d '{"name": "key1", "value": "value1"}' -H "Content-Type: application/json" http://localhost:8080/actuator/event
```
我们看到kafka中多了三条消息
```
{"type":"MyCustomRemoteEvent","timestamp":1553502643137,"originService":"application:0:8c0933e8abe0729c7a843cf23414a301","destinationService":"**","id":"f7d501e7-48d6-4fae-9bbe-d3c3c351d417","message":"bf8c4542-b869-4208-bab0-936fb49c59d6"}
{"type":"AckRemoteApplicationEvent","timestamp":1553502643149,"originService":"application:0:8c0933e8abe0729c7a843cf23414a301","destinationService":"**","id":"0dba62b0-6a97-495d-91a8-433dafb3ecd0","ackId":"f7d501e7-48d6-4fae-9bbe-d3c3c351d417","ackDestinationService":"**","event":"com.github.edgar615.example.bus.MyCustomRemoteEvent"}
{"type":"AckRemoteApplicationEvent","timestamp":1553502643152,"originService":"cloud-bus-kafka2:9001:984d42e380589881af8e53f224241ddf","destinationService":"**","id":"25078bb5-0f34-41ff-8ebd-c2bed0fc0e7b","ackId":"f7d501e7-48d6-4fae-9bbe-d3c3c351d417","ackDestinationService":"**","event":"com.github.edgar615.example.bus.MyCustomRemoteEvent"}
```
我们也可以不通过endpoint，而是直接通过controller发布事件
<pre class="line-numbers "><code class="language-java">
  @PostMapping("/event")
  public String event() {
    String data = UUID.randomUUID().toString();
    // context.getId()不是cloud-bus的实例ID
    RemoteApplicationEvent event = new MyCustomRemoteEvent(this, busProperties.getId(), data);
    publisher.publishEvent(event);
    return data;
  }
</code></pre>