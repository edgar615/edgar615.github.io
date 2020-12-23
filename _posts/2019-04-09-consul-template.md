---
layout: post
title: Consul - Template
date: 2019-04-08
categories:
    - Consul
comments: true
permalink: consul-template.html
---

# 1. 简介

Consul-Template是基于Consul的自动替换配置文件的应用。Consul-Template提供了一个便捷的方式从Consul中获取存储的值，Consul-Template守护进程会查询Consul实例来更新系统上指定的任何模板。当更新完成后，模板还可以选择运行一些任意的命令。

**Consul-Template的使用场景**

Consul-Template可以查询Consul中的服务目录、Key、Key-values等。这种强大的抽象功能和查询语言模板可以使Consul-Template特别适合动态的创建配置文件。例如：创建Apache/Nginx Proxy Balancers、Haproxy Backends、Varnish Servers、Application  Configurations等。

**Consul-Template特性**

- Quiescence：Consul-Template内置静止平衡功能，可以智能的发现Consul实例中的更改信息。这个功能可以防止频繁的更新模板而引起系统的波动。
- Dry Mode：不确定当前架构的状态，担心模板的变化会破坏子系统？无须担心。因为Consul-Template还有Dry模式。在Dry模式，Consul-Template会将结果呈现在STDOUT，所以操作员可以检查输出是否正常，以决定更换模板是否安全。
- CLI and Config：Consul-Template同时支持命令行和配置文件。
- Verbose Debugging：即使每件事你都做的近乎完美，但是有时候还是会有失败发生。Consul-Template可以提供更详细的Debug日志信息。

# 2. Get Started

下载：

```
curl -O https://releases.hashicorp.com/consul-template/0.25.1/consul-template_0.25.1_linux_amd64.tgz
tar -zx -f consul-template_0.25.1_linux_amd64.tgz
mv consul-template /usr/local/bin/
```

创建模板in.tpl

```
echo '{{ key "foo" }}' > in.tpl
```

启动

```
consul-template -template "in.tpl:out.txt" -once
```

写入KV

```
consul kv put foo bar
```

查看out.txt的内容

```
cat out.txt 
bar
```

# 3. 常用命令

- 参数

```
-auth=<user[:pass]>      设置基本的认证用户名和密码
-consul-addr=<address>   设置Consul实例的地址
-max-stale=<duration>    查询过期的最大频率，默认是1s
-dedup                   启用重复数据删除，当许多consul template实例渲染一个模板的时候可以降低consul的负载
-ssl                     使用https连接Consul使用SSL
-ssl-verify              通过SSL连接的时候检查证书
-ssl-cert                SSL客户端证书发送给服务器
-ssl-key                 客户端认证时使用的SSL/TLS私钥
-ssl-ca-cert             验证服务器的CA证书列表
-token=<token>           设置Consul API的token
-syslog                  把标准输出和标准错误重定向到syslog，syslog的默认级别是local0。
-syslog-facility=<f>     设置syslog级别，默认是local0，必须和-syslog配合使用
-template=<template>     增加一个需要监控的模板，格式是：'templatePath:outputPath(:command)'，多个模板则可以设置多次
-wait=<duration>         当呈现一个新的模板到系统和触发一个命令的时候，等待的最大最小时间。如果最大值被忽略，默认是最小值的4倍。
-retry=<duration>        当在和consul api交互的返回值是error的时候，重试的等待时间，默认是5s。
-config=<path>           配置文件或者配置目录的路径
-pid-file=<path>         PID文件的路径
-log-level=<level>       设置日志级别，可以是"debug","info", "warn" (default), and "err"
-dry                     Dump生成的模板到标准输出，不会生成到磁盘
-once                    运行consul-template一次后退出，不以守护进程运行
-reap                    子进程自动收割
```

- 渲染模板

```
consul-template \
    -template "/tmp/template.ctmpl:/tmp/result"
```

- 每次模板变化后执行其他的命令

```
$ consul-template \
    -template "/tmp/nginx.ctmpl:/var/nginx/nginx.conf:nginx -s reload" \
    -template "/tmp/redis.ctmpl:/var/redis/redis.conf:service redis restart" \
    -template "/tmp/haproxy.ctmpl:/var/haproxy/haproxy.conf"
```

- 指定consul地址

```
consul-template \
    -consul-addr "10.4.4.6:8500" \
    -vault-addr "https://10.5.32.5:8200"
```

- 写入标准输出，忽略-template中的目的文件

```
$ consul-template -dry -template "in.tpl:out.txt"
> out.txt
baz
> out.txt
bar
```

- Render all templates and then spawn and monitor a child process as a supervisor:

```
$ consul-template \
  -template "/tmp/in.ctmpl:/tmp/result" \
  -exec "/sbin/my-server"
```

# 4. 配置文件

consul-template也支持使用配置文件配置

这里只是简单测试一下，更多内容参考[官方文档](https://github.com/hashicorp/consul-template#configuration-file-format)

```
$ echo 'template {
	source = "/server/data/consul-tpl/in.tpl"
	destination = "/server/data/consul-tpl/out.txt"
	create_dest_dirs = true
}' > /server/config/consul-tpl/tpl.conf

$ consul-template -config /server/config/consul-tpl/tpl.conf
```

# 5. 模版语法

Consul-Template模板文件的语法和Go Template的格式一样，Confd也是遵循Go Template的。

这里主要说下API相关的函数：

- datacenters

在Consul目录中查询所有的数据中心。使用以下语法:

```
{{datacenters}}
```

- file

读取并输出磁盘上的本地文件，如果无法读取产生一个错误。使用如下语法：

```
{{file "/path/to/local/file"}}
```

这个例子将输出`/path/to/local/file`文件内容到模板. 注意:这不会在嵌套模板中被处理。

- key

查询Consul指定key的值，如果key的值不能转换为字符串，将产生错误。使用如下语法:

```
{{key "service/redis/maxconns@east-aws"}}
```

上面的例子查询了在east-aws数据中心的service/redis/maxconns的值。如果忽略数据中心参数，将会查询本地数据中心的值：

```
{{key "service/redis/maxconns"}}
```

- keyOrDefault

查询Consul中指定的key的值，如果key不存在，则返回默认值。使用方式如下：

```
{{keyOrDefault "service/redis/maxconns@east-aws" "5"}}
```

注意Consul Template使用了多个阶段的运算。在第一阶段的运算如果Consul没有返回值，则会一直使用默认值。后续模板解析中如果值存在了则会读取真实的值。Consul Templae不会因为keyOrDefault没找到key而阻塞模板的的渲染。即使key存在如果Consul没有按时返回这个数据，也会使用默认值来进行替代。

- ls

查看Consul的所有以指定前缀开头的key-value对。如果有值无法转换成字符串则会产生一个错误:

```
{{range ls "service/redis@east-aws"}}
{{.Key}} {{.Value}}{{end}}
```

如果Consul实例在east-aws数据中心存在这个结构service/redis，渲染后的模板应该类似这样:

```
minconns 2
maxconns 12
```

如果你忽略数据中心属性,则会返回本地数据中心的查询结果。

- node

查询目录中的一个节点信息。

```
{{node "node1"}}
```

如果未指定任何参数，则当前Agent所在节点将会被返回:

```
{{node}}
```

你可以指定一个可选的参数来指定数据中心:

```
{{node "node1" "@east-aws"}}
```

如果指定的节点没有找到则会返回nil，如果节点存在就会列出节点的信息和节点提供的服务。

```
{{with node}}{{.Node.Node}} ({{.Node.Address}}){{range .Services}}
  {{.Service}} {{.Port}} ({{.Tags | join ","}}){{end}}
{{end}}
```

- nodes

查询Consul目录中的全部节点，使用如下语法：

```
{{nodes}}
```

这个例子会查询Consul的默认数据中心。你可以使用可选参数指定一个可选参数来指定数据中心:

```
{{nodes "@east-aws"}}
```

这个例子会查询east-aws数据中心的所有节点。

- secret

查询Vault中指定路径的密匙，如果指定的路径不存在或者Vault的Token没有足够权限去读取指定的路径，将会产生一个错误。如果路径存在但是key不存在则返回`<no value>`。

```
{{with secret "secret/passwords"}}{{.Data.password}}{{end}}
```

- service

查询Consul中匹配表达式的服务，语法如下:

```
{{service "release.web@east-aws"}}
```

上面的例子查询Consul中，在east-aws数据中心存在的健康的web服务。tag和数据中心参数是可选的，从当前数据中心查询所有节点的web服务而不管tag。查询语法如下:

```
{{service "web"}}
```

这个函数返回`[]*HealthService`结构.可按照如下方式应用到模板:

```
{{range service "web@data center"}}
server {{.Name}} {{.Address}}:{{.Port}}{{end}}

```

产生类似如下输出:

```
server nyc_web_01 1.2.3.4:8080
server nyc_web_02 10.20.30.40:8080
```

默认值会返回健康的服务。如果你想获取所有服务，可以增加any选项。如下：

```
{{service "web" "any"}}
```

这样就会返回注册过的所有服务，而不论它的状态如何。

如果你想过滤指定的一个或者多个健康状态，你可以通过逗号隔开多个健康状态：

```
{{service "web" "passing, warning"}}
```

这样将会返回被它们的节点和服务级别的检查定义标记为”passing”或者”warning”的服务。请注意逗号是OR而不是AND的意思。指定了超过一个状态过滤，并包含any将会返回一个错误。因为any是比所有状态更高级的过滤。

后面这2种方式有些架构上的不同:

```
{{service "web"}}
{{service "web" "passing"}}
```

前者会返回Consul认为healthy和passing的所有服务。后者将返回所有已经在Consul注册的服务，然后会执行一个客户端的过滤。通常如果你想获取健康的服务，你应该不要使用passing参数。直接忽略第三个参数即可。然而第三个参数在你想查询passing或者warning的服务会比较有用,如下:

```
{{service "web" "passing, warning"}}
```

服务的状态也是可见的，如果你想自己做一些额外的过滤。语法如下：

```
{{range service "web" "any"}}
{{if eq .Status "critical"}}
// Critical state!{{end}}
{{if eq .Status "passing"}}
// Ok{{end}}
```

执行命令时，在Consul将服务设置为维护模式只需要在你的命令上包上Consul的maint调用:

```
#!/bin/sh
set -e
consul maint -enable -service web -reason "Consul Template updated"
service nginx reload
consul maint -disable -service web
```

另外如果你没有安装Consul agent,你可以直接调用API请求:

```
#!/bin/sh
set -e
curl -X PUT "http://$CONSUL_HTTP_ADDR/v1/agent/service/maintenance/web?enable=true&reason=Consul+Template+Updated"
service nginx reload
curl -X PUT "http://$CONSUL_HTTP_ADDR/v1/agent/service/maintenance/web?enable=false"
```

- services

查询Consul目录中的所有服务，使用如下语法:

```
{{services}}
```

这个例子将查询Consul的默认数据中心，你可以指定一个可选参数来指定数据中心:

```
{{services "@east-aws"}}
```

请注意: services函数与service是不同的，service接受更多参数并且查询监控的服务列表。这个查询Consul目录并返回一个服务的tag的Map。如下:

```
{{range services}}
{{.Name}}
{{range .Tags}}
  {{.}}{{end}}
{{end}}
```

- tree

查询所有指定前缀的key-value值对，如果其中的值有无法转换为字符串的则引发错误:

```
{{range tree "service/redis@east-aws"}}
{{.Key}} {{.Value}}{{end}}
```

如果Consul实例在east-aws数据中心有一个service/redis结构，模板的渲染结果类似下面:

```
minconns 2
maxconns 12
nested/config/value "value"
```

和ls不同tree返回前缀下的所有key，和Unix的tree命令比较像。如果忽略数据中心参数，则会使用本地数据中心。

更多辅助函数，比如 scratch、byKey、byTag、contains、explode、in、loop、trimSpace、join、parseBool、parseFloat、parseInt、parseJSON、parseUint、regexMatch等可参考：官方文档

# 6. 生成nginx

## 6.1. 命令行方式

```
$ vim nginx.conf.tpl

{{range services}} {{$name := .Name}} {{$service := service .Name}}
upstream {{$name}} {
  zone upstream-{{$name}} 64k;
  {{range $service}}server {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;
  {{end}}
} {{end}}

server {
  listen 80 default_server;

  location / {
    root /usr/share/nginx/html/;
    index index.html;
  }

  location /stub_status {
    stub_status;
  }

{{range services}} {{$name := .Name}}
  location /{{$name}} {
	rewrite /{{$name}}/(.*) /$1 break;
    proxy_pass http://{{$name}};
  }
{{end}}
}
```

```
consul-template -template "nginx.conf.tpl:/workspace/nginx/nginx.conf:docker restart nginx" -once

consul-template -template "nginx.conf.tpl:/workspace/nginx/nginx.conf:docker exec -it nginx nginx -s reload" -once
```

第二种方式一直没试验成功，容器内的nginx.conf一直没变

## 6.2. 配置文件方式

```
$ vim nginx.hcl

consul {
address = "192.168.2.210:8500"
}

template {
source = "nginx.conf.ctmpl"
destination = "/usr/local/nginx/conf/conf.d/default.conf"
command = "service nginx reload"
}
```

执行以下命令就可渲染文件了：

```
$ consul-template -config "nginx.hcl"
```

官方示例参考

https://github.com/hashicorp/consul-template/tree/master/examples

# 7. 参考资料

https://github.com/hashicorp/consul-template

https://www.hi-linux.com/posts/36431.html

https://learn.hashicorp.com/tutorials/consul/consul-template