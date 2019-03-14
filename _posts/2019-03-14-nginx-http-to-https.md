---
layout: post
title: nginx配置http自动跳转到https
date: 2019-03-14
categories:
    - nginx
comments: true
permalink: http-to-https.html
---


# rewrite

```
    server {  
        listen  192.168.1.111:80;  
        server_name test.com;  
          
        rewrite ^(.*)$  https://$host$1 permanent;  
    }  
```

# 497错误码

解释：当此虚拟站点只允许https访问时，当用http访问时nginx会报出497错误码

```
497 - normal request was sent to HTTPS  
```

```
    server {  
        listen       192.168.1.11:443;  #ssl端口  
        listen       192.168.1.11:80;   #用户习惯用http访问，加上80，后面通过497状态码让它自动跳到443端口  
        server_name  test.com;  
        #为一个server{......}开启ssl支持  
        ssl                  on;  
        #指定PEM格式的证书文件   
        ssl_certificate      /etc/nginx/test.pem;   
        #指定PEM格式的私钥文件  
        ssl_certificate_key  /etc/nginx/test.key;  
          
        #让http请求重定向到https请求   
        error_page 497  https://$host$uri?$args;  
    }  
```

# index刷新

```
    <html>  
    <meta http-equiv="refresh" content="0;url=https://test.com/">  
    </html>  
```

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
