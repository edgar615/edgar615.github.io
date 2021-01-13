---
layout: post
title: 安装Prometheus和Grafana
date: 2020-12-01
categories:
	- prometheus
comments: true
permalink: prometheus.html
---

# 1. Prometheus

直接到官网下载prometheus后解压

直接启动

```
# 默认存储路径是./data
./prometheus --config.file=prometheus.yml  -–storage.tsdb.path=/server/data/prometheus
```

也可以将prometheus放在/user/local/bin中

```
tar -zxvf prometheus-2.24.0.linux-amd64.tar.gz
cd prometheus-2.24.0.linux-amd64
cp prometheus promtool  /usr/local/bin/
prometheus --version
prometheus, version 2.24.0 (branch: HEAD, revision: 02e92236a8bad3503ff5eec3e04ac205a3b8e4fe)
  build user:       root@d9f90f0b1f76
  build date:       20210106-13:48:37
  go version:       go1.15.6
  platform:         linux/amd64
  
# 配置文件
mkdir -p /etc/prometheus
cp prometheus.yml /etc/prometheus/
promtool  check config /etc/prometheus/prometheus.yml
Checking /etc/prometheus/prometheus.yml
  SUCCESS: 0 rule files found
  
# 启动
prometheus --config.file=/etc/prometheus/prometheus.yml  --storage.tsdb.path=/server/data/prometheus
```

# 1.1. 配置说明

```
global:
  scrape_interval:     15s # 设置抓取间隔，默认为1分钟
  evaluation_interval: 15s #估算规则的默认周期，每15秒计算一次规则。默认1分钟
  # scrape_timeout  #默认抓取超时，默认为10s

# Alertmanager相关配置
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# 规则文件列表，使用'evaluation_interval' 参数去抓取
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

#  抓取配置列表
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
```

# 1.2. 使用systemd来启停prometheus

```
vim /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/server/data/prometheus
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

# 2. **Grafana**

虽然说prometheus能展示一些图表，但对比Grafana，那只是个过家家。接下来我们需要在同一个服务器上安装Grafana服务，用来展示prometheus收集到的数据。

```
wget https://dl.grafana.com/oss/release/grafana-7.3.6.linux-amd64.tar.gz
tar -zxvf grafana-7.3.6.linux-amd64.tar.gz
```



# 3. 测试

下载node_exporter-1.0.1.linux-amd64.tar.gz并解压

```
./node_exporter --web.listen-address 127.0.0.1:8080
./node_exporter --web.listen-address 127.0.0.1:8081
./node_exporter --web.listen-address 127.0.0.1:8082
```

