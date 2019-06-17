---
layout: post
title: spring boot配置文件生成随机数
date: 2019-03-19
categories:
    - Spring
comments: true
permalink: Random-Configuration-Property-Values.html
---

spring通过`RandomValuePropertySource`实现了配置文件的随机数（源码很简单），我们可以在配置文件的value中通过`${random.<type>}`来生成随机数，目前仅支持`String`, `Integer`, `Long` 和 `UUID`四种类型，还可以通过`${random.int[<start>,<end>]}`来指定随机数的范围


```
my:
  secret:
    password: ${random.value}
    intValue: ${random.int}
    intValueRange: ${random.int[1,99]}
    longValue: ${random.long}
    longValueRange: ${random.long[111111111111,999999999999]}
    uuid: ${random.uuid}
```

测试

<pre class="line-numbers "><code class="language-java">
@SpringBootApplication
public class Application {


    @Value("${my.secret.password}")
    private String password;

    @Value("${my.secret.intValue}")
    private Integer intValue;

    @Value("${my.secret.intValueRange}")
    private Integer intValueRange;

    @Value("${my.secret.longValue}")
    private Long longValue;

    @Value("${my.secret.longValueRange}")
    private Long longValueRange;

    @Value("${my.secret.uuid}")
    private UUID uuid;

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }

    @PostConstruct
    private void init(){
        System.out.println("Configure Random Property Values using Spring Boot");
        System.out.println("password: " + password);
        System.out.println("intValue: " + intValue);
        System.out.println("intValueRange: " + intValueRange);
        System.out.println("longValue: " + longValue);
        System.out.println("longValueRange: " + longValueRange);
        System.out.println("uuid: " + uuid);
    }

}
</code></pre>

输出
```
Configure Random Property Values using Spring Boot
password: 688f4360f45e6b95b37db3a53752e663
intValue: -1218213620
intValueRange: 14
longValue: 2523323419993803822
longValueRange: 821223638186
uuid: 0f916df1-c838-4615-beb6-7ae6f5ab40aa
```


参考资料

https://memorynotfound.com/spring-boot-random-configuration-property-values/