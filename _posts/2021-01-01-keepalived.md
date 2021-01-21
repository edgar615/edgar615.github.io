---
layout: post
title: 高可用解决方案 - Keepalived
date: 2021-01-01
categories:
    - 架构设计
comments: true
permalink: keepalived.html
---

# 1. Keepalived介绍

**Keepalived**是基于**vrrp**协议的一款高可用软件。它的作用是检测服务器的状态，如果有一台web服务器宕机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的服务器。

VRRP(Virtual RouterRedundancy Protocol)协议是用于实现路由器冗余的协议， VRRP  协议将两台或多台路由器设备虚拟成一个设备，对外提供虚拟路由器 IP(一个或多个)，而在路由器组内部，如果实际拥有这个对外 IP  的路由器如果工作正常的话就是 MASTER，或者是通过算法选举产生， MASTER 实现针对虚拟路由器 IP 的各种网络功能，如 ARP 请求， ICMP，以及数据的转发等；其他设备不拥有该虚拟 IP，状态是 BACKUP，除了接收 MASTER 的VRRP  状态通告信息外，不执行对外的网络功能。当主机失效时， BACKUP 将接管原先 MASTER 的网络功能。VRRP 协议使用多播数据来传输  VRRP 数据， VRRP 数据使用特殊的虚拟源 MAC 地址发送数据而不是自身网卡的 MAC 地址， VRRP 运行时只有 MASTER  路由器定时发送 VRRP 通告信息，表示 MASTER 工作正常以及虚拟路由器 IP(组)， BACKUP 只接收 VRRP  数据，不发送数据，如果一定时间内没有接收到 MASTER 的通告信息，各 BACKUP 将宣告自己成为 MASTER，发送通告信息，重新进行  MASTER 选举状态。

**keepalived高可用架构示意图**

![](/assets/images/posts/keepalived/keepalived-1.png)

**keepalived的工作原理**

![](/assets/images/posts/keepalived/keepalived-2.jpg)

- `Checkers`：对**后端服务器**节点（RS）进行健康状态监测和故障隔离，我们知道，LVS本身是没有健康状态监测功能，keepalived起初就是为LVS生成ipvs规则和增加健康状态检测功能而设计的。
- `VRRP Stack`：这是keepalived实现VRRP功能的模块，VRRP功能可以实现对**前端调度器**集群的故障切换(failover)。
- `SMTP`：Checkers和VRRP Stack都是对节点进行状态监测，不同的是监测前端调度器和监测后端RS，但是这两个模块都可以通过SMTP通知管理员故障信息的邮件。
- `System Call`：Checkers和VRRP Stack同样都可以调用系统内核功能完成故障隔离和故障切换。
- `IPVS wrapper`：Checkers通过监测后端RS的工作状态得出信息，IPVS wrapper通过这些信息添加ipvs规则，内核中的IPVS则通过这些规则进行工作。
- `Netlink Reflector`：在调度器发生故障切换的时候，该模块充当调用内核NETLINK的接口，完成虚拟IP的设置和切换。
- `Watch Dog`：keepalived的核心模块就是Checkers和VRRP Stack，当这两个模块发生故障的时候怎么办呢，这时候Watch Dog就发生了作用，它是一个硬件检测工具，一旦Checkers和VRRP  Stack发生故障，Watch Dog就能够检测到并采取恢复措施（重启）。

# 2. 安装配置

```
cd keepalived-2.2.1
./configure
make && make install
```

KeepAlived的所有配置都在一个配置文件里设置，支持的配置可分为以下三类：

- 全局配置（global configure）
- VRRPD配置
- LVS配置

很明显，全局配置就是对整个keepalived生效的配置，不管是否使用LVS，VRRPD是keepalived的核心，LVS配置只在要使用keepalived来配置和管理LVS时使用，如果仅使用keepalived来做HA，LVS不需要配置。

配置文件都是以块（block）形式组织的，每个块都在{}范围内，#和!表示注释。

**MASTER**

```
#全局配置
global_defs {
   #keepalived切换的时候，发消息到指定的email，可配置多个email
   notification_email {
     edgar@github.com
     edgar@github.com
   }
   #通知邮件从哪个地址发出
   notification_email_from edgar@github.com
   #通知邮件的smtp地址
   smtp_server smtp.exmail.github.com
   #连接smtp服务器的超时时间，单位秒
   smtp_connect_timeout 30
   #Keepalived的机器标识，一个网络内保持唯一，通常为 hostname
   router_id server-1
}

## keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级。
## 如果脚本执行结果为 0，并且 weight 配置的值大于 0，则优先级相应的增加。
## 如果脚本执行结果非 0，并且 weight配置的值小于 0，则优先级相应的减少。
## 其他情况，维持原本配置的优先级，即配置文件中 priority 对应的值。
vrrp_script chk_nginx {
	script "/etc/keepalived/nginx_check.sh" ## 检测 nginx 状态的脚本路径
	interval 2 ## 检测时间间隔
	weight -20 ## 如果条件成立，权重-20
}

#keepalived实例配置
vrrp_instance VI_1 {
    #指定实例的初始状态，MASTER或BACKUP两种状态，并且需要大写
    state MASTER
    #实例绑定的网卡
    interface ens33
    #虚拟路由标识，是一个数字，整个VRRP内唯一，如果keepalived配置了主备，需要相同
    virtual_router_id 51
    #优先级，数值愈大，优先级越高
    priority 100
    #MASTER与BACKUP之间同步检查的时间间隔，单位为秒
    advert_int 1
    #认证权限密码，防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    #追踪外围脚本
    track_script {
        #这里配置vrrp_script的名称
        chk_nginx
    }
    #虚拟ip配置，可配置多个
    virtual_ipaddress {
        192.168.159.141
    }
}
```

启动keepalived后，可以看到多了一个IP：192.168.159.141

```
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:fa:bc:99 brd ff:ff:ff:ff:ff:ff
    inet 192.168.159.131/24 brd 192.168.159.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 192.168.159.141/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fefa:bc99/64 scope link
       valid_lft forever preferred_lft forever

```

**BACKUP**

备份机设置基本相同

```
#全局配置
global_defs {
   #keepalived切换的时候，发消息到指定的email，可配置多个email
   notification_email {
     edgar@github.com
     edgar@github.com
   }
   #通知邮件从哪个地址发出
   notification_email_from edgar@github.com
   #通知邮件的smtp地址
   smtp_server smtp.exmail.github.com
   #连接smtp服务器的超时时间，单位秒
   smtp_connect_timeout 30
   #Keepalived的机器标识，一个网络内保持唯一，通常为 hostname
   router_id server-2
}

## keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级。
## 如果脚本执行结果为 0，并且 weight 配置的值大于 0，则优先级相应的增加。
## 如果脚本执行结果非 0，并且 weight配置的值小于 0，则优先级相应的减少。
## 其他情况，维持原本配置的优先级，即配置文件中 priority 对应的值。
vrrp_script chk_nginx {
	script "/etc/keepalived/nginx_check.sh" ## 检测 nginx 状态的脚本路径
	interval 2 ## 检测时间间隔
	weight -20 ## 如果条件成立，权重-20
}

#keepalived实例配置
vrrp_instance VI_1 {
    #指定实例的初始状态，MASTER或BACKUP两种状态，并且需要大写
    state BACKUP
    #实例绑定的网卡
    interface ens33
    #虚拟路由标识，是一个数字，整个VRRP内唯一，如果keepalived配置了主备，需要相同
    virtual_router_id 51
    #优先级，数值愈大，优先级越高
    priority 90
    #MASTER与BACKUP之间同步检查的时间间隔，单位为秒
    advert_int 1
    #认证权限密码，防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    #追踪外围脚本
    track_script {
        #这里配置vrrp_script的名称
        chk_nginx
    }
    #虚拟ip配置，可配置多个
    virtual_ipaddress {
        192.168.159.141
    }
}
```

启动keepalived后，并没有虚IP

```
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:56:38:7e:d7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.159.132/24 brd 192.168.159.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe38:7ed7/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
```

此时我们访问http://192.168.159.141/发现访问的server-1的Nginx服务。停掉server-1的keepalived，可以发现虚IP跑到了server-2上，在访问http://192.168.159.141，就发现访问的是server-2的Nginx服务

在启动server-1的keepalived，虚IP又回到了server-1上。

Nginx 状态检查脚本**check_nginx.sh**

如果 nginx 停止运行，尝试启动，如果无法启动则杀死本机的 keepalived 进程， keepalived将虚拟 ip 绑定到 BACKUP 机器上。

```
#!/bin/bash

A=`ps -C nginx --no-header |wc -l`
# 判断nginx是否宕机，如果宕机了，尝试重启
if [ $A -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    # 等待一小会再次检查nginx，如果没有启动成功，则停止keepalived，使其启动备用机
    sleep 3
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi
```

你也可以根据自己的业务需求，总结出在什么情形下关闭keepalived，如 curl 主页连续2个5s没有响应则切换：

```
#!/bin/bash
# curl -IL http://localhost/member/login.htm
# curl --data "memberName=fengkan&password=22" http://localhost/member/login.htm

count = 0
for (( k=0; k<2; k++ )) 
do 
    check_code=$( curl --connect-timeout 3 -sL -w "%{http_code}\\n" http://localhost/login.html -o /dev/null )
    if [ "$check_code" != "200" ]; then
        count = count +1
        continue
    else
        count = 0
        break
    fi
done
if [ "$count" != "0" ]; then
#   /etc/init.d/keepalived stop
    exit 1
else
    exit 0
fi

```

MySQL检查脚本

```
#!/bin/bash
MYSQL_HOST=localhost 
MYSQL_USER=root 
MYSQL_PASSWORD=123456
   
mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASSWORD -e "show status;" >/dev/null 2>&1 
if [ $? == 0 ] 
then 
    echo " $host mysql login successfully " 
    exit 0 
else 
    killall keepalived
    exit 2 
fi
```

