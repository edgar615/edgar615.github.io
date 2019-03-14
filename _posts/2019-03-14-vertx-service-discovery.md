---# Service Discovery

**前部分内容基本来自于官方文档**

Vert.x提供了服务发现组件，每个服务都被封装为一个Record对象

服务提供者：

- 发布一个服务
- 注销一个服务
- 更新已发布的服务

服务消费者：

- 查询服务
- 使用选定的服务
- 一旦用户用完服务，释放服务
- 监听服务的发布、注销和修改

# 核心概念
**Service records 服务记录**

用来描述服务提供者发布的服务的对象。它包含名称，元数据，地址（描述服务在哪里）等属性

**Service Provider and publisher 服务提供者和发布者**

用来发布或注销一个服务

**Service Consumer 服务消费者**

服务消费者在服务发现中查找服务（可能返回0个或多个服务记录）。从这些记录中，消费者可以获得ServiceReference对象，用来表示消费者和提供者之间的绑定。这个对象允许消费者使用该服务，并释放服务。**需要注意释放服务引用来清理对象和更新服务**

**Service object 服务对象**

服务对象是提供访问服务的对象。它可以以各种形式出现，如代理、客户端，甚至有些不存在的服务类型。服务对象的性质取决于服务类型。

**Service types 服务类型**

服务可以是各种类型的服务，比如功能性服务，数据库，REST API等。每个服务类型定义了：
	
- 这个服务的地址（URL，事件总线地址，IP/DNS）
- 服务对象的性质（服务代理，HTTP客户端，消息消费者）

某些服务类型由服务发现组件实现并提供，我们也可以添加自己的服务类型。

**Service events 服务事件**

每次发布或注销服务时，总会在事件总线上触发一个announce事件。这个事件包括已经修改的服务记录。此外为了跟踪谁在使用服务，每次ServiceReference的绑定和释放都会触发一个usage事件


**Backend**

服务发现使用分布式结构存储服务记录，因此群集的所有成员都可以访问所有记录。也可以实现你自己的servicediscoverybackend SPI来存储服务记录.

# 使用
## 创建一个服务发现实例
<pre class="line-numbers "><code class="language-java">
	ServiceDiscovery discovery = ServiceDiscovery.create(vertx);
</code></pre>
使用默认配置创建了一个服务发现实例，发布和注销服务时触发的事件地址为`vertx.discovery.announce`，绑定和释放ServiceReference时触发的事件地址为`vertx.discovery.usage`

我们可以使用自定义的配置
<pre class="line-numbers "><code class="language-java">
    ServiceDiscovery discovery = ServiceDiscovery.create(vertx,
                new ServiceDiscoveryOptions()
                        .setAnnounceAddress("service-announce")
                        .setName("my-name"));
</code></pre>

Vertx.官方目前提供了下面几种类型的服务发现（ServiceType）

- HttpEndpoint restAPI
- EventBusService  服务代理
- MessageSource 消息
- JDBCDataSource JDBC

## 发布服务
发布服务的过程如下：

- 为服务提供者创建服务记录(Record)
- 发布这个服务记录
- 保存已发布的服务记录，用来注销或者修改服务
<pre class="line-numbers "><code class="language-java">
    Record record = new Record()
            .setType("eventbus-service-proxy")
            .setLocation(new JsonObject().put("endpoint", "the-service-address"))
            .setName("my-service")
            .setMetadata(new JsonObject().put("some-label", "some-value"));
    discovery.publish(record, ar -> {
      if (ar.succeeded()) {
        Record publishedRecord = ar.result();
        System.out.println(record.getRegistration());
        System.out.println(publishedRecord.getLocation());
        System.out.println(publishedRecord.getType());
        System.out.println(publishedRecord.getMetadata());
        System.out.println(publishedRecord.getName());
      } else {

      }
    });
</code></pre>
服务发布之后，会给服务记录生成一个唯一的ID(registration)

Vert.x也提供了一下基本服务类型用于创建服务记录（后面再描述）

## 注销服务
在服务发布之后，可以使用registration将服务注销
<pre class="line-numbers "><code class="language-java">
	discovery.unpublish(record.getRegistration(), ar -> {
	  if (ar.succeeded()) {
	    // Ok
	  } else {
	    // cannot un-publish the service, may have already been removed, or the record is not published
	  }
	});
</code></pre>
## 查找服务
服务消费者可以通过getRecord搜索单个服务记录,或者通过getRecords搜索所有匹配的服务记录。

服务消费者可以通过一个过滤器来匹配服务，过滤器有两种：

- Function<Record, Boolean> filter，以Record为参数，返回一个bool的函数
- JsonObject，对JsonObject中的每个属性检查，必须所有的属性都匹配的记录才是要搜索的服务记录。

- { "name" = "a" } => 匹配所有name属性为a的记录
- { "color" = "*" } => 匹配所有设置了color属性的记录
- { "color" = "red" } => 匹配所有color属性为red的记录
- { "color" = "red", "name" = "a"} => 匹配所有name属性为a且color属性为red的记录

如果JsonObject为null或者空，会匹配所有的服务记录

**搜索全部记录**
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> true, ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords((JsonObject) null, ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
输出
```
	some-rest-api4:{"num":1}
	some-rest-api3:{"color":"white"}
	some-rest-api1:{}
	some-rest-api2:{"color":"red"}
```
**搜索所有http类型的记录**
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> r.getType().equals("http-endpoint"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(new JsonObject().put("type", "http-endpoint"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
输出
```
	some-rest-api4:{"num":1}
	some-rest-api3:{"color":"white"}
	some-rest-api1:{}
	some-rest-api2:{"color":"red"}
```
**搜索所有名称为some-rest-api1**的记录
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> r.getName().equals("some-rest-api1"), ar -> {
        List<Record> records = ar.result();
        System.out.println("some-rest-api1");
        for (Record record : records) {
            System.out.println("-" + record.getName());
        }
    });

    discovery.getRecords(new JsonObject().put("name", "some-rest-api1"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
**搜索所有color=red的记录**
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> "red".equals(r.getMetadata().getString("color")), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });

    discovery.getRecords(new JsonObject().put("color", "red"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
**搜索所有包含color属性的记录**
<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> r.getMetadata().containsKey("color"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });

    discovery.getRecords(new JsonObject().put("color", "*"), ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
            System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
**只搜索单个记录**
<pre class="line-numbers "><code class="language-java">
    discovery.getRecord(r -> true, ar -> {
        Record record = ar.result();
        System.out.println(record.getName() + ":" + record.getMetadata());
    });
</code></pre>
**getRecords和getRecord默认只会查询status=UP的记录**，如果需要查询其他状态的记录，有两种方式：

- 使用JsonObject过滤status属性，可以指定某个具体的status或者使用*表示全部
- 使用Function过滤器，并将includeOutOfService参数设为true

## 获取服务引用
一旦你获取到了一个服务记录，就可以获取到一个ServiceReference以及服务对象。
ServiceReference用来表示与服务提供者之间的绑定关系
<pre class="line-numbers "><code class="language-java">
    ServiceReference serviceReference = discovery.getReference(record);
    HttpClient httpClient = serviceReference.get();
    // You need to path the complete path
    httpClient.getNow(record.getLocation().getString("root"), response -> {
      // Dont' forget to release the service
      serviceReference.release();
    });
</code></pre>
一旦使用完ServiceReference，需要释放它.

我们也可以用discovery.getReferenceWithConfiguration来配置服务对象

Vertx.官方目前提供了下面几种类型的服务发现（ServiceType）

- HttpEndpoint restAPI，对应的服务对象为HttpClient
- EventBusService  异步RPC服务代理，对应的服务对象为对应的服务接口
- MessageSource 消息，对应的服务对象为MessageConsumer
- JDBCDataSource JDBC，对应的服务对象为JDBCClient

Vert.x还定义了一种unknown类型的服务类型，但这种类型无法获取对应的服务引用。我们可以通过服务记录中的location和metadata属性类创建服务对象。**这些服务对象的使用不会触发usage事件**.
当然，我们也可以实现自定义的服务类型（后面描述）

### HTTP endpoints
**发布**
<pre class="line-numbers "><code class="language-java">
	Record record1 = HttpEndpoint.createRecord(
	    "some-http-service", // The service name
	    "localhost", // The host
	    8433, // the port
	    "/api" // the root of the service
	);
	
	discovery.publish(record1, ar -> {
	  // ...
	});
	
	Record record2 = HttpEndpoint.createRecord(
	    "some-other-name", // the service name
	    true, // whether or not the service requires HTTPs
	    "localhost", // The host
	    8433, // the port
	    "/api", // the root of the service
	    new JsonObject().put("some-metadata", "some value")
	);
</code></pre>
**消费**
<pre class="line-numbers "><code class="language-java">
	discovery.getRecord(new JsonObject().put("name", "some-http-service"), ar -> {
	  if (ar.succeeded()  && ar.result() != null) {
	    // Retrieve the service reference
	    ServiceReference reference = discovery.getReference(ar.result());
	    // Retrieve the service object
	    HttpClient client = reference.get();
	
	    // You need to path the complete path
	    client.getNow("/api/persons", response -> {
	
	      // ...
	
	      // Dont' forget to release the service
	      reference.release();
	
	    });
	  }
	});
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
	HttpEndpoint.getClient(discovery, new JsonObject().put("name", "some-http-service"), ar -> {
	  if (ar.succeeded()) {
	    HttpClient client = ar.result();
	
	    // You need to path the complete path
	    client.getNow("/api/persons", response -> {
	
	      // ...
	
	      // Dont' forget to release the service
	      ServiceDiscovery.releaseServiceObject(discovery, client);
	
	    });
	  }
	});
</code></pre>
### Event bus services
**发布**
<pre class="line-numbers "><code class="language-java">
	Record record = EventBusService.createRecord(
	    "some-eventbus-service", // The service name
	    "address", // the service address,
	    "examples.MyService", // the service interface as string
	    new JsonObject()
	        .put("some-metadata", "some value")
	);
	
	discovery.publish(record, ar -> {
	  // ...
	});
</code></pre>
也可以直接使用接口的class
<pre class="line-numbers "><code class="language-java">
	Record record = EventBusService.createRecord(
        "some-eventbus-service", // The service name
        "address", // the service address,
        MyService.class // the service interface
        );
	
	discovery.publish(record, ar -> {
	// ...
	});
</code></pre>
**消费**
<pre class="line-numbers "><code class="language-java">
	discovery.getRecord(new JsonObject().put("name", "some-eventbus-service"), ar -> {
	if (ar.succeeded() && ar.result() != null) {
        // Retrieve the service reference
        ServiceReference reference = discovery.getReference(ar.result());
        // Retrieve the service object
        MyService service = reference.get();

        // Dont' forget to release the service
        reference.release();
	}
	});
</code></pre>
### Message source
**发布**
<pre class="line-numbers "><code class="language-java">
	Record record = MessageSource.createRecord(
	    "some-message-source-service", // The service name
	    "some-address" // The event bus address
	);
	
	discovery.publish(record, ar -> {
	  // ...
	});
	
	record = MessageSource.createRecord(
	    "some-other-message-source-service", // The service name
	    "some-address", // The event bus address
	    "examples.MyData" // The payload type
	);
</code></pre>
也可以直接使用Payload的class
<pre class="line-numbers "><code class="language-java">
	Record record1 = MessageSource.createRecord(
        "some-message-source-service", // The service name
        "some-address", // The event bus address
        JsonObject.class // The message payload type
        );
	
	Record record2 = MessageSource.createRecord(
        "some-other-message-source-service", // The service name
        "some-address", // The event bus address
        JsonObject.class, // The message payload type
        new JsonObject().put("some-metadata", "some value")
	);
</code></pre>
**消费**
<pre class="line-numbers "><code class="language-java">
	discovery.getRecord(new JsonObject().put("name", "some-message-source-service"), ar -> {
	  if (ar.succeeded() && ar.result() != null) {
	    // Retrieve the service reference
	    ServiceReference reference = discovery.getReference(ar.result());
	    // Retrieve the service object
	    MessageConsumer<JsonObject> consumer = reference.get();
	
	    // Attach a message handler on it
	    consumer.handler(message -> {
	      // message handler
	      JsonObject payload = message.body();
	    });
	
	    // ...
	    // when done
	    reference.release();
	  }
	});
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
	MessageSource.<JsonObject>getConsumer(discovery, new JsonObject().put("name", "some-message-source-service"), ar -> {
	  if (ar.succeeded()) {
	    MessageConsumer<JsonObject> consumer = ar.result();
	
	    // Attach a message handler on it
	    consumer.handler(message -> {
	      // message handler
	      JsonObject payload = message.body();
	    });
	    // ...
	
	    // Dont' forget to release the service
	    ServiceDiscovery.releaseServiceObject(discovery, consumer);
	
	  }
	});
</code></pre>
### JDBC Data source
**发布**
<pre class="line-numbers "><code class="language-java">
	Record record = JDBCDataSource.createRecord(
	    "some-data-source-service", // The service name
	    new JsonObject().put("url", "some jdbc url"), // The location
	    new JsonObject().put("some-metadata", "some-value") // Some metadata
	);
	
	discovery.publish(record, ar -> {
	  // ...
	});
</code></pre>
**消费**
<pre class="line-numbers "><code class="language-java">
	discovery.getRecord(
	    new JsonObject().put("name", "some-data-source-service"),
	    ar -> {
	      if (ar.succeeded() && ar.result() != null) {
	        // Retrieve the service reference
	        ServiceReference reference = discovery.getReferenceWithConfiguration(
	            ar.result(), // The record
	            new JsonObject().put("username", "clement").put("password", "*****")); // Some additional metadata
	
	        // Retrieve the service object
	        JDBCClient client = reference.get();
	
	        // ...
	
	        // when done
	        reference.release();
	      }
	    });
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
	JDBCDataSource.<JsonObject>getJDBCClient(discovery,
	    new JsonObject().put("name", "some-data-source-service"),
	    new JsonObject().put("username", "clement").put("password", "*****"), // Some additional metadata
	    ar -> {
	      if (ar.succeeded()) {
	        JDBCClient client = ar.result();
	
	        // ...
	
	        // Dont' forget to release the service
	        ServiceDiscovery.releaseServiceObject(discovery, client);
	
	      }
	    });
</code></pre>
### Redis Data source
**发布**
<pre class="line-numbers "><code class="language-java">
	Record record = RedisDataSource.createRecord(
	  "some-redis-data-source-service", // The service name
	  new JsonObject().put("url", "localhost"), // The location
	  new JsonObject().put("some-metadata", "some-value") // Some metadata
	);
	
	discovery.publish(record, ar -> {
	  // ...
	});
</code></pre>
**消费**
<pre class="line-numbers "><code class="language-java">
	discovery.getRecord(
	  new JsonObject().put("name", "some-redis-data-source-service"), ar -> {
	    if (ar.succeeded() && ar.result() != null) {
	      // Retrieve the service reference
	      ServiceReference reference = discovery.getReference(ar.result());
	
	      // Retrieve the service instance
	      RedisClient client = reference.get();
	
	      // ...
	
	      // when done
	      reference.release();
	    }
	  });
</code></pre>
或者
<pre class="line-numbers "><code class="language-java">
	RedisDataSource.getRedisClient(discovery,
	  new JsonObject().put("name", "some-redis-data-source-service"),
	  ar -> {
	    if (ar.succeeded()) {
	      RedisClient client = ar.result();
	
	      // ...
	
	      // Dont' forget to release the service
	      ServiceDiscovery.releaseServiceObject(discovery, client);
	
	    }
	  });
</code></pre>
## 监听服务的发布和注销
每次服务提供者发布或者注销，都会有一个事件被发布到vertx.discovery.announce地址上（这个地址可以通过ServiceDiscoveryOptions修改）

事件中收到的状态属性表示服务记录的新状态

- UP : 服务可用，可以使用这个服务
- DOWN : 服务不可用，不应该再使用这个服务
- OUT_OF_SERVICE : 服务还没有启动，不应该使用这个服务，但是可能过一段事件之后服务可用

## 监听服务的使用
每次一个服务引用被获取（绑定）或者释放，都会有一个事件被发布到vertx.discovery.usage地址上（这个地址可以通过ServiceDiscoveryOptions修改）

事件的JsonObject包括

- record 服务记录
- type 事件类型 bind或者release
- ID 服务发现的ID,它可能是ServiceDiscovery的名字或者的节点的ID，可以通过ServiceDiscoveryOptions指定，在单机模式下默认为localhost，在集群模式下是该节点在集群下的ID

也可以将usage的地址设为null来关闭usage事件的触发

示例
<pre class="line-numbers "><code class="language-java">
    ServiceDiscovery discovery = ServiceDiscovery.create(vertx);
    vertx.eventBus().consumer("vertx.discovery.announce", msg -> {
      System.out.println("announce: " + msg.body());
    });

    vertx.eventBus().consumer("vertx.discovery.usage", msg -> {
      System.out.println("usage: " + msg.body());
    });

    vertx.setPeriodic(5000, l -> {
      discovery.getRecords(r -> true, ar -> {
        List<Record> records = ar.result();
        for (Record record : records) {
          ServiceReference serviceReference = discovery.getReference(record);
          HttpClient httpClient = serviceReference.get();
          serviceReference.release();
        }
      });
    });
    discovery.close();
</code></pre>
输出
```
	announce: {"location":{"endpoint":"http://localhost:8080/api","host":"localhost","port":8080,"root":"/api","ssl":false},"metadata":{},"name":"some-rest-api","status":"UP","type":"http-endpoint"}
	announce: {"location":{"host":"localhost","endpoint":"http://localhost:8080/api","port":8080,"ssl":false,"root":"/api"},"metadata":{},"name":"some-rest-api","status":"DOWN","type":"http-endpoint"}
	usage: {"type":"bind","record":{"name":"some-rest-api","location":{"host":"localhost","endpoint":"http://localhost:8080/api","port":8080,"ssl":false,"root":"/api"},"metadata":{},"registration":"214bafff-2da9-4c26-b9d9-bc31f08febaa","type":"http-endpoint","status":"UP"},"id":"cfc09ae8-6a03-438c-962f-bb7b4ff9b4ae"}
	usage: {"type":"release","record":{"name":"some-rest-api","location":{"host":"localhost","endpoint":"http://localhost:8080/api","port":8080,"ssl":false,"root":"/api"},"metadata":{},"registration":"214bafff-2da9-4c26-b9d9-bc31f08febaa","type":"http-endpoint","status":"UP"},"id":"cfc09ae8-6a03-438c-962f-bb7b4ff9b4ae"}
```
## Service discovery bridges
通过Bridges可以从其他服务发现组件上（Docker, Kubernates, Consul,Zookeeper）导入导出服务提供者。我们只需要实现ServiceImporter接口来定义自己的Bridges，并通过ServiceDiscovery.registerServiceImporter方法注册Bridges

当Bridges被注册之后，start会被执行,在导入（或导出）服务记录之后，必须将future设置为完成。
如果start方法是阻塞方法，必须使用executeBlocking来运行。

当ServiceDiscovery停止时，Bridges也被停止，stop方法会被调用用来清理对应的资源，删除导入（或导出）的服务记录。同样，在执行完毕之后不需要将future设置为完成

**在集群模式下，只需要有一个节点注册bridge，所有的服务都会通过事件的方式广播给所有的成员**

# 扩展
## 自定义服务类型
可以按照下面的步骤实现自定义的服务类型

- (可选)创建一个继承ServiceType的接口.
- 创建一个实现了上一步接口（或者ServiceType）的实现类. 这个实现类包括一个名字和对应ServiceReference的创建方法.
- 创建一个继承了AbstractServiceReference的ServiceReference类，这个子类需要实现retrieve() 用于创建对应的服务对象.**AbstractServiceReference会将创建的服务对象缓存，在release之前这个方法只会调用一次**。如果服务对象需要一些清理操作，还需要重写close()方法
- 在META-INF/services/io.vertx.ext.discovery.spi.ServiceType文件中加入自定义的ServiceType的实现类

## 自定义bridge
Vert.x官方的Zookeeper桥接尚未正式发布，下面是参考Consul实现的zookeeper桥接

创建一个辅助类，维护了Record和服务ID之间的关系，并且封装了服务的发布和注销方法
<pre class="line-numbers "><code class="language-java">
	public class ImportedZookeeperService {
	
	  private final String name;
	
	  private final Record record;
	
	  private final String id;
	
	  /**
	   * Creates a new instance of {@link ImportedZookeeperService}.
	   *
	   * @param name   the service name
	   * @param id     the service id, may be the name
	   * @param record the record (not yet registered)
	   */
	  public ImportedZookeeperService(String name, String id, Record record) {
	    Objects.requireNonNull(name);
	    Objects.requireNonNull(id);
	    Objects.requireNonNull(record);
	    this.name = name;
	    this.record = record;
	    this.id = id;
	  }
	
	  /**
	   * @return the name
	   */
	  public String name() {
	    return name;
	  }
	
	  /**
	   * Registers the service and completes the given future when done.
	   *
	   * @param publisher  the service publisher instance
	   * @param completion the completion future
	   * @return the current {@link ImportedZookeeperService}
	   */
	  public ImportedZookeeperService register(ServicePublisher publisher,
	                                           Future<ImportedZookeeperService> completion) {
	    publisher.publish(record, ar -> {
	      if (ar.succeeded()) {
	        record.setRegistration(ar.result().getRegistration());
	        completion.complete(this);
	      } else {
	        completion.fail(ar.cause());
	      }
	    });
	    return this;
	  }
	
	  /**
	   * Unregisters the service and completes the given future when done, if not {@code null}
	   *
	   * @param publiher   the service publisher instance
	   * @param completion the completion future
	   */
	  public void unregister(ServicePublisher publiher, Future<Void> completion) {
	    if (record.getRegistration() != null) {
	      publiher.unpublish(record.getRegistration(), ar -> {
	        if (ar.succeeded()) {
	          record.setRegistration(null);
	        }
	        if (completion != null) {
	          completion.complete();
	        }
	      });
	    } else {
	      if (completion != null) {
	        completion.fail("Record not published");
	      }
	    }
	  }
	
	  /**
	   * @return the id
	   */
	  String id() {
	    return id;
	  }
	}
</code></pre>
2. 实现Zookeeper的ServiceImporter

start方法会创建zookeeper的连接，并通过retrieveIndividualServices方法来从zookeeper中导入服务记录;
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public void start(Vertx vertx, ServicePublisher publisher, JsonObject configuration,
	                    Future<Void> future) {
	    this.vertx = vertx;
	    this.publisher = publisher;
	    this.zkConnect = configuration.getString("zookeeper.connect", "localhost:2181");
	    this.basePath = configuration.getString("zookeeper.path", "/services");
	    this.sleepMsBetweenRetries = configuration.getInteger("zookeeper.retry.sleep", 1000);
	    this.retryTimes = configuration.getInteger("zookeeper.retry.times", 5);
	    // When the bridge is configured, ready and has imported / exported the initial services, it
	    // must complete the given Future. If the bridge starts method is blocking, it must use an
	    // executeBlocking construct, and complete the given future object
	    vertx.<Void>executeBlocking(
	            f -> {
	              try {
	                client = CuratorFrameworkFactory.newClient(zkConnect,
	                                                           new RetryNTimes(retryTimes,
	                                                                           sleepMsBetweenRetries));
	                client.start();
	
	                serviceDiscovery =
	                        ServiceDiscoveryBuilder.builder(String.class)
	                                .basePath(basePath)
	                                .watchInstances(true)
	                                .client(client).build();
	
	                serviceDiscovery.start();
	
	                cache = TreeCache.newBuilder(client, basePath).build();
	                cache.start();
	                cache.getListenable().addListener(this);
	
	                f.complete();
	              } catch (Exception e) {
	                future.fail(e);
	              }
	            },
	            ar -> {
	              if (ar.failed()) {
	                future.fail(ar.cause());
	              } else {
	                Future<Void> f = Future.future();
	                f.setHandler(x -> {
	                  if (x.failed()) {
	                    future.fail(x.cause());
	                  } else {
	                    started = true;
	                    future.complete();
	                  }
	                });
	                retrieveIndividualServices(f);
	              }
	            }
	    );
	  }
</code></pre>
retrieveIndividualServices方法是服务导入的核心逻辑，它先从zookeeper中读取到所有的服务实例，然后将zookeeper中的服务实例发布，并将zookeeper中不存在的服务实例注销
<pre class="line-numbers "><code class="language-java">
	  private synchronized void retrieveIndividualServices(Future<Void> completed) {
	    List<ServiceInstance<String>> instances = new ArrayList<>();
	    try {
	      Collection<String> names = serviceDiscovery.queryForNames();
	      for (String name : names) {
	        instances.addAll(serviceDiscovery.queryForInstances(name));
	      }
	    } catch (KeeperException.NoNodeException e) {
	      // no services
	      // Continue
	    } catch (Exception e) {
	      if (completed != null) {
	        completed.fail(e);
	      } else {
	        LOGGER.error("Unable to retrieve service instances from Zookeeper", e);
	        return;
	      }
	    }
	    Future<List<ImportedZookeeperService>> future = Future.future();
	    importService(instances, future);
	    future.setHandler(ar -> {
	      if (ar.failed()) {
	        //completed.fail(ar.cause());
	        unregisterAllServices(completed);
	      } else {
	        List<ImportedZookeeperService> services = future.result();
	        List<String> retrievedIds =
	                services.stream().map(ImportedZookeeperService::id).collect(Collectors.toList());
	        List<String> existingIds =
	                imports.stream().map(ImportedZookeeperService::id).collect(Collectors.toList());
	
	        LOGGER.trace("Imported services: " + existingIds + ", Retrieved services form Zookeeper: "
	                     + retrievedIds);
	
	        services.forEach(svc -> {
	          String id = svc.id();
	
	          if (!existingIds.contains(id)) {
	            LOGGER.info("Imported service: " + id);
	            imports.add(svc);
	          }
	        });
	
	        //使用foreach删除会出现ConcurrentModificationException
	        Iterator<ImportedZookeeperService> iterator = imports.iterator();
	        while (iterator.hasNext()) {
	          ImportedZookeeperService svc = iterator.next();
	          if (!retrievedIds.contains(svc.id())) {
	            LOGGER.info("Unregistering " + svc.id());
	            iterator.remove();
	            svc.unregister(publisher, null);
	          }
	        }
	
	        completed.complete();
	      }
	    });
	  }
</code></pre>
close方法中将所有的服务实例注销
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public void close(Handler<Void> closeHandler) {
	    Future<Void> done = Future.future();
	    //删除所有服务实例
	    unregisterAllServices(done);
	
	    done.setHandler(v -> {
	      try {
	        if (cache != null) {
	          CloseableUtils.closeQuietly(cache);
	        }
	        if (serviceDiscovery != null) {
	          CloseableUtils.closeQuietly(serviceDiscovery);
	        }
	        if (client != null) {
	          CloseableUtils.closeQuietly(client);
	        }
	      } catch (Exception e) {
	        // Ignore them
	      }
	      closeHandler.handle(null);
	    });
	  }

	  private synchronized void unregisterAllServices(Future<Void> completed) {
	    List<Future> list = new ArrayList<>();
	
	    imports.forEach(svc -> {
	      Future<Void> unreg = Future.future();
	      svc.unregister(publisher, unreg);
	      list.add(unreg);
	    });
	
	    CompositeFuture.all(list).setHandler(x -> {
	      if (x.failed()) {
	        completed.fail(x.cause());
	      } else {
	        completed.complete();
	      }
	    });
	  }
</code></pre>
同时，我们还需要监听整个服务节点的变化，来重新导入服务实例
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public void childEvent(CuratorFramework curatorFramework,
	                         TreeCacheEvent treeCacheEvent) throws Exception {
	    if (started) {
	      retrieveIndividualServices(Future.future());
	    }
	  }
</code></pre>

# 源码分析

## ServiceDiscovery
**构造方法**
<pre class="line-numbers "><code class="language-java">
	  public DiscoveryImpl(Vertx vertx, ServiceDiscoveryOptions options) {
	    this.vertx = vertx;
	    this.announce = options.getAnnounceAddress();
	    this.usage = options.getUsageAddress();
	
	    this.backend = getBackend(options.getBackendConfiguration().getString("backend-name", null));
	    this.backend.init(vertx, options.getBackendConfiguration());
	
	    this.id = options.getName() != null ? options.getName() : getNodeId(vertx);
	    this.options = options;
	  }
</code></pre>
DiscoveryImpl的构造方法主要做一些配置工作
- 配置announce和usage事件的地址
- 初始化对应的backend
- 创建对应的ID

getBackend主要送从classpath中读取对应的ServiceDiscoveryBackend对象，如果没有找到合适的ServiceDiscoveryBackend，就使用默认值DefaultServiceDiscoveryBackend。关于ServiceDiscoveryBackend会在后面在描述

**registerServiceImporter**
直接调用importer.start方法启动对应的importer，在成功之后将import加入到importers中
<pre class="line-numbers "><code class="language-java">
	  public ServiceDiscovery registerServiceImporter(ServiceImporter importer, JsonObject configuration,
	                                                  Handler<AsyncResult<Void>> completionHandler) {
	    JsonObject conf;
	    if (configuration == null) {
	      conf = new JsonObject();
	    } else {
	      conf = configuration;
	    }
	
	    Future<Void> completed = Future.future();
	    completed.setHandler(
	        ar -> {
	          if (ar.failed()) {
	            LOGGER.error("Cannot start the service importer " + importer, ar.cause());
	            if (completionHandler != null) {
	              completionHandler.handle(Future.failedFuture(ar.cause()));
	            }
	          } else {
	            importers.add(importer);
	            LOGGER.info("Service importer " + importer + " started");
	
	            if (completionHandler != null) {
	              completionHandler.handle(Future.succeededFuture(null));
	            }
	          }
	        }
	    );
	
	    importer.start(vertx, this, conf, completed);
	    return this;
	  }
</code></pre>
**close方法**
将所有import、export关闭，将所有的ServiceReference释放，并删除绑定关系。
<pre class="line-numbers "><code class="language-java">
	  public void close() {
	    LOGGER.info("Stopping service discovery");
	    List<Future> futures = new ArrayList<>();
	    for (ServiceImporter importer : importers) {
	      Future<Void> future = Future.future();
	      // TODO Change this call to call close, once the stop method has been removed
	      importer.stop(vertx, this, future);
	      futures.add(future);
	    }
	
	    for (ServiceExporter exporter : exporters) {
	      Future<Void> future = Future.future();
	      exporter.close(future::complete);
	      futures.add(future);
	    }
	
	    bindings.forEach(ServiceReference::release);
	    bindings.clear();
	
	    CompositeFuture.all(futures).setHandler(ar -> {
	      if (ar.succeeded()) {
	        LOGGER.info("Discovery bridges stopped");
	      } else {
	        LOGGER.warn("Some discovery bridges did not stopped smoothly", ar.cause());
	      }
	    });
	  }
</code></pre>
**publish 发布服务记录**

将服务记录保存到ServiceDiscoveryBackend。如果注册有exporter，还需要调用exporter的onPublish方法向对应的Bridges发布服务。然后触发announce对象。
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public void publish(Record record, Handler<AsyncResult<Record>> resultHandler) {
	    Status status = record.getStatus() != null
	        && record.getStatus() != Status.UNKNOWN
	        && record.getStatus() != Status.DOWN
	        ? record.getStatus() : Status.UP;
	
	    backend.store(record.setStatus(status), resultHandler);
	    for (ServiceExporter exporter : exporters) {
	      exporter.onPublish(new Record(record));
	    }
	    Record announcedRecord = new Record(record);
	    announcedRecord
	        .setRegistration(null)
	        .setStatus(status);
	    vertx.eventBus().publish(announce, announcedRecord.toJson());
	  }
</code></pre>
**unpublish 注销服务**

从ServiceDiscoveryBackend中删除服务记录。如果注册有exporter，还需要调用exporter的onUnpublish方法向对应的Bridges注销服务。然后触发announce事件
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public void unpublish(String id, Handler<AsyncResult<Void>> resultHandler) {
	    backend.remove(id, record -> {
	      if (record.failed()) {
	        resultHandler.handle(Future.failedFuture(record.cause()));
	        return;
	      }
	
	      for (ServiceExporter exporter : exporters) {
	        exporter.onUnpublish(id);
	      }
	
	      Record announcedRecord = new Record(record.result());
	      announcedRecord
	          .setRegistration(null)
	          .setStatus(Status.DOWN);
	      vertx.eventBus().publish(announce, announcedRecord.toJson());
	      resultHandler.handle(Future.succeededFuture());
	    });
	
	  }
</code></pre>
**getReferenceWithConfiguration方法**

创建对应的ServiceReference，然后在bindings中保存绑定关系，发布bind类型的usage事件
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public ServiceReference getReferenceWithConfiguration(Record record, JsonObject configuration) {
	    ServiceReference reference = ServiceTypes.get(record).get(vertx, this, record, configuration);
	    bindings.add(reference);
	    sendBindEvent(reference);
	    return reference;
	  }
</code></pre>
**release方法**

删除bindings中的绑定关系，释放ServiceReference，发布release类型的usage事件
<pre class="line-numbers "><code class="language-java">
	  @Override
	  public boolean release(ServiceReference reference) {
	    boolean removed = bindings.remove(reference);
	    reference.release();
	    sendUnbindEvent(reference);
	    return removed;
	  }
</code></pre>
**getRecord方法比较简单，不做描述**

## ServiceDiscoveryBackend
服务实例的存储类，默认使用分布式内存存储（DefaultServiceDiscoveryBackend）。DefaultServiceDiscoveryBackend并不复杂，它内部使用一个AsyncMap来保存服务实例。
<pre class="line-numbers "><code class="language-java">
	  private AsyncMap<String, String> registry;
	
	  @Override
	  public void init(Vertx vertx, JsonObject config) {
	    this.registry = new AsyncMap<>(vertx, "service.registry");
	  }
</code></pre>
AsyncMap的构造方法会判断节点是否在集群模式下，如果在集群模式下会使用clusterManager提供的分布式MAP，否则使用一个本地MAP(多个ServiceDiscovery共享同一个map).(LocalMapWrapper是借助ConcurrentMap对map的一个简单封装)
<pre class="line-numbers "><code class="language-java">
	//这是3.4.X版本的源码，会有性能问题，3.5.0之后的版本已经修改了实现方式
	  public AsyncMap(Vertx vertx, String name) {
	    this.vertx = vertx;
	    ClusterManager clusterManager = ((VertxInternal) vertx).getClusterManager();
	    if (clusterManager == null) {
	      syncMap = new LocalMapWrapper<>(vertx.sharedData().<K, V>getLocalMap(name));
	    } else {
	      syncMap = clusterManager.getSyncMap(name);
	    }
	  }
</code></pre>
**store方法**
为每个服务记录生成一个唯一ID，**只有ID为null的服务记录才表示未被发布**
<pre class="line-numbers "><code class="language-java">
  @Override
  public void store(Record record, Handler<AsyncResult<Record>> resultHandler) {
    String uuid = UUID.randomUUID().toString();
    if (record.getRegistration() != null) {
      throw new IllegalArgumentException("The record has already been registered");
    }

    record.setRegistration(uuid);
    registry.put(uuid, record.toJson().encode(), ar -> {
      if (ar.succeeded()) {
        resultHandler.handle(Future.succeededFuture(record));
      } else {
        resultHandler.handle(Future.failedFuture(ar.cause()));
      }
    });
  }
</code></pre>
**remove方法**
从AsyncMap中删除对应的ID
<pre class="line-numbers "><code class="language-java">
  @Override
  public void remove(String uuid, Handler<AsyncResult<Record>> resultHandler) {
    Objects.requireNonNull(uuid, "No registration id in the record");
    registry.remove(uuid, ar -> {
      if (ar.succeeded()) {
        if (ar.result() == null) {
          // Not found
          resultHandler.handle(Future.failedFuture("Record '" + uuid + "' not found"));
        } else {
          resultHandler.handle(Future.succeededFuture(
              new Record(new JsonObject(ar.result()))));
        }
      } else {
        resultHandler.handle(Future.failedFuture(ar.cause()));
      }
    });
  }
</code></pre>
## ServiceReference
所有服务类型的ServiceReference都可以继承自AbstractServiceReference。AbstractServiceReference提供了一个简单的缓存处理。ServiceReference未被释放时都从缓存中取.
<pre class="line-numbers "><code class="language-java">
	  public synchronized <X> X get() {
	    if (service == null) {
	      service = retrieve();
	    }
	    return cached();
	  }

	  public synchronized void release() {
	    ((DiscoveryImpl) discovery).unbind(this);
	    if (service != null) {
	      close();
	      service = null;
	    }
	  }
</code></pre>
它有一个retrieve的抽象方法用于获取每个服务类型对应的服务对象，需要各个子类实现
<pre class="line-numbers "><code class="language-java">
	protected abstract T retrieve();
</code></pre>
例如HTTP类型的实现如下
<pre class="line-numbers "><code class="language-java">
    public HttpClient retrieve() {
      HttpClientOptions options;
      if (config != null) {
        options = new HttpClientOptions(config);
      } else {
        options = new HttpClientOptions();
      }
      options.setDefaultPort(location.getPort()).setDefaultHost(location.getHost());
      if (location.isSsl()) {
        options.setSsl(true);
      }

      return vertx.createHttpClient(options);
    }
</code></pre>