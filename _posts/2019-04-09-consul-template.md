---
layout: post
title: Consul - Template
date: 2019-04-07
categories:
    - Consul
comments: true
permalink: consul-template.html
---

https://github.com/hashicorp/consul-template

https://www.hi-linux.com/posts/36431.html

下载：

```
curl -O https://releases.hashicorp.com/consul-template/0.19.4/consul-template_0.19.4_linux_amd64.tgz
tar -zx -f consul-template_0.19.4_linux_amd64.tgz
```

启动

```
consul-template -template "in.tpl:out.txt" -once
//in.tpl的内容为{{ key "foo" }}
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



渲染模板

```
consul-template \
    -template "/tmp/template.ctmpl:/tmp/result"
```

每次模板变化后执行其他的命令

```
$ consul-template \
    -template "/tmp/nginx.ctmpl:/var/nginx/nginx.conf:nginx -s reload" \
    -template "/tmp/redis.ctmpl:/var/redis/redis.conf:service redis restart" \
    -template "/tmp/haproxy.ctmpl:/var/haproxy/haproxy.conf"
```

指定consul地址

```
consul-template \
    -consul-addr "10.4.4.6:8500" \
    -vault-addr "https://10.5.32.5:8200"
```

Render all templates and then spawn and monitor a child process as a supervisor:

```
$ consul-template \
  -template "/tmp/in.ctmpl:/tmp/result" \
  -exec "/sbin/my-server"
```

配置文件：

参考官方文档

### Consul-Template模版语法

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

### 生成nginx

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

./consul-template -template "nginx.conf.tpl:/workspace/nginx/nginx.conf:docker restart nginx" -once

./consul-template -template "nginx.conf.tpl:/workspace/nginx/nginx.conf:docker exec -it nginx nginx -s reload" -once

第二种方式一直没试验成功，容器内的nginx.conf一直没变



### 使用配置文件的方式

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