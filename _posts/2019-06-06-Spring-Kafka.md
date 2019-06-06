---
layout: post
title: Spring集成kafka
date: 2019-06-06
categories:
    - Spring Boot
    - Spring Cloud
comments: true
permalink: Spring-kafka.html
---

公司项目用的MQ是kafka，封装了一个消息组件。顺便了解下spring与kafka的集成，并未在项目中使用

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
<pre class="line-numbers "><code class="language-java">
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
</code></pre>

增加消费消息的类
<pre class="line-numbers "><code class="language-java">
@Component
public class MessageReceiver {

	@KafkaListener(topics = "myTopic")
	public void processMessage(String content) {
		System.out.println(content);
	}
}
</code></pre>

测试
<pre class="line-numbers "><code class="language-java">
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
</code></pre>

一些配置项就不写了，要用的时候在补充

参考资料

https://majing.io/posts/10000005201221

https://blog.csdn.net/qq_32734365/article/details/81413218