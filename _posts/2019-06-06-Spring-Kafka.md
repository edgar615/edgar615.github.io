---
layout: post
title: Spring - kafka
date: 2019-06-06
categories:
    - Spring
comments: true
permalink: spring-kafka.html
---

公司项目用的MQ是kafka，自己封装了一个消息组件。简单了解下spring与kafka的集成中自己会用到的地方，并未在项目中使用。详细的用法参考官方文档

# 1. Get Started

增加依赖

```
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
```

增加配置
```
spring:
    kafka:
        bootstrap-servers: 192.168.1.212:9092
        consumer:
            group-id: test
            enable-auto-commit: true
```

增加发送消息的类

```
@Component
public class MessageSender {

  @Autowired
  private KafkaTemplate template;

  public void send() {
    this.template.send("myTopic", "message1");
    this.template.send("myTopic", "message2");
    this.template.send("myTopic", "message3");
  }

}
```

增加消费消息的类

```
@Component
public class MessageReceiver {

	@KafkaListener(topics = "myTopic")
	public void processMessage(String content) {
		System.out.println(content);
	}
}
```

测试

```
@SpringBootApplication(scanBasePackages = {"com.github.edgar615.**"})
@Configuration
public class Application implements CommandLineRunner {

  @Autowired
  private MessageSender messageSender;

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

  @Override
  public void run(String... args) throws Exception {
    messageSender.send();
  }
}
```

# 2. 创建Topic

我们可以通过TopicBuilder创建Topic

```
@Bean
public NewTopic topic1() {
return TopicBuilder.name("thing1")
		.partitions(5)
		.compact()
		.build();
}
```

分区数可以增加，但不会减少

```
# ./bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic thing1 --describe
Topic: thing1   PartitionCount: 5       ReplicationFactor: 1    Configs: cleanup.policy=compact,segment.bytes=1073741824
        Topic: thing1   Partition: 0    Leader: 0       Replicas: 0     Isr: 0
        Topic: thing1   Partition: 1    Leader: 0       Replicas: 0     Isr: 0
        Topic: thing1   Partition: 2    Leader: 0       Replicas: 0     Isr: 0
        Topic: thing1   Partition: 3    Leader: 0       Replicas: 0     Isr: 0
        Topic: thing1   Partition: 4    Leader: 0       Replicas: 0     Isr: 0

```

阅读源码可以发现，`KafkaAdmin`在创建后，会从上下文中读取`NewTopic`，然后创建Topic

```
public final boolean initialize() {
	Collection<NewTopic> newTopics = this.applicationContext.getBeansOfType(NewTopic.class, false, false).values();
	...
}
```

# 3.  自定义KafkaTemplate

```
@Bean
public KafkaTemplate<String, String> stringTemplate(ProducerFactory<String, String> pf) {
	return new KafkaTemplate<>(pf);
}

@Bean
public KafkaTemplate<String, byte[]> bytesTemplate(ProducerFactory<String, byte[]> pf) {
	return new KafkaTemplate<>(pf,
		Collections.singletonMap(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class));
}
```

# 4. 发送消息

## 4.1. send

KafkaTemplate提供了很多send的重载方法，主要支持下列参数

- topic：Topic的名字
- partition：分区的id，从0开始。表示指定发送到该分区中
- timestamp：时间戳，一般默认当前时间戳
- key：消息的键
- data：消息的数据
- ProducerRecord：消息对应的封装类，包含上述字段
- Message<?>：Spring自带的Message封装类，包含消息及消息头

sendDefault方法可以想消息发送到默认Topic中，需要在创建KafkaTemplate时指定默认topic

```
@Bean
public KafkaTemplate<String, String> stringTemplate(ProducerFactory<String, String> pf) {
	KafkaTemplate<String, String> kafkaTemplate = new KafkaTemplate<>(pf);
	kafkaTemplate.setDefaultTopic("default-topic");
	return kafkaTemplate;
}
```

## 4.2. 监听器

Spring提供了一个全局监听器，在创建`KafkaTemplate在绑定`kafkaTemplate.setProducerListener(kafkaProducerListener);`

```
public interface ProducerListener<K, V> {

    void onSuccess(ProducerRecord<K, V> producerRecord, RecordMetadata recordMetadata);

    void onError(ProducerRecord<K, V> producerRecord, RecordMetadata recordMetadata,
            Exception exception);

}
```

Spring默认提供了一个`LoggingProducerListener`用于记录异常日志

```
@ConditionalOnMissingBean(KafkaTemplate.class)
public KafkaTemplate<?, ?> kafkaTemplate(ProducerFactory<Object, Object> kafkaProducerFactory,
      ProducerListener<Object, Object> kafkaProducerListener,
      ObjectProvider<RecordMessageConverter> messageConverter) {
   KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<>(kafkaProducerFactory);
   messageConverter.ifUnique(kafkaTemplate::setMessageConverter);
   kafkaTemplate.setProducerListener(kafkaProducerListener);
   kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic());
   return kafkaTemplate;
}

@Bean
@ConditionalOnMissingBean(ProducerListener.class)
public ProducerListener<Object, Object> kafkaProducerListener() {
   return new LoggingProducerListener<>();
}
```

自己实现一个监听器

```
@Component
public class LogProducerListener implements ProducerListener<Object, Object> {

    @Override
    public void onSuccess(ProducerRecord<Object, Object> record, RecordMetadata metadata) {
        StringBuffer logOutput = new StringBuffer();
        logOutput.append("send a message")
                .append(" with key='")
                .append(record.key())
                .append("'")
                .append(" and payload='")
                .append(record.value())
                .append("'")
                .append(" to topic ").append(record.topic());
        if (record.partition() != null) {
            logOutput.append(" and partition ").append(record.partition());
        }
        logOutput.append(":");
        System.out.println(logOutput);
    }

    @Override
    public void onError(ProducerRecord<Object, Object> producerRecord, Exception exception) {

    }
}
```

## 4.3 回调

我们也可以为单个发送消息的方法添加监听器，KafkaTemplate发送消息是采取异步方式发送的，send方法返回的都是`ListenableFuture`

```
public ListenableFuture<SendResult<K, V>> send(String topic, @Nullable V data) {
   ProducerRecord<K, V> producerRecord = new ProducerRecord<>(topic, data);
   return doSend(producerRecord);
}
```

```
ListenableFuture<SendResult<String, String>> future =this.template.send("myTopic", "message1");
future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
  @Override
  public void onFailure(Throwable ex) {
    KafkaProducerException kafkaProducerException = (KafkaProducerException) ex;
    ProducerRecord<String, String> record = kafkaProducerException.getFailedProducerRecord();
  }

  @Override
  public void onSuccess(SendResult<String, String> result) {
    ProducerRecord<String, String> record = result.getProducerRecord();
    RecordMetadata metadata = result.getRecordMetadata();
    StringBuffer logOutput = new StringBuffer();
    logOutput.append("send a message")
            .append(" with key='")
            .append(record.key())
            .append("'")
            .append(" and payload='")
            .append(record.value())
            .append("'")
            .append(" to topic ").append(record.topic());
    if (record.partition() != null) {
      logOutput.append(" and partition ").append(record.partition());
    }
    logOutput.append(":");
    System.out.println(logOutput);
  }
});
```

或者

```java
future.addCallback(result -> {
        ...
    }, (KafkaFailureCallback<Integer, String>) ex -> {
            ProducerRecord<Integer, String> failed = ex.getFailedProducerRecord();
            ...
    });
```

# 5. 接收消息

Spring通过配置MessageListenerContainer和提供消息监听器或使用@KafkaListener来接收消息。

`MessageListener`和它的一系列子接口来实现消息处理的功能（主要区别在commit方式）

`KafkaMessageListenerContainer`使用一个单线程接受所有的topic和partition

`ConcurrentMessageListenerContainer`提供多线程支持（通过委托给一个或多个`KafkaMessageListenerContainer`）

## 5.1. 手动创建监听器

```
  @Bean
  public KafkaMessageListenerContainer<String, String> listenerContainer(ConsumerFactory<String, String> cf) {
    ContainerProperties properties = new ContainerProperties("myTopic");

    properties.setGroupId("consumer-test");

    properties.setMessageListener((MessageListener<String, String>) record
            -> System.out.println("myTopic receive : " + record.toString()));
    KafkaMessageListenerContainer<String, String> listenerContainer = new KafkaMessageListenerContainer<>(cf, properties);
    return listenerContainer;
  }
```

## 5.2. @KafkaListener

`@KafkaListener`注解的方法支持下列参数

- data ： 对于data值的类型其实并没有限定，根据KafkaTemplate所定义的类型来决定。data为List集合的则是用作批量消费
- ConsumerRecord：具体消费数据类，包含Headers信息、分区信息、时间戳等
- Acknowledgment：用作Ack机制的接口
- Consumer：消费者类，使用该类我们可以手动提交偏移量、控制消费速率等功能

`@KafkaListener`的注解都提供了下列属性

- id：消费者的id，当GroupId没有被配置的时候，默认id为GroupId
- containerFactory：上面提到了@KafkaListener区分单数据还是多数据消费只需要配置一下注解的containerFactory属性就可以了，这里面配置的是监听容器工厂，也就是`ConcurrentKafkaListenerContainerFactory`的BeanName
- topics：需要监听的Topic，可监听多个
- topicPartitions：可配置更加详细的监听信息，必须监听某个Topic中的指定分区，或者从offset为200的偏移量开始监听
- errorHandler：监听异常处理器
- groupId：消费组ID
- idIsGroup：id是否为GroupId
- clientIdPrefix：消费者Id前缀
- beanRef：真实监听容器的BeanName，需要在 BeanName前加 "__"

## 5.3. 批量接收

```
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> listenerContainer(ConsumerFactory<String, String> cf) {
  ConcurrentKafkaListenerContainerFactory<String, String> listenerContainer = new ConcurrentKafkaListenerContainerFactory<>();
  listenerContainer.setConsumerFactory(new DefaultKafkaConsumerFactory<>(cf));
  listenerContainer.setConcurrency(1);
  listenerContainer.setBatchListener(true);
  return listenerContainer;
}

@KafkaListener(topics = "myTopic")
public void processMessage4(List<String> content) {
	System.out.println(content);
}
```

## 5.4. 指定分区、偏移量

```
@KafkaListener(topicPartitions = {
        @TopicPartition(topic = "myTopic", partitions = {"0"})
})
public void processMessage5(String content) {
    System.out.println(content);
}

@KafkaListener(topicPartitions = {
		@TopicPartition(topic = "myTopic", partitionOffsets = {
				@PartitionOffset(partition = "0", initialOffset = "10")
		})
})
public void processMessage5(String content) {
	System.out.println(content);
}

// 同一个分区不能指定两次
@KafkaListener(topicPartitions = {
		@TopicPartition(topic = "myTopic", partitions = {"0"}, partitionOffsets = {
				@PartitionOffset(partition = "1", initialOffset = "10")
		})
})
public void processMessage5(String content) {
	System.out.println(content);
}
```

## 5.5. 获取消息头及消息体

我们还可以在方法参数上增加注解获取消息头及消息体（ConsumerRecord也可以获得）

- @Payload：获取的是消息的消息体，也就是发送内容
- @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY)：获取发送消息的key
- @Header(KafkaHeaders.RECEIVED_PARTITION_ID)：获取当前消息是从哪个分区中监听到的
- @Header(KafkaHeaders.RECEIVED_TOPIC)：获取监听的TopicName
- @Header(KafkaHeaders.RECEIVED_TIMESTAMP)：获取时间戳

```
@KafkaListener(topics = "myTopic")
public void processMessage6(@Payload String data,
                               @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) Integer key,
                               @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                               @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                               @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long ts) {
       System.out.println("receive : \n"+
               "data : "+data+"\n"+
               "key : "+key+"\n"+
               "partitionId : "+partition+"\n"+
               "topic : "+topic+"\n"+
               "timestamp : "+ts+"\n"
       );
```

## 5.6. ACK

Kafka通过最新保存偏移量进行消息消费，而且确认消费的消息并不会立刻删除，所以我们可以重复的消费未被删除的数据，当第一条消息未被确认，而第二条消息被确认的时候，Kafka会保存第二条消息的偏移量，也就是说第一条消息再也不会被监听器所获取，除非是根据第一条消息的偏移量手动获取。

使用Kafka的Ack机制比较简单，只需简单的三步即可：

1. 设置ENABLE_AUTO_COMMIT_CONFIG=false，禁止自动提交
2. 设置AckMode=MANUAL_IMMEDIATE
3. 监听方法加入Acknowledgment ack 参数

```
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> listenerContainer(ConsumerFactory<String, String> cf) {
	ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
	factory.setConsumerFactory(cf);
	factory.setConcurrency(1);
//    factory.setBatchListener(true);
	factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
	return factory;
}
  
@KafkaListener(topics = "myTopic", containerFactory = "listenerContainer")
public void processMessage7(ConsumerRecord record,  Acknowledgment ack) {
	System.out.println(record);
	ack.acknowledge();
}
```

如果我们不执行`ack.acknowledge();`每次启动都会重新读取消息

# 6. Reply

Spring提供了`ReplyingKafkaTemplate`用了实现请求/响应模式的消息通信

创建`ReplyingKafkaTemplate`

```
@Configuration
public class ReplyConfiguration {
    @Bean
    public ReplyingKafkaTemplate<String, String, String> replyingTemplate(
            ProducerFactory<String, String> pf,
            ConcurrentMessageListenerContainer<String, String> repliesContainer) {

        return new ReplyingKafkaTemplate<>(pf, repliesContainer);
    }

    @Bean
    public ConcurrentMessageListenerContainer<String, String> repliesContainer(
            ConcurrentKafkaListenerContainerFactory<String, String> containerFactory) {
		// 指定回复的主题
        ConcurrentMessageListenerContainer<String, String> repliesContainer =
                containerFactory.createContainer("replies");
        // 设置消费方的配置
        repliesContainer.getContainerProperties().setGroupId("repliesGroup");
        repliesContainer.setAutoStartup(false);
        return repliesContainer;
    }

    @Bean
    public NewTopic kRequests() {
        return TopicBuilder.name("kRequests")
                .build();
    }

}
```

发送消息

```
ProducerRecord<String, String> record = new ProducerRecord<>("kRequests", "foo");
RequestReplyFuture<String, String, String> replyFuture = template.sendAndReceive(record);
SendResult<String, String> sendResult = replyFuture.getSendFuture().get(10, TimeUnit.SECONDS);
System.out.println("Sent ok: " + sendResult.getRecordMetadata());
ConsumerRecord<String, String> consumerRecord = replyFuture.get(10, TimeUnit.SECONDS);
System.out.println("Return value: " + consumerRecord.value());
```

可以看到发送的消息增加了2个消息头`kafka_replyTopic`，`kafka_correlationId`

```
ProducerRecord(topic = kRequests, partition = 0, leaderEpoch = 0, offset = 3, CreateTime = 1608002315669, serialized key size = -1, serialized value size = 3, headers = RecordHeaders(headers = [RecordHeader(key = kafka_replyTopic, value = [114, 101, 112, 108, 105, 101, 115]), RecordHeader(key = kafka_correlationId, value = [-4, -117, -53, 59, 17, -26, 66, 69, -95, 89, -36, 35, 56, 78, -8, 117])], isReadOnly = false), key = null, value = foo)
```

这`kafka_replyTopic`两个消息头也可以自行指定

```
ProducerRecord<String, String> record = new ProducerRecord<>("kRequests", "foo");
record.headers().add(KafkaHeaders.REPLY_TOPIC, "custom_reply".getBytes());
```

```
ProducerRecord(topic = kRequests, partition = 0, leaderEpoch = 0, offset = 4, CreateTime = 1608002920455, serialized key size = -1, serialized value size = 3, headers = RecordHeaders(headers = [RecordHeader(key = kafka_replyTopic, value = [99, 117, 115, 116, 111, 109, 95, 114, 101, 112, 108, 121]), RecordHeader(key = kafka_correlationId, value = [-100, -19, -37, -51, 11, -101, 64, -85, -113, -9, -44, -114, 119, 62, 79, -17])], isReadOnly = false), key = null, value = foo)
```

读取消息并回复

```
@KafkaListener(id="server", topics = "kRequests")
@SendTo // use default replyTo expression
public String listen(String in) {
	System.out.println("Server received: " + in);
	return in.toUpperCase();
}
```

也可以直接返回Message

```
@KafkaListener(topics = "kRequests")
@SendTo // use default replyTo expression
public Message<String> listen(String in) {
	System.out.println("Server received: " + in);
	return MessageBuilder.withPayload(in.toUpperCase())
			.build();
}
```

> 读取消息我是新创建的工程，如果和发送消息在一起，需要单独创建ConcurrentKafkaListenerContainerFactory，并设置ReplyTemplate

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<Integer, String> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(cf());
    factory.setReplyTemplate(template());
    return factory;
}
```

> ReplyingKafkaTemplate仅支持一对一的响应，如果一个请求需要回复多个响应，可以使用AggregatingReplyingKafkaTemplate

# 7. pause、resume

如果消息产生的远远原因超过消息消费的速度，可能会导致内存溢出。我们可以通过pause方法暂停消费，resume方法恢复消费

> pause方法会继续发生心跳请求

`@KafkaListener`这个注解所标注的方法并没有在IOC容器中注册为Bean，而是会被注册在`KafkaListenerEndpointRegistry`中

```
registry.getListenerContainer("pause.resume").pause();

registry.getListenerContainer("pause.resume").resume();

@KafkaListener(id = "pause.resume", topics = "pause.resume.topic")
public void processMessage8(String content) {
	System.out.println(String.format("%d:%s", Instant.now().getEpochSecond(), content));
}
```

# 8.手动设置偏移量

ConsumerSeekAware提供了手动设置偏移量的方法

```
// called when the container is started and whenever partitions are assigned
void registerSeekCallback(ConsumerSeekCallback callback);

// called when partitions are assigned
void onPartitionsAssigned(Map<TopicPartition, Long> assignments, ConsumerSeekCallback callback);

// called when the container is stopped or Kafka revokes assignments
void onPartitionsRevoked(Collection<TopicPartition> partitions)

void onIdleContainer(Map<TopicPartition, Long> assignments, ConsumerSeekCallback callback);
```

```
void seek(String topic, int partition, long offset);

void seekToBeginning(String topic, int partition);

void seekToBeginning(Collection=<TopicPartitions> partitions);

void seekToEnd(String topic, int partition);

void seekToEnd(Collection=<TopicPartitions> partitions);

void seekRelative(String topic, int partition, long offset, boolean toCurrent);

void seekToTimestamp(String topic, int partition, long timestamp);

void seekToTimestamp(Collection<TopicPartition> topicPartitions, long timestamp);
```

示例

```
@Component
class Listener implements ConsumerSeekAware {

    private static final Logger logger = LoggerFactory.getLogger(Listener.class);

    private final ThreadLocal<ConsumerSeekCallback> callbackForThread = new ThreadLocal<>();

    private final Map<TopicPartition, ConsumerSeekCallback> callbacks = new ConcurrentHashMap<>();

    @Override
    public void registerSeekCallback(ConsumerSeekCallback callback) {
        // called when the container is started and whenever partitions are assigned
        this.callbackForThread.set(callback);
    }

    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments, ConsumerSeekCallback callback) {
        // called when partitions are assigned
        assignments.keySet().forEach(tp -> this.callbacks.put(tp, this.callbackForThread.get()));

        TopicPartition tp = new TopicPartition("myTopic", 0);
        if (assignments.containsKey(tp)) {
            callback.seekToBeginning(Collections.singletonList(tp));
        }
    }

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // called when the container is stopped or Kafka revokes assignments
        partitions.forEach(tp -> this.callbacks.remove(tp));
        this.callbackForThread.remove();
    }

    @Override
    public void onIdleContainer(Map<TopicPartition, Long> assignments, ConsumerSeekCallback callback) {
    }

    @KafkaListener(topics = "myTopic")
    public void processMessage(String content) {
        System.out.println(content);
    }

}
```

**注意：@KafkaListener要实现在ConsumerSeekAware内部**

我们也可以将`ConsumerSeekAware`注入到其他Bean中调用

```
public void seekToStart() {
	this.callbacks.forEach((tp, callback) -> callback.seekToBeginning(tp.topic(), tp.partition()));
}
listener.seekToStart();
```

Spring提供了`AbstractConsumerSeekAware`类已经实现了上述保存callback的功能。

```
@Component
class Listener2 extends AbstractConsumerSeekAware {

    @KafkaListener(topics = "myTopic")
    public void processMessage(String content) {
        System.out.println(content);
    }

    public void seekToBeginning() {
        getSeekCallbacks().forEach((tp, callback) -> callback.seekToBeginning(tp.topic(), tp.partition()));
    }

}
```

# 9. 异常处理

KafkaListener要做的事只是监听Topic中的数据并消费，如果在KafkaListener中还需要对异常进行处理则会显得代码块非常臃肿不利于维护，我们可以把异常处理的这些代码抽象出来，构造成一个异常处理器，KafkaListener中所抛出的异常都会经过ConsumerAwareErrorHandler异常处理器进行处理，这样就非常方便我们进行后期维护，比如后期更改异常处理业务的时候，只需要修改ConsumerAwareErrorHandler处理器就行了，而不需要KafkaListener的一堆代码中去修改代码。

```
@Component
public class CustomExceptionHandler implements ConsumerAwareListenerErrorHandler {

    @Override
    public Object handleError(Message<?> message, ListenerExecutionFailedException exception, Consumer<?, ?> consumer) {
        StringBuffer logOutput = new StringBuffer();
        logOutput.append("cosume a message failed")
                .append(" , payload='")
                .append(message.getPayload())
                .append("'");
        System.out.println(logOutput);
        return null;
    }
}
```

```
@KafkaListener(topics = "myTopic", errorHandler = "customExceptionHandler")
public void processMessage9(String content) {
	System.out.println(content);
	throw new RuntimeException("occur err");
}
```

