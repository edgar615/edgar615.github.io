---
layout: post
title: 通过程序重启spring boot
date: 2019-03-10
categories:
    - Spring Boot
    - Spring Cloud
comments: true
permalink: spring-restart.html
---

方式1

<pre class="line-numbers "><code class="language-java">
@SpringBootApplication
public class Application {

    private static ConfigurableApplicationContext context;

    public static void main(String[] args) {
        context = SpringApplication.run(Application.class, args);
    }

    public static void restart() {
        ApplicationArguments args = context.getBean(ApplicationArguments.class);

        Thread thread = new Thread(() -> {
            context.close();
            context = SpringApplication.run(Application.class, args.getSourceArgs());
        });

        thread.setDaemon(false);
        thread.start();
    }
}
</code></pre>

<pre class="line-numbers "><code class="language-java">
@RestController
public class RestartController {

    @PostMapping("/restart")
    public void restart() {
        Application.restart();
    }
}
</code></pre>

<pre class="line-numbers"><code class="language-shell">
$ curl -s -X POST localhost:8080/restart
</code></pre>

方式2 RestartEndpoint
引入依赖
```
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-context</artifactId>
      <version>${cloud.start.version}</version>
    </dependency>
```
配置
```
management:
  endpoint:
    restart:
      enabled: true
  endpoints:
    web:
      exposure:
        include: "*"
```
<pre class="line-numbers"><code class="language-shell">
$ curl -s -X POST localhost:8080/actuator/restart
{"message":"Restarting"}
</code></pre>
当然也可以通过`RestartEndpoint`来在任意地方重启应用
<pre class="line-numbers "><code class="language-java">
@Service
public class RestartService {
     
    @Autowired
    private RestartEndpoint restartEndpoint;
     
    public void restartApp() {
        restartEndpoint.restart();
    }
}
</code></pre>
