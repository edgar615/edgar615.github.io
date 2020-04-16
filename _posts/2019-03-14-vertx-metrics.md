---
layout: post
title: Vert.x性能度量
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-metrics.html
---

# vertx-dropwizard-metrics

Vert.x提供了metric组件，用于收集应用的各种指标

# 快速入门

创建Vert.x的时候开启metric

    Vertx.vertx(new VertxOptions()
                        .setMetricsOptions(new MetricsOptions().setEnabled(true)))

MetricsOptions可以接受一个JSON对象作为参数，用于扩展MetricsOptions未提供的属性但是DropwizardMetricsOptions却提供的属性，例如registryName、jmxEnabled等

也可以在启动时通过命令行参数控制

	java -jar your-fat-jar -Dvertx.metrics.options.enabled=true


`vertx.metrics.options.registryName`可以配置Dropwizard Registry

java -jar your-fat-jar -Dvertx.metrics.options.enabled=true -Dvertx.metrics.options.registryName=my-registry

`vertx.metrics.options.jmxEnabled` 和 `vertx.metrics.options.jmxDomain` 可以配置JMX

	java -ja	r your-fat-jar -Dvertx.metrics.options.enabled=true -Dvertx.metrics.options.jmxEnabled=true

也可以直接通过`vertx.metrics.options.configPath`属性来制定metrics的配置文件

	{
	  "enabled" : true,
	  "registerName" : "my-register",
	  "monitoredHttpServerUris" : [
	    {
	      "value" : "/login",
	      "type" : "EQUALS"
	    }
	  ]
	}

	java -jar your-fat-jar -Dvertx.metrics.options.enabled=true -Dvertx.metrics.options.configPath=\conf\metrics.json

**使用`-Dvertx.metrics.options.configPath`的时候，一定要同时使用-Dvertx.metrics.options.enabled=true开启metrics，不然configPath不会有任何作用**

**创建MetricsService**

	MetricsService service = MetricsService.create(vertx);

MetricsService定义了指标收集的一些接口

**获取指标**

      JsonObject metrics = service.getMetricsSnapshot(vertx.eventBus());
      System.out.println(metrics);

getMetricsSnapshot可以通过每个度量指标的名称搜索。`service.getMetricsSnapshot("vertx.eventbus.handlers");`也可以通过实现了`Measured`接口的组件来获取

也可以通过metricsNames方法获取所有的指标名称

      Set<String> metricsNames = service.metricsNames();
      for (String metricsName : metricsNames) {
        System.out.println("Known metrics name " + metricsName);
      }

## 获取Dropwizard Registry
如果指定了registryName,就可以通过下面的方法获取到registry

	MetricRegistry registry = SharedMetricRegistries.getOrCreate("my-registry");

# 实现
Dropwizard/metrics提供了下列几种指标
- Gauges 最简单的度量指标，只有一个简单的返回值，Gauge同时也有几个扩展Ratio Gauges、Cached Gauges、Derivative Gauges、JMX Gauges
- Counters 计数器，Counter 只是用 Gauge 封装了 AtomicLong 。
- Meters 度量一系列事件发生的速率(rate)，例如TPS。Meters会统计最近1分钟，5分钟，15分钟，还有全部时间的速率。
- Histograms 统计数据的分布情况。比如最小值，最大值，中间值，还有中位数，75百分位, 90百分位, 95百分位, 98百分位, 99百分位, 和 99.9百分位的值(percentiles)。
- Timers  Histogram 和 Meter 的结合， histogram 某部分代码/调用的耗时， meter统计TPS。

Vert.x在这个基础上实现了ThroughputMeter和Throughput Timer两种类型

Vert.x会将这些度量指标格式化为JSON格式：
AbstractMetrics的metrics方法会从MetricRegistry中读取指定的度量指标，然后转换为JSON。JSON对象的键为度量指标名，值会根据不同的度量指标类型而不同

	public JsonObject metrics(String baseName) {
		Map<String, Object> map = registry.getMetrics().
			entrySet().
			stream().
			filter(e -> e.getKey().startsWith(baseName)).
			collect(Collectors.toMap(
				e -> projectName(e.getKey()),
				e -> Helper.convertMetric(e.getValue(), TimeUnit.SECONDS, TimeUnit.MILLISECONDS)));
		return new JsonObject(map);
	}

convertMetric会根据度量指标的类型来创建对应的度量指标值

	public static JsonObject convertMetric(Metric metric, TimeUnit rateUnit, TimeUnit durationUnit) {
		if (metric instanceof Timer) {
		  return toJson((Timer) metric, rateUnit, durationUnit);
		} else if (metric instanceof Gauge) {
		  return toJson((Gauge) metric);
		} else if (metric instanceof Counter) {
		  return toJson((Counter) metric);
		} else if (metric instanceof Histogram) {
		  return toJson((Histogram) metric);
		} else if (metric instanceof Meter) {
		  return toJson((Meter) metric, rateUnit);
		} else {
		  throw new IllegalArgumentException("Unknown metric " + metric);
		}
	}

**Gauge**

	{
	  "type"  : "gauge",
	  "value" : value // any json value
	}

**Counter**

	{
	  "type"  : "counter",
	  "count" : 1 // number
	}

**Histogram**

	{
	  "type"   : "histogram",
	  "count"  : 1 // long 数量
	  "min"    : 1 // long 最小值
	  "max"    : 1 // long 最大值
	  "mean"   : 1.0 // double 平均值
	  "stddev" : 1.0 // double 标准偏差
	  "median" : 1.0 // double 中位数
	  "75%"    : 1.0 // double 75百分位的值
	  "95%"    : 1.0 // double 95百分位的值
	  "98%"    : 1.0 // double 98百分位的值
	  "99%"    : 1.0 // double 99百分位的值
	  "99.9%"  : 1.0 // double 99.9百分位的值
	}

**Meter**

	{
	  "type"              : "meter", 
	  "count"             : 1 // long 数量
	  "meanRate"          : 1.0 // double 平均值
	  "oneMinuteRate"     : 1.0 // double 一分钟的值
	  "fiveMinuteRate"    : 1.0 // double 五分钟的值
	  "fifteenMinuteRate" : 1.0 // double 十分钟的值
	  "rate"              : "events/second" // string representing rate 速率的单位
	}

**ThroughputMeter**

Meter的子类，扩展了吞吐量的值

	{
	  "type"              : "meter",
	  "count"             : 40 // long
	  "meanRate"          : 2.0 // double
	  "oneSecondRate"     : 3 // long - number of occurence for the last second最近一秒钟的值
	  "oneMinuteRate"     : 1.0 // double
	  "fiveMinuteRate"    : 1.0 // double
	  "fifteenMinuteRate" : 1.0 // double
	  "rate"              : "events/second" // string representing rate
	}

**Timer**
	
	{
	  "type": "timer",
	
	  // histogram data
	  "count"  : 1 // long
	  "min"    : 1 // long
	  "max"    : 1 // long
	  "mean"   : 1.0 // double
	  "stddev" : 1.0 // double
	  "median" : 1.0 // double
	  "75%"    : 1.0 // double
	  "95%"    : 1.0 // double
	  "98%"    : 1.0 // double
	  "99%"    : 1.0 // double
	  "99.9%"  : 1.0 // double
	
	  // meter data
	  "meanRate"          : 1.0 // double
	  "oneMinuteRate"     : 1.0 // double
	  "fiveMinuteRate"    : 1.0 // double
	  "fifteenMinuteRate" : 1.0 // double
	  "rate"              : "events/second" // string representing rate
	}

**ThroughputTimer**

Timer的子类，扩展了吞吐量的值

	{
	  "type": "timer",
	
	  // histogram data
	  "count"      : 1 // long
	  "min"        : 1 // long
	  "max"        : 1 // long
	  "mean"       : 1.0 // double
	  "stddev"     : 1.0 // double
	  "median"     : 1.0 // double
	  "75%"        : 1.0 // double
	  "95%"        : 1.0 // double
	  "98%"        : 1.0 // double
	  "99%"        : 1.0 // double
	  "99.9%"      : 1.0 // double
	
	  // meter data
	  "meanRate"          : 1.0 // double
	  "oneSecondRate"     : 3 // long - number of occurence for the last second
	  "oneMinuteRate"     : 1.0 // double
	  "fiveMinuteRate"    : 1.0 // double
	  "fifteenMinuteRate" : 1.0 // double
	  "rate"              : "events/second" // string representing rate
	}

ThroughputMeter和ThroughputTimer都是在内部通过一个InstantThroughput对象来实现oneSecondRate度量。

ThroughputMeter重写了Meter的mark方法，再调用Meter的mark的同时，也调用了InstantThroughput的mark方法

	public class ThroughputMeter extends Meter {
	
	  private final InstantThroughput instantThroughput = new InstantThroughput();
	
	  public Long getValue() {
	    return instantThroughput.count();
	  }
	
	  @Override
	  public void mark() {
	    super.mark();
	    instantThroughput.mark();
	  }
	}

ThroughputTimer与ThroughputMeter类似，重写了Timer的update方法

	@Override
	public void update(long duration, TimeUnit unit) {
		super.update(duration, unit);
		instantThroughput.mark();
	}

InstantThroughput定义了3个volatile变量

	  private volatile long prevCount = -1;
	  private volatile long timestamp = 0;
	  private volatile long count = 0;

timestamp记录最新的时间（纳秒级），prevCount记录上一秒的数量，**显示的度量值**，count记录当前秒的数量

在执行mark方式时，首先会执行check方法，检查当前时间与上次的时间是否是同一秒，如果当前时间与上次的时间是同一秒，直接将count加1。如果不是同一秒，需要将count重置为0，timestamp修改为当前时间。如果已经超过2秒，还需要将上一秒的数量prevCount置为0；如果尚未超过2秒，将上一秒的数量重置为count

	  public void mark() {
	//    System.out.println("mark");
	    check();
	    count++;
	  }

	  void check() {
	    long now = System.nanoTime();
	    if (now - timestamp > ONE_SEC) {
	      if (now - timestamp < TWO_SECS) {
	        prevCount = count;
	      } else {
	        if (prevCount > 0) {
	          prevCount = 0;
	        }
	      }
	      timestamp = now;
	      count = 0;
	    }
	  }

oneSecondRate的度量值使用的上一秒的数量（因为当前时间可能还有请求没有过来，使用count的话度量不准确）。**在系统刚刚启动，还没有记录prevCount的时候使用的是count**

	  public long count() {
	//    System.out.println("count");
	    check();
	    return prevCount != -1 ? prevCount : count;
	  }

## MetricsService
MetricsService从Measured中收集度量指标，然后生成快照给调用方

	  @Override
	  public String getBaseName(Measured measured) {
	    AbstractMetrics codahaleMetrics = AbstractMetrics.unwrap(measured);
	    return codahaleMetrics != null ? codahaleMetrics.baseName() : null;
	  }
	
	  @Override
	  public JsonObject getMetricsSnapshot(Measured measured) {
	    AbstractMetrics codahaleMetrics = AbstractMetrics.unwrap(measured);
	    return codahaleMetrics != null ? codahaleMetrics.metrics() : null;
	  }

## AbstractMetrics
AbstractMetrics封装了度量的一些基本方法，例如：创建度量指标，提取JSON格式的度量指标

## MetricsProvider
度量指标的接口，如果需要收集某个组件的度量指标，该组件需要实现这个接口

  Metrics getMetrics();//获取具体的度量对象

## Metrics
收集度量指标的接口，每个组件需要收集度量指标都会有一些差异，所以针对不同的组件需要实现不同的Metrics对象，比如PoolMetrics、EventBusMetrics等等

## VertxMetrics
VertxMetrics定义了一些基本的metric接口：verticle的部署和卸载，EventBus、HttpServer等组件的metric对象的创建。

Vertx对象在创建时，会创建一个对应的VertxMetrics对象

	  private VertxMetrics initialiseMetrics(VertxOptions options) {
	    if (options.getMetricsOptions() != null && options.getMetricsOptions().isEnabled()) {
	      VertxMetricsFactory factory = options.getMetricsOptions().getFactory();
	      if (factory == null) {
	        factory = ServiceHelper.loadFactoryOrNull(VertxMetricsFactory.class);
	        if (factory == null) {
	          log.warn("Metrics has been set to enabled but no VertxMetricsFactory found on classpath");
	        }
	      }
	      if (factory != null) {
	        VertxMetrics metrics = factory.metrics(this, options);
	        Objects.requireNonNull(metrics, "The metric instance created from " + factory + " cannot be null");
	        return metrics;
	      }
	    }
	    return DummyVertxMetrics.INSTANCE;
	  }

如果在classpath中没有找到对应Metric的工厂实现，会使用core包中的DummyVertxMetrics。DummyVertxMetrics不会收集任何度量指标，它所有的方法实现都是空

	@Override
	public void verticleDeployed(Verticle verticle) {
	}
	
	@Override
	public void verticleUndeployed(Verticle verticle) {
	}

## VertxMetricsFactoryImpl 
下面我们看一下vertx-dropwizard-metrics提供的的实现

metrics方法首先会创建一个 MetricRegistry用于存放所有的指标数据。

	MetricRegistry registry = new MetricRegistry();
	boolean shutdown = true;
	if (metricsOptions.getRegistryName() != null) {
	  MetricRegistry other = SharedMetricRegistries.add(metricsOptions.getRegistryName(), registry);
	  if (other != null) {
	    registry = other;
	    shutdown = false;
	  }
	}

SharedMetricRegistries内部使用了一个MAP来存放registry对象，这样可以通过registryName获得对应的registry对象
**必须使用`-Dvertx.metrics.options.registryName=my-registry`指明register的名称，configPath中的registerName不会有任何作用**

接着会创建一个VertxMetrics对象

	VertxMetricsImpl metrics = new VertxMetricsImpl(registry, shutdown, options, metricsOptions);

如果开启了JMX支持，还会创建一个JmxReporter，并将它的关闭注册为VertxMetrics关闭的回调函数。然后启动 JmxReporter;

    if (metricsOptions.isJmxEnabled()) {
      String jmxDomain = metricsOptions.getJmxDomain();
      if (jmxDomain == null) {
        jmxDomain = "vertx" + "@" + Integer.toHexString(vertx.hashCode());
      }
      JmxReporter reporter = JmxReporter.forRegistry(metrics.registry()).inDomain(jmxDomain).build();
      metrics.setDoneHandler(v -> reporter.stop());
      reporter.start();
    }

## VertxMetricsImpl
VertxMetricsImpl的构造方法首先会创建两个Counter类型的度量指标

    this.timers = counter("timers");
    this.verticles = counter("verticles");

- vertx.timers 定时器的数量
- vertx.verticles 当前部署的verticle数量
- vertx.verticles.<verticle-name> 某个verticle当前部署的数量

创建几个Guage类型的度量指标

    gauge(options::getEventLoopPoolSize, "event-loop-size");
    gauge(options::getWorkerPoolSize, "worker-pool-size");
    if (options.isClustered()) {
      gauge(options::getClusterHost, "cluster-host");
      gauge(options::getClusterPort, "cluster-port");
    }

- vertx.event-loop-size eventloop的大小
- vertx.worker-pool-size 工作线程池的大小

如果Vert.x处于集群环境，还会创建

- vertx.cluster-host 集群的HOST
- vertx.cluster-port 集群的端口

在部署和卸载Verticle时，会调用VertxMetrics对应的方法更新vertx.verticles

	  @Override
	  public void verticleDeployed(Verticle verticle) {
	    verticles.inc();
	    counter("verticles", verticleName(verticle)).inc();
	  }
	
	  @Override
	  public void verticleUndeployed(Verticle verticle) {
	    verticles.dec();
	    counter("verticles", verticleName(verticle)).dec();
	  }
部署

	vertx.metricsSPI().verticleDeployed(verticle);

卸载

	vertx.metricsSPI().verticleUndeployed(verticleHolder.verticle);

在创建和取消定时/延时任务时，会更新vertx.timers

	@Override
	public void timerCreated(long id) {
	timers.inc();
	}
	
	@Override
	public void timerEnded(long id, boolean cancelled) {
	timers.dec();
	}

创建

	metrics.timerCreated(timerID);

取消

	metrics.timerEnded(timerID, true);

如果是延时任务，在执行完成之后会将timer减1

    private void cleanupNonPeriodic() {
      VertxImpl.this.timeouts.remove(timerID);
      metrics.timerEnded(timerID, false);
      ContextImpl context = getContext();
      if (context != null) {
        context.removeCloseHook(this);
      }
    }

## EventBusMetrics
EventBusMetricsImpl的构造方法会创建一系列EventBus相关的度量指标，这些度量指标的基础名称都是vertx.eventbus

	  @Override
	  public EventBusMetrics createMetrics(EventBus eventBus) {
	    return new EventBusMetricsImpl(this, nameOf("eventbus"), options);
	  }

创建的度量指标

    handlerCount = counter("handlers");
    pending = counter("messages", "pending");
    pendingLocal = counter("messages", "pending-local");
    pendingRemote = counter("messages", "pending-remote");
    receivedMessages = throughputMeter("messages", "received");
    receivedLocalMessages = throughputMeter("messages", "received-local");
    receivedRemoteMessages = throughputMeter("messages", "received-remote");
    deliveredMessages = throughputMeter("messages", "delivered");
    deliveredLocalMessages = throughputMeter("messages", "delivered-local");
    deliveredRemoteMessages = throughputMeter("messages", "delivered-remote");
    sentMessages = throughputMeter("messages", "sent");
    sentLocalMessages = throughputMeter("messages", "sent-local");
    sentRemoteMessages = throughputMeter("messages", "sent-remote");
    publishedMessages = throughputMeter("messages", "published");
    publishedLocalMessages = throughputMeter("messages", "published-local");
    publishedRemoteMessages = throughputMeter("messages", "published-remote");
    replyFailures = meter("messages", "reply-failures");
    bytesRead = meter("messages", "bytes-read");
    bytesWritten = meter("messages", "bytes-written");
    

Couter类型

- handlers eventbus的handler数量
- messages.pending 已经收到，但还没有被handler处理的消息数量
- messages.pending-local 已经收到，但还没有被handler处理的本地消息数
- messages.pending-remote 已经收到，但还没有被handler处理的远程消息数

ThroughputMeter类型

- messages.sent 发送的消息
- messages.sent-local 发送的本地消息
- messages.sent-remote 发送的远程消息
- messages.published 发布的消息
- messages.published-local 发布的本地消息
- messages.published-remote 发布的远程消息
- messages.received 收到的消息
- messages.received-local 收到的本地消息
- messages.received-remote 收到的远程消息
- messages.delivered - A [throughpu_metert] representing the rate of which messages are being delivered to an handler
- messages.delivered-local - A ThroughputMeter representing the rate of which local messages are being delivered to an handler
- messages.delivered-remote - A ThroughputMeter representing the rate of which remote messages are being delivered to an handler
- messages.bytes-read 读取的远程消息字节
- messages.bytes-written 发送的远程消息字节
- messages.reply-failures 回应失败

EventBus在发送消息的时候回调用metric的方法记录度量指标

	  protected <T> void sendOrPub(SendContextImpl<T> sendContext) {
	    MessageImpl message = sendContext.message;
	    metrics.messageSent(message.address(), !message.send(), true, false);
	    deliverMessageLocally(sendContext);
	  }

	  @Override
	  public void messageSent(String address, boolean publish, boolean local, boolean remote) {
	    if (publish) {
	      publishedMessages.mark();
	      if (local) {
	        publishedLocalMessages.mark();
	      } else {
	        publishedRemoteMessages.mark();
	      }
	    } else {
	      sentMessages.mark();
	      if (local) {
	        sentLocalMessages.mark();
	      } else {
	        sentRemoteMessages.mark();
	      }
	    }
	  }


## TCPMetricsImpl
在TCP Server的listen方法连接成功之后，会创建TCPMetricsImpl对象。

	metrics = vertx.metricsSPI().createMetrics(this, new SocketAddressImpl(id.port, id.host), options);

它创建的度量指标的基础名字都是 vertx.net.servers

	  @Override
	  public TCPMetrics<?> createMetrics(NetServer server, SocketAddress localAddress, NetServerOptions options) {
	    String baseName = MetricRegistry.name(nameOf("net.servers"), TCPMetricsImpl.addressName(localAddress));
	    return new TCPMetricsImpl(registry, baseName);
	  }

创建TCP Client时也会创建TCPMetricsImpl对象。

	public NetClientImpl(VertxInternal vertx, NetClientOptions options, boolean useCreatingContext) {
		//etc
		this.metrics = vertx.metricsSPI().createMetrics(this, options);
	}

它创建的度量指标的基础名字都是 vertx.net.clients，如果通过NetClientOptions指定了metricsName，基础名字为vertx.net.clients.<metricsName>

  @Override
  public TCPMetrics<?> createMetrics(NetClient client, NetClientOptions options) {
    String baseName;
    if (options.getMetricsName() != null) {
      baseName = nameOf("net.clients", options.getMetricsName());
    } else {
     baseName = nameOf("net.clients");
    }
    return new TCPMetricsImpl(registry, baseName);
  }

TCPMetricsImpl的构造方法会创建下列度量指标

    this.openConnections = counter("open-netsockets");
    this.connections = timer("connections");
    this.exceptions = counter("exceptions");
    this.bytesRead = histogram("bytes-read");
    this.bytesWritten = histogram("bytes-written");

Counter类型

- open-netsockets socket连接数
- open-netsockets.<remote-host> 某个远程主机的socket连接数

TCP Server在连接建立或关闭之后，会更新这这个度量指标

	private void connected(Channel ch, HandlerHolder<Handler<NetSocket>> handler) {
	  // Need to set context before constructor is called as writehandler registration needs this
	  // etc ...
	  handler.context.executeFromIO(() -> {
		sock.setMetric(metrics.connected(sock.remoteAddress(), sock.remoteName()));
		handler.handler.handle(sock);
	  });
	}

	protected synchronized void handleClosed() {
		if (metrics instanceof TCPMetrics) {
		  ((TCPMetrics) metrics).disconnected(metric(), remoteAddress());
		}
		if (closeHandler != null) {
		  closeHandler.handle(null);
		}
	}


TCPMetricsImpl在连接建立时将open-netsockets加1

  @Override
  public Long connected(SocketAddress remoteAddress, String remoteName) {
    // Connection metrics
    openConnections.inc();

    // On network outage the remoteAddress can be null.
    // Do not report the open-connections when it's null
    if (remoteAddress != null) {
      // Remote address connection metrics
      counter("open-connections", remoteAddress.host()).inc();

    }

    // A little clunky, but it's possible we got here after closed has been called
    if (closed) {
      removeAll();
    }

    return System.nanoTime();
  }

在连接关闭之后会将计数器减1

	@Override
	public void disconnected(Long ctx, SocketAddress remoteAddress) {
		openConnections.dec();
		connections.update(System.nanoTime() - ctx, TimeUnit.NANOSECONDS);
	
		// On network outage the remoteAddress can be null.
		// Do not report the open-connections when it's null
		if (remoteAddress != null) {
		  // Remote address connection metrics
		  Counter counter = counter("open-connections", remoteAddress.host());
		  counter.dec();
		  if (counter.getCount() == 0) {
			remove("open-connections", remoteAddress.host());
		  }
		}
	
		// A little clunky, but it's possible we got here after closed has been called
		if (closed) {
		  removeAll();
		}
	}


- exceptions 异常的数量

在收到异常时将该度量指标加1

	protected synchronized void handleException(Throwable t) {
		metrics.exceptionOccurred(metric(), remoteAddress(), t);
		if (exceptionHandler != null) {
		  exceptionHandler.handle(t);
		} else {
		  log.error(t);
		}
	}
	
	public void exceptionOccurred(Long socketMetric, SocketAddress remoteAddress, Throwable t) {
		exceptions.inc();
	}

Timer类型

- connections - A Timer of a connection and the rate of it’s occurrence

在连接建立时生产一个纳秒级时间，在连接断开时，计算时间差，更新该度量指标

	connections.update(System.nanoTime() - ctx, TimeUnit.NANOSECONDS);


Histogram类型

- bytes-read 读取的字节数
- bytes-written 写入的字节数

在收到和写入数据时会更新这两个值

	synchronized void handleDataReceived(Buffer data) {
		// etc ...
		reportBytesRead(data.length());
		if (dataHandler != null) {
		  dataHandler.handle(data);
		}
	}
	
	private void write(ByteBuf buff) {
		reportBytesWritten(buff.readableBytes());
		writeFuture = super.writeToChannel(buff);
	}

	public void reportBytesRead(long numberOfBytes) {
		if (metrics.isEnabled()) {
		  metrics.bytesRead(metric(), remoteAddress(), numberOfBytes);
		}
	}
	
	public void reportBytesWritten(long numberOfBytes) {
		if (metrics.isEnabled()) {
		  metrics.bytesWritten(metric(), remoteAddress(), numberOfBytes);
		}
	}

	@Override
	public void bytesRead(Long socketMetric, SocketAddress remoteAddress, long numberOfBytes) {
		bytesRead.update(numberOfBytes);
	}
	
	@Override
	public void bytesWritten(Long socketMetric, SocketAddress remoteAddress, long numberOfBytes) {
		bytesWritten.update(numberOfBytes);
	}

## HttpServerMetricsImpl
在HTTP Server的listen方法连接成功之后，会创建HttpServerMetricsImpl对象。

HttpServerMetricsImpl的构造方法会创建一系列HTTPSERVER相关的度量指标，这些度量指标的基础名称都是vertx.http.servers

	  @Override
	  public HttpServerMetrics<?, ?, ?> createMetrics(HttpServer server, SocketAddress localAddress, HttpServerOptions options) {
	    return new HttpServerMetricsImpl(registry, nameOf("http.servers"), this.options.getMonitoredHttpServerUris(), localAddress);
	  }

HttpServerMetricsImpl继承自HttpMetricsImpl。HttpMetricsImpl继承自TCPMetricsImpl。它除了TCP的度量指标之外还实现了下列度量指标

HttpMetricsImpl

    openWebSockets = counter("open-websockets");
    requests = throughputTimer("requests");
    responses = new ThroughputMeter[]{
        throughputMeter("responses-1xx"),
        throughputMeter("responses-2xx"),
        throughputMeter("responses-3xx"),
        throughputMeter("responses-4xx"),
        throughputMeter("responses-5xx")
    };
    methodRequests = new EnumMap<>(HttpMethod.class);
    for (HttpMethod method : HttpMethod.values()) {
      methodRequests.put(method, throughputTimer(method.toString().toLowerCase() + "-requests"));
    }

Counter类型

- open-websockets websocket连接数
- open-websockets.<remote-host> 某个远程服务器的websocket连接数

ThroughputMeter

- requests 请求的Throughput Timer
- <http-method>-requests 某个特定方法的请求Throughput Timer，例如: get-requests, post-requests
- <http-method>-requests./<uri> - 某个特定方法和URI的请求Throughput Timer，例如: get-requests./some/uri, post-requests./some/uri?foo=bar
- responses-1xx 响应码是1xx的Throughput Timer
- responses-2xx 响应码是2xx的Throughput Timer
- responses-3xx 响应码是3xx的Throughput Timer
- responses-4xx 响应码是4xx的Throughput Timer
- responses-5xx 响应码是5xx的Throughput Timer

**<http-method>-requests./<uri> 需要使用addMonitoredHttpServerUri方法设置匹配规则才起作用**

	Vertx vertx = Vertx.vertx(new VertxOptions().setMetricsOptions(
	    new DropwizardMetricsOptions().
	        setEnabled(true).
	        addMonitoredHttpServerUri(
	            new Match().setValue("/")).
	        addMonitoredHttpServerUri(
	            new Match().setValue("/foo/.*").setType(MatchType.REGEX))
	));

在请求处理完成之后responseComplete方法会更新度量指标值

	 synchronized void responseComplete() {
	    if (metrics.isEnabled()) {
	      reportBytesWritten(bytesWritten);
	      bytesWritten = 0;
	      if (requestFailed) {
	        metrics.requestReset(requestMetric);
	        requestFailed = false;
	      } else {
	        metrics.responseEnd(requestMetric, pendingResponse);
	      }
	    }
	    pendingResponse = null;
	    checkNextTick();
	  }

	protected long end(RequestMetric metric, int statusCode, boolean monitorUri) {
	    if (closed) {
	      return 0;
	    }
	
	    long duration = System.nanoTime() - metric.requestBegin;
	    int responseStatus = statusCode / 100;
	
	    //
	    if (responseStatus >= 1 && responseStatus <= 5) {
	      responses[responseStatus - 1].mark();
	    }
	
	    // Update generic requests metric
	    requests.update(duration, TimeUnit.NANOSECONDS);
	
	    // Update specific method / uri request metrics
	    if (metric.method != null) {
	      methodRequests.get(metric.method).update(duration, TimeUnit.NANOSECONDS);
	      if (metric.uri != null && monitorUri) {
	        throughputTimer(metric.method.toString().toLowerCase() + "-requests", metric.uri).update(duration, TimeUnit.NANOSECONDS);
	      }
	    } else if (metric.uri != null && monitorUri) {
	      throughputTimer("requests", metric.uri).update(duration, TimeUnit.NANOSECONDS);
	    }
	
	    return duration;
	  }

## HttpClientMetricsImpl
在创建每个HttpClient对象的时候，会创建对应的HttpClientMetricsImpl

HttpClientMetricsImpl除了HttpServerMetricsImpl的度量指标外，还实现了下列度量指标

	  public HttpClientReporter(MetricRegistry registry, String baseName, SocketAddress localAdress) {
	    super(registry, baseName, localAdress);
	
	    // max pool size gauge
	    gauge(() -> totalMaxPoolSize, "connections", "max-pool-size");
	
	    // connection pool ratio
	    RatioGauge gauge = new RatioGauge() {
	      @Override
	      protected Ratio getRatio() {
	        return Ratio.of(connections(), totalMaxPoolSize);
	      }
	    };
	    gauge(gauge, "connections", "pool-ratio");
	  }


它创建的度量指标的基础名字都是 vertx.http.clients，如果通过HttpClientOptions指定了metricsName，基础名字为vertx.http.clients.<metricsName>

	  public synchronized HttpClientMetrics<?, ?, ?, ?, ?> createMetrics(HttpClient client, HttpClientOptions options) {
	    String name = options.getMetricsName();
	    String baseName;
	    if (name != null && name.length() > 0) {
	      baseName = nameOf("http.clients", name);
	    } else {
	      baseName = nameOf("http.clients");
	    }
	    HttpClientReporter reporter = clientReporters.computeIfAbsent(baseName, n -> new HttpClientReporter(registry, baseName, null));
	    return new HttpClientMetricsImpl(this, reporter, options, this.options.getMonitoredHttpClientUris(), this.options.getMonitoredHttpClientEndpoint());
	  }

Gauge

- connections.max-pool-size 最大连接池数量
- connections.pool-ratio - A ratio Gauge of the open connections / max connection pool size

Timer

- responses-1xx 响应码是1xx的Timer
- responses-2xx 响应码是2xx的Timer
- responses-3xx 响应码是3xx的Timer
- responses-4xx 响应码是4xx的Timer
- responses-5xx 响应码是5xx的Timer

也可以分开根据每个远程服务器的提供度量指标

Timer

- endpoint.<host:port>.queue-delay 待处理的请求在队列中的等待时间
- endpoint.<host:port>.usage 请求开始到结束的耗时
- endpoint.<host:port>.ttfb 收到请求的响应到请求完成的耗时

Counter

- endpoint.<host:port>.queue-size 实际的队列大小
- endpoint.<host:port>.open-netsockets 实际打开的连接数量
- endpoint.<host:port>.in-use A Counter of the actual number of request/response

HTTP Client的请求在连接池ConnectionManager里面，实现了对度量指标的更新
代码比较复杂，还没仔细看，阅读连接池实现的时候在看

## Datagram socket metrics
平时基本没用到

## Pool metrics
连接池的度量指标

基础名字为: vertx.pool.<type>.<name>，type表示连接池的类型（例如 worker,datasource），

它包括了下列度量指标

Timer

- queue-delay 从连接池中获取资源的耗时, 例如：在队列中的等待时间
- usage 资源被使用的时间

Counter

- queue-size 队列中等待的数量
- in-use - 正在被使用的资源数量
- pool-ratio - A ratio Gauge of the in use resource / pool size
- max-pool-size 队列的最大数量

Vertx在创建的时候就创建了两个metric对象
vertx.pool.worker.vert.x-worker-thread和 vertx.pool.worker.vert.x-internal-blocking

	PoolMetrics workerPoolMetrics = isMetricsEnabled() ? metrics.createMetrics(workerExec, "worker", "vert.x-worker-thread", options.getWorkerPoolSize()) : null;
	PoolMetrics internalBlockingPoolMetrics = isMetricsEnabled() ? metrics.createMetrics(internalBlockingExec, "worker", "vert.x-internal-blocking", options.getInternalBlockingPoolSize()) : null;

	  @Override
	  public <P> PoolMetrics<?> createMetrics(P pool, String poolType, String poolName, int maxPoolSize) {
	    String baseName = nameOf("pools", poolType, poolName);
	    return new PoolMetricsImpl(registry, baseName, maxPoolSize);
	  }

PoolMetricsImpl在创建时创建了下列度量指标

    this.queueSize = counter("queue-size");
    this.queueDelay = timer("queue-delay");
    this.usage = timer("usage");
    this.inUse = counter("in-use");
    if (maxSize > 0) {
      RatioGauge gauge = new RatioGauge() {
        @Override
        protected Ratio getRatio() {
          return Ratio.of(inUse.getCount(), maxSize);
        }
      };
      gauge(gauge, "pool-ratio");
      gauge(() -> maxSize, "max-pool-size");
    }

在executeBlocking方法中会对连接池的度量指标做更新

	  <T> void executeBlocking(Action<T> action, Handler<Future<T>> blockingCodeHandler,
	      Handler<AsyncResult<T>> resultHandler,
	      Executor exec, PoolMetrics metrics) {
	    Object queueMetric = metrics != null ? metrics.submitted() : null;
	    try {
	      exec.execute(() -> {
	        Object execMetric = null;
	        if (metrics != null) {
	          execMetric = metrics.begin(queueMetric);
	        }
	        Future<T> res = Future.future();
	        try {
	          if (blockingCodeHandler != null) {
	            ContextImpl.setContext(this);
	            blockingCodeHandler.handle(res);
	          } else {
	            T result = action.perform();
	            res.complete(result);
	          }
	        } catch (Throwable e) {
	          res.fail(e);
	        }
	        if (metrics != null) {
	          metrics.end(execMetric, res.succeeded());
	        }
	        if (resultHandler != null) {
	          runOnContext(v -> res.setHandler(resultHandler));
	        }
	      });
	    } catch (RejectedExecutionException e) {
	      // Pool is already shut down
	      if (metrics != null) {
	        metrics.rejected(queueMetric);
	      }
	      throw e;
	    }
	  }


Vert.x有些组件的度量实现比较复杂，我并没有深入去了解具体的实现
