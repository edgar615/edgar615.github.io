---
layout: post
title: Consul - Registrator
date: 2019-04-06
categories:
    - Consul
comments: true
permalink: consul-registrator.html
---

http://gliderlabs.github.io/registrator/latest/



启动

```
docker run -d \
    --name=registrator \
    --net=host \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
      consul://localhost:8500
```

## Docker Options

| Option                                           | Required    | Description                                      |
| ------------------------------------------------ | ----------- | ------------------------------------------------ |
| `--volume=/var/run/docker.sock:/tmp/docker.sock` | yes         | Allows Registrator to access Docker API          |
| `--net=host`                                     | recommended | Helps Registrator get host-level IP and hostname |

An alternative to host network mode would be to set the container hostname to the hosthostname (`-h $HOSTNAME`) and using the `-ip` Registrator option below.

## Registrator Options

| Option                           | Since | Description                                                  |
| -------------------------------- | ----- | ------------------------------------------------------------ |
| `-cleanup`                       | v7    | Cleanup dangling services，建议带上                          |
| `-deregister <mode>`             | v6    | Deregister exited services "always" or "on-success". Default: always |
| `-internal`                      |       | Use exposed ports instead of published ports                 |
| `-ip <ip address>`               |       | Force IP address used for registering services               |
| `-resync <seconds>`              | v6    | Frequency all services are resynchronized. Default: 0, never |
| `-retry-attempts <number>`       | v7    | Max retry attempts to establish a connection with the backend |
| `-retry-interval <milliseconds>` | v7    | Interval (in millisecond) between retry-attempts             |
| `-tags <tags>`                   | v5    | Force comma-separated tags on all registered services        |
| `-ttl <seconds>`                 |       | TTL for services. Default: 0, no expiry (supported backends only) |
| `-ttl-refresh <seconds>`         |       | Frequency service TTLs are refreshed (supported backends only) |
| `-useIpFromLabel <label>`        |       | Uses the IP address stored in the given label, which is assigned to a container, for registration with Consul |

示例

```
docker run -d --name=registrator --net=host --volume=/var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest -internal -cleanup consul://localhost:8500 
```