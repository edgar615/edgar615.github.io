---
layout: post
title: linux systemd
date: 2019-10-20
categories:
    - linux
comments: true
permalink: linux-systemdhtml
---

system：系统启动和服务器守护进程管理器，负责在系统启动或运行时，激活系统资源，服务器进程和其他进程，根据管理，字母d是守护进程（daemon）的缩写，systemd这个名字的含义就是它要守护整个系统。

 **Systemd新特性**

- 系统引导时实现服务并行启动
- 按需启动守护进程
- 自动化的服务依赖关系管理
- 同时采用socket式与D-Bus总线式激活服务
- 系统状态快照和恢复
- 利用Linux的cgroups监视进程
- 维护挂载点和自动挂载点
- 各服务间基于依赖关系进行精密控制

CentOS 7的服务systemctl脚本存放在：`/usr/lib/systemd/`，有系统 system 和用户 user 之分， 即：`/usr/lib/systemd/system` 和 `/usr/lib/systemd/user`

# 1. **systemd核心概念unit**

unit表示不同类型的sytemd对象，由其相关的配置文件进行标识、识别和配置，也就是说一个unit到底定义与否，由其配置文件进行标识。这类配置文件中主要包含了几个类别:系统服务，监听的socket、保存的快照以及其他与init相关的信息，这些配置文件中主要保存在：

- `/usr/lib/systemd/system`：每个服务最主要的启动脚本设置，类似于之前的/etc/initd.d

- `/run/system/system`:系统执行过程中所产生的服务脚本，比上面的目录优先运行

- `/etc/system/system`：管理员建立的执行脚本，类似于/etc/rc.d/rcN.d/Sxx类的功能，比上面目录优先运行，在三者之中，此目录优先级最高

如果同一选项三个地方都配置了，优先级高的会覆盖优先级低的。 系统安装时，默认会将unit文件放在`/lib/systemd/system`目录。如果我们想要修改系统默认的配置，比如`nginx.service`，一般有两种方法： 

- 在`/etc/systemd/system`目录下创建`nginx.service`文件，里面写上我们自己的配置。
- 在`/etc/systemd/system`下面创建`nginx.service.d`目录，在这个目录里面新建任何以.conf结尾的文件，然后写入我们自己的配置。推荐这种做法。

`/run/systemd/system`这个目录一般是进程在运行时动态创建unit文件的目录，一般很少修改，除非是修改程序运行时的一些参数时，即Session级别的，才在这里做修改

## 1.1. **unit的常见类型：**

- service unit：这类unit的文件扩展名为.service，主要用于定义系统服务
- target unit：这类unit的文件扩展名为.target，主要用于模拟实现"运行级别"的概念
- device unit：这类unit文件扩展名为.device，用于定义内核识别的设备，然后udev利用systemd识别的硬件，完成创建设备文件名
- mount unit：这类unit文件扩展名为.mount，主要用于定义文件系统挂载点
- socket unit：这类unit文件扩展名为.socket，用于标识进程间通信用到的socket文件
- snapshot unit：这类unit文件扩展名为.snapshot，主要用于实现管理系统快照
- swap unit：这类unit文件扩展名为.swap，主要用于标识管理swap设备
- automount unit：这类unit文件扩展名为.automount，主要用于文件系统自动挂载设备
- path unit：这类unit文件扩展名为.path，主要用于定义文件系统中的文件或目录

## 1.2. **service unit file文件的组成：**

每一个服务以`.service`结尾，一般会分为3部分：[Unit]、[Service]和[Install]

- [Unit]：定义与Unit类型无关的通用选项，用于提供unit的描述信息，unit行为及依赖关系等;

- [Service]：与特定类型相关的专用选项，此处为service类型

- [Install]：定义由“systemctl enable”以及“systemctl disable”命令在实现服务启用或仅用时用到的一些选项;

nginx示例

```
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**unit段的常用选项：**

- Description：描述信息，意义性描述
- After：定义unit启动次序，表示当前unit应该晚于哪些unit启动，其功能与Before相反
- Requires：依赖到的其他units;强依赖，被依赖的unit无法激活时，当前unit也无法激活
- Wants：指明依赖到的其他units;弱依赖，被依赖的unit无法激活时，当前unit可以被激活
- Conflicts：定义units间的冲突关系

**service段的常用选项：**

- Type：用于定义ExecStart及相关参数的功能的unit进程启动类型;
  - simple：默认值，表示由ExecStart启动的进程为主进程，systemd认为该服务将立即启动，服务进程不会fork。如果该服务要启动其他服务，不要使用此类型启动，除非该服务是socket激活型
  - forking：表示由ExecStart启动的进程生成的其中一个子进程将成为主进程，启动完成后，父进程会退出。对于常规的守护进程（daemon），除非你确定此启动方式无法满足需求， 使用此类型启动即可。使用此启动类型应同时指定`PIDFile=`，以便systemd能够跟踪服务的主进程
  - oneshot：功能类似于simple，但是在启动后续的units进程之前，主进程将会退出。这一选项适用于只执行一项任务、随后立即退出的服务。可能需要同时设置 `RemainAfterExit=yes` 使得 `systemd` 在服务进程退出之后仍然认为服务处于激活状态。
  - notify：类似于simple，表示后续的units，仅在通过sdnotify函数发送通知以后，才能运行该命令，这一通知的实现由 `libsystemd-daemon.so` 提供
  - dbus：若以此方式启动，当指定的 BusName 出现在DBus系统总线上时，systemd认为服务就绪。
- EnvironmentFile ：指明环境配置文件，为真正ExecStart执行之前提供环境配置或变量定义等文件
- PIDFile ： pid文件路径
- ExecStart：指明启动unit要运行的命令或脚本;
- ExecStartPre、ExecStartPost表示启动前或启动后要执行的命令或脚本
- ExecStop：指明停止unit要运行的命令或脚本
- Restart：表示进程意外终止了，会自动重启
- PrivateTmp：True表示给服务分配独立的临时空间

**install段的常用选项：**

- Alias：当前unit的别名
- RequiredBy：被那些units所依赖，强依赖
- WantedBy：被那些units所依赖，弱依赖

服务安装的用户模式，从字面上看，就是想要使用这个服务的有是谁？上文中使用的是：`multi-user.target` ，就是指想要使用这个服务的目录是多用户。 每一个.target实际上是链接到我们单位文件的集合,当我们执行   `systemctl enable nginx.service`    就会在 `/etc/systemd/system/multi-user.target.wants/` 目录下新建一个 `/usr/lib/systemd/system/nginx.service` 文件的链接。

注意：对于新创建的unit文件，或修改了的unit文件，必须要让systemd重新识别此配置文件，可利用：·systemctl daemon-reload` 进行重载



# 2. **命令**

## 2.1. 管理服务

命令：`systemctl command name.service`

启动：`service name start -->systemctl start name.service`

停止：`service name stop -->systemctl stop name.service`

重启：`service name restart-->systemctl restart name.service`

状态：`service name status-->systemctl status name.service`

**条件式重启**：已启动才重启，否则不做任何操作

```
systemctl try-restart name.service
```

**重载或重启服务**：先加载，然后再启动

```
systemctl reload-or-try-restart name.service
```

**禁止自动和手动启动**

```
systemctl mask name.service
```

执行此条命令实则创建了一个链接ln -s '/dev/null' '/etc/systemd/system/sshd.service'

 **取消禁止**

```
systemctl unmask name.service
```

删除此前创建的链接 

**设置默认的target**

```
systemctl set-default multi-user.target
```

## 2.2. 服务查看

**查看某服务当前激活与否的状态**

```
systemctl is-active name.service
```

如果启动会显示active，否则会显示unknown

 **查看所有已经激活的服务**

**列出已启动的所有unit**，就是已经被加载到内存中

```
systemctl list-units
```

列出系统已经安装的所有unit，包括那些没有被加载到内存中的unit

```
systemctl list-unit-files
```

查看服务是否开机自启

```
systemctl is-enabled name.servcice
```

查看服务的依赖关系：

```
systemctl list-dependencies
```

查看启动失败的服务

```
systemctl -failed  -t service
```

 查看所有target下的unit

```
systemctl list-unit-files --type=target
```

查看默认target，即默认的运行级别。对应于旧的`runlevel`命令

```
systemctl get-default
```

查看某一target下的unit

```
systemctl list-dependencies multi-user.target
```

查看服务的日志

```
journalctl -u nginx.service 
# 还可以配合`-b`一起使用，只查看自本次系统启动以来的日志
```

# 3. 参考资料 

https://blog.51cto.com/fszxxxks/1855299