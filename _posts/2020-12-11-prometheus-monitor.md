---
layout: post
title: prometheus监控一些常用组件
date: 2020-12-11
categories:
    - prometheus
comments: true
permalink: prometheus-monitor.html
---

# 1. 黑盒

> 基本复制自https://mp.weixin.qq.com/s/DMnIC2QFE094AWU1_6ebOA

blackbox_exporter 是 Prometheus 拿来对 http/https、tcp、icmp、dns、进行的黑盒监控工具

到https://github.com/prometheus/blackbox_exporter/releases下载最新版本

**常用参数**

```
./blackbox_exporter --help
usage: blackbox_exporter [<flags>]

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --config.file="blackbox.yml"  
                                 Blackbox exporter configuration file.
      --web.listen-address=":9115"  
                                 The address to listen on for HTTP requests.

      --log.level=info           Only log messages with the given severity or above. One of: [debug, info, warn, error]
```

**启动**

```
# 默认端口为9115
nohup ./blackbox_exporter --config.file="blackbox.yml" 2>&1 &
```

**docker启动**

```
# 如果你不需要开 debug，请去掉最后的 --log.level=debug
docker run --rm -d -p 9115:9115 --name blackbox_exporter -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime:ro -v /data/prometheus/blackbox_exporter/blackbox.yml:/config/blackbox.yml prom/blackbox-exporter:master --config.file=/config/blackbox.yml --log.level=debug
```

**配置文件**

```
# 官方默认的配置文件
modules:
  http_2xx:
    prober: http
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
```

**prometheus配置**

http

```
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # 模块对应 blackbox.yml 
    static_configs:
      - targets:
        - http://baidu.com    # http
        - https://baidu.com   # https
        - http://xx.com:8080 # 8080端口的域名
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # blackbox安装在哪台机器
```

TCP

```
  - job_name: blackbox_tcp
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
        - 192.168.1.2:280
        - 192.168.1.2:7013

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.1.99:9115 # Blackbox exporter.
```

**告警规则**

> 可以从https://awesome-prometheus-alerts.grep.to/rules#blackbox这里找规则

```
# 以下两条二选一
groups:
  - name: http
    rules:
    - alert: xxx域名解析失败
      expr: probe_success{instance="https://xx.com"} == 0
      for: 1m
      labels:
        severity: "error"
      annotations:
        summary: "xxx域名解析失败"
    - alert: xxx域名解析失败
      expr: probe_http_status_code{instance="https://xx.com"} != 200
      for: 5m
      labels:
        severity: "error"
      annotations:
        summary: "xxx域名解析失败"
```

**自定义模块**

有时可能对于某些 URL 需要带参数，如 header、body 之类的，就需要自定义一个模块

`blackbox.yml`

```
  http_2xx_wxjj:
    prober: http
    timeout: 5s
    http:
      method: GET
      headers:
        Cookie: JSESSIONID=C123455dfdf
        sid: 41c912344555-24rkjkffd
        appid: 1221kj2h1k3hjk13hk
      body: '{}'
```

`Prometheus.yml`

```
  - job_name: 'blackbox_wxjl'
    metrics_path: /probe
    params:
      module: [http_2xx_wxjj]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://192.168.201.173:808/byxxxxx/41234456661f-4357c9?head=APP_GeList&user=%E9%BB%84%E5%AE%15
   # Target to probe with http.

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 172.18.11.154:9115  # The blackbox exporter's real hostname:port.
```

**开启 debug**

当你觉得自己设置没错，http 状态码却返回不正确，想要调试一下，就需要打开debug。

- 启动时指定 --log.level=debug
- targets 后面带上 &debug=true，即  http://172.18.11.154:9115/probe?module=http_2xx_wxjj&target=http://192.168.201.173:808/byxxxxx/41234456661f-4357c9?head=APP_GeList&user=%E9%BB%84%E5%AE%15&debug=tru

targets 开启 debug 会比正常链接输出更多信息

**Grafana**

id: 7587

![](/assets/images/posts/prometheus-export/blackbox-1.png)

# 2. MySQL

监控MySQL实例可以分为2种部署方式：

- **分离部署+环境变量**

这种方式是在每个mysql服务器上跑一个exporter程序，比如10.10.20.14服务器上跑自己的mysqld_exporter，而登到10.10.20.15服务器上也启动自己的mysqld_exporter，也就是分离部署，这样的话每个mysql服务器上除了mysqld进程外还会多一个mysqld_exporter的进程。

- **集中部署+配置文件**

如果我们想要保持mysql服务器零入侵的纯净环境，这时候就可以尝试一下集中部署+配置文件的方式。集中部署，就是说我们将所有的mysqld_exporter部署在同一台服务器上，在这台服务器上对mysqld_exporter进行统一的管理。

**添加一个用户授权**

```
# 建议设置对用户的最大连接数限制，以免在高峰期导致业务挂掉
CREATE USER 'exporter'@'localhost' IDENTIFIED BY '123456' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

**使用环境变量启动**

```
export DATA_SOURCE_NAME='exporter:123456@(localhost:3306)/'
nohup ./mysqld_exporter &
```

**使用配置文件 my.cnf 启动**

```
cat <<EOF> my.cnf
[client]
user=exporter
password=123456
EOF

nohup ./mysqld_exporter --config.my-cnf=my.cnf 2>&1 &
```

`curl http://localhost:9104/metrics` 即可访问到数据

**添加到 Prometheus**

```
 - job_name: 'mysqld'
   static_configs:
    - targets:
      - 192.168.159.131:9104
```

**添加到 Grafana**

ID: 7362

![](/assets/images/posts/prometheus-export/mysql-1.png)

**告警规则**

https://awesome-prometheus-alerts.grep.to/rules#mysql

**参考资料**

https://github.com/prometheus/mysqld_exporter

# 3. Consul

启动

```
nohup ./consul_exporter --consul.server="http://localhost:8500" 2>&1 &

```

`curl http://localhost:9107/metrics` 即可访问到数据

**添加到 Prometheus**

```
 - job_name: 'consul'
   static_configs:
    - targets:
      - 192.168.159.131:9107
```

**添加到 Grafana**

ID: 13396

**告警规则**

https://awesome-prometheus-alerts.grep.to/rules#consul

**参考资料**

https://github.com/prometheus/consul_exporter

# 4. Redis

# 5. Nginx

# 6. JVM

# 7. Spring

# 8. Kafka

# 9.  Zookeeper



