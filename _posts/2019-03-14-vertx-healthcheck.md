---
layout: post
title: Vert.x健康检查
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-healthcheck.html
---

# HealthCheck

#　快速入门

	HealthChecks hc = HealthChecks.create(vertx);

	hc.register("my-procedure", future -> future.complete(Status.OK()));

	HealthCheckHandler healthCheckHandler = HealthCheckHandler.createWithHealthChecks(hc);

	Router router = Router.router(vertx);
	router.get("/ping").handler(healthCheckHandler);

	vertx.createHttpServer()
	.requestHandler(router::accept)
	.listen(8080);
	
上面对例子首先创建类一个HealthChecks对象，这个对象用来注册和注销应用．
接着创建一个HealthCheckHandler对象绑定到`/ping`这个地址上．

	curl localhost:8080/ping
	{"checks":[{"id":"my-procedure","status":"UP"}],"outcome":"UP"}
	
我们也可以通过HealthCheckHandler在任何时候来注册和注销应用

    HealthCheckHandler healthCheckHandler = HealthCheckHandler.create(vertx);

	Router router = Router.router(vertx);
	router.get("/health").handler(healthCheckHandler);

	healthCheckHandler.register("my-procedure", future -> {
	future.complete(Status.OK());
	});
	
测试

	curl localhost:8080/health
	{"checks":[{"id":"my-procedure","status":"UP"}],"outcome":"UP"}

## Procedure
Procedure是一个通过某些方面来推断当前系统是否健康的函数．它只报告一个状态(Status)，用来表明健康检查是成功还是失败，这个函数不能被阻塞．

注册一个Procedure时，我们需要提供一个名称，和一个执行健康检查的函数

	HealthChecks register(String name, Handler<Future<Status>> procedure)

- 如果future失败，健康状态被认为KO
- 如果future成功，但是没有任何状态，健康状态被认为OK
- 如果future成功，状态被标记为OK，健康状态被认为OK
- 如果future成功，状态被标记为KO，健康状态被认为KO

Status也可以包含一些额外的信息

	healthCheckHandler.register("my-procedure-name", future -> {
	future.complete(Status.OK(new JsonObject().put("available-memory", "2mb")));
	});

	healthCheckHandler.register("my-second-procedure-name", future -> {
	future.complete(Status.KO(new JsonObject().put("load", 99)));
	});

输出

	 curl localhost:8080/health
	{
	    "checks": [
		{
		    "id": "my-procedure-name", 
		    "status": "UP", 
		    "data": {
		        "available-memory": "2mb"
		    }
		}, 
		{
		    "id": "my-second-procedure-name", 
		    "status": "DOWN", 
		    "data": {
		        "load": 99
		    }
		}
	    ], 
	    "outcome": "DOWN"
	}	
	
Procedures也可以用组来划分，将procedure的名字按照树型结构构造

	curl localhost:8080/health
	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "DOWN", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "DOWN", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "DOWN"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "UP"
		        }
		    ]
		}
	    ], 
	    "outcome": "DOWN"
	}
	
将healthCheckHandler绑定到`router.get("/health/*")`地址上，还可以根据树形结构的目录来返回健康检查的结果

    router.get("/health").handler(healthCheckHandler);
    router.get("/health/*").handler(healthCheckHandler);

	curl localhost:8080/health/a-group/a-second-group
	{"checks":[{"id":"my-second-procedure-name","status":"DOWN"}],"outcome":"DOWN"}

## HTTP响应和JSON格式化
如果没有注册procedure，HTTP响应为204 - NO CONTENT,表明系统的状态是UP但是没有procedure被执行．响应不包含任何有效载荷．

	curl -i http://localhost:8080/health
	HTTP/1.1 204 No Content
	Content-Type: application/json;charset=UTF-8
	Content-Length: 0

如果注册了procedure，可能对响应码：

- 200 : Everything is fine
- 503 : At least one procedure has reported a non-healthy state
- 500 : One procedure has thrown an error or has not reported a status in time

200

	curl -i http://localhost:8080/health
	HTTP/1.1 200 OK
	Content-Type: application/json;charset=UTF-8
	Content-Length: 63

	{"checks":[{"id":"my-procedure","status":"UP"}],"outcome":"UP"}
	
	curl -i http://localhost:8080/health
	HTTP/1.1 200 OK
	Content-Type: application/json;charset=UTF-8
	Content-Length: 207

	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "UP", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "UP", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "UP"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "UP"
		        }
		    ]
		}
	    ], 
	    "outcome": "UP"
	}
	
503

	curl -i http://localhost:8080/health
	HTTP/1.1 503 Service Unavailable
	Content-Type: application/json;charset=UTF-8
	Content-Length: 215

	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "DOWN", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "DOWN", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "DOWN"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "UP"
		        }
		    ]
		}
	    ], 
	    "outcome": "DOWN"
	}

500

	curl -i http://localhost:8080/health
	HTTP/1.1 500 Internal Server Error
	Content-Type: application/json;charset=UTF-8
	Content-Length: 279

	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "DOWN", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "DOWN", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "DOWN"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "DOWN", 
		            "data": {
		                "procedure-execution-failure": true, 
		                "cause": "Timeout"
		            }
		        }
		    ]
		}
	    ], 
	    "outcome": "DOWN"
	}

outcome属性表明健康检查对总体结果UP或者DOWN

checks属性表明所有procedure对检查结果

如果一个procedure抛出一个异常，报告了一个异常，JSON结果会再data属性里说明异常信息

	curl -i http://localhost:8080/health
	HTTP/1.1 503 Service Unavailable
	Content-Type: application/json;charset=UTF-8
	Content-Length: 252

	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "DOWN", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "DOWN", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "DOWN"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "DOWN", 
		            "data": {
		                "cause": "undefined error"
		            }
		        }
		    ]
		}
	    ], 
	    "outcome": "DOWN"
	}

如果procedure在检查报告之前超时，会显示Timeout

	curl -i http://localhost:8080/health
	HTTP/1.1 500 Internal Server Error
	Content-Type: application/json;charset=UTF-8
	Content-Length: 279

	{
	    "checks": [
		{
		    "id": "a-group", 
		    "status": "DOWN", 
		    "checks": [
		        {
		            "id": "a-second-group", 
		            "status": "DOWN", 
		            "checks": [
		                {
		                    "id": "my-second-procedure-name", 
		                    "status": "DOWN"
		                }
		            ]
		        }, 
		        {
		            "id": "my-procedure-name", 
		            "status": "DOWN", 
		            "data": {
		                "procedure-execution-failure": true, 
		                "cause": "Timeout"
		            }
		        }
		    ]
		}
	    ], 
	    "outcome": "DOWN"
	}
	
## JDBC健康检查

	    healthCheckHandler.register("datasource", future -> {
	      jdbcClient.getConnection(ar -> {
		if (ar.succeeded()) {
		  ar.result().close();
		  future.complete(Status.OK());
		} else {
		  future.fail(ar.cause());
		}
	      });
	    });
	    
输出

	curl -i http://localhost:8080/health
	HTTP/1.1 200 OK
	Content-Type: application/json;charset=UTF-8
	Content-Length: 61

	{"checks":[{"id":"datasource","status":"UP"}],"outcome":"UP"}

## Service availability

	handler.register("my-service",
	  future -> HttpEndpoint.getClient(discovery,
	    (rec) -> "my-service".equals(rec.getName()),
	    client -> {
	      if (client.failed()) {
		future.fail(client.cause());
	      } else {
		client.result().close();
		future.complete(Status.OK());
	      }
	    }));

## Event bus

	handler.register("receiver",
	  future ->
	    vertx.eventBus().send("health", "ping", response -> {
	      if (response.succeeded()) {
		future.complete(Status.OK());
	      } else {
		future.complete(Status.KO());
	      }
	    })
	);
	
## Authentication安全验证
AuthProvider用来校验web请求的权限

## 将健康检查暴露到eventbus

	HealthChecks hc = HealthChecks.create(vertx);

	hc.register("my-procedure", future -> future.complete(Status.OK()));

	vertx.eventBus().consumer("health",
		message -> hc.invoke(message::reply));

# 原理
## Procedure

	public interface Procedure {

	  void check(Handler<JsonObject> resultHandler);

	}

Procedure的check方法用于检查应用的健康状态

## DefaultProcedure
它有三个属性

	  private final Handler<Future<Status>> handler;
	  private final String name;
	  private final long timeout;

- name 名称
- timeout 超时时间
- handler 用于执行健康检查的函数

check函数的实现

	  @Override
	  public void check(Handler<JsonObject> resultHandler) {
	    Future<Status> future = Future.<Status>future()
	      .setHandler(ar -> {
		if (ar.cause() instanceof ProcedureException) {
		  resultHandler.handle(StatusHelper.onError(name, (ProcedureException) ar.cause()));
		} else {
		  resultHandler.handle(StatusHelper.from(name, ar));
		}
	      });

	    if (timeout >= 0) {
	      vertx.setTimer(timeout, l -> {
		if (!future.isComplete()) {
		  future.fail(new ProcedureException("Timeout"));
		}
	      });
	    }

	    try {
	      handler.handle(future);
	    } catch (Exception e) {
	      future.fail(new ProcedureException(e));
	    }
	  }
	  
它主要分为三个部分

第一部分. 执行健康检查的函数

	    try {
	      handler.handle(future);
	    } catch (Exception e) {
	      future.fail(new ProcedureException(e));
	    }

handler就是在注册健康检查时声明的procedure

    hc.register("my-procedure", future -> future.complete(Status.OK()));
    
procedure将future标记为成功或失败，会触发future标记的回调函数（第二部分）

	    Future<Status> future = Future.<Status>future()
	      .setHandler(ar -> {
		if (ar.cause() instanceof ProcedureException) {
		  resultHandler.handle(StatusHelper.onError(name, (ProcedureException) ar.cause()));
		} else {
		  resultHandler.handle(StatusHelper.from(name, ar));
		}
	      });
	      
这个回调函数resultHandler的就是将检查结果作为http或者eventbus的响应输出

第三部分. 超时检查，如果future在规定对时间仍未完成，则会将future设置为失败(超时)

	    if (timeout >= 0) {
	      vertx.setTimer(timeout, l -> {
		if (!future.isComplete()) {
		  future.fail(new ProcedureException("Timeout"));
		}
	      });
	    }

## CompositeProcedure
CompositeProcedure就是树形结构的Procedure，DefaultCompositeProcedure是它的实现类。
DefaultCompositeProcedure内部使用了一个map来维护子节点

	private Map<String, Procedure> children = new HashMap<>();

它的check方法会依次调用子节点的check方法

    Map<String, Future<JsonObject>> tasks = new HashMap<>();
    List<Future> completed = new ArrayList<>();
    for (Map.Entry<String, Procedure> entry : copy.entrySet()) {
      Future<JsonObject> future = Future.future();
      completed.add(future);
      tasks.put(entry.getKey(), future);
      entry.getValue().check(future::complete);
    }

然后在通过CompositeFuture所有子节点的检查结果汇总到checks属性下面

    CompositeFuture.join(completed)
      .setHandler(ar -> {
        boolean success = true;
        for (Map.Entry<String, Future<JsonObject>> entry : tasks.entrySet()) {
          Future<JsonObject> json = entry.getValue();
          boolean up = isUp(json);
          success = success && up;

          JsonObject r = new JsonObject()
            .put("id", json.result().getString("id", entry.getKey()))
            .put("status", up ? "UP" : "DOWN");

          if (json.result() != null) {
            JsonObject data = json.result().getJsonObject("data");
            JsonArray children = json.result().getJsonArray("checks");
            if (data != null) {
              data.remove("result");
              r.put("data", data);
            } else if (children != null) {
              r.put("checks", children);
            }
          }

          checks.add(r);
        }

只有所有的检查结果都是UP，才会认为CompositeProcedure的检查结果是UP，然后调用回调函数resultHandler输出检查结果。

	boolean success = true;
	boolean up = isUp(json);
	success = success && up;
	
	if (success) {
	  result.put("outcome", "UP");
	} else {
	  result.put("outcome", "DOWN");
	}
	
	resultHandler.handle(result);

## HealthChecks
HealthChecks用来注册或者卸载一个健康检查的Procedure，它也提供了一个invoke方法用来触发某个具体节点的健康检查。

注册Procedure的方法如下，它会将注册的名称`name`按照`/`来分割，创建一个树形结构的CompositeProcedure。**健康检查的超时时间设置为1秒，所以每个检查检查的函数不应该执行阻塞方法。**

	  @Override
	  public HealthChecks register(String name, Handler<Future<Status>> procedure) {
	    Objects.requireNonNull(name);
	    if (name.isEmpty()) {
	      throw new IllegalArgumentException("The name must not be empty");
	    }
	    Objects.requireNonNull(procedure);
	    String[] segments = name.split("/");
	    CompositeProcedure parent = traverseAndCreate(segments);
	    String lastSegment = segments[segments.length - 1];
	    parent.add(lastSegment,
	      new DefaultProcedure(vertx, lastSegment, 1000, procedure));
	    return this;
	  }
	
	  private CompositeProcedure traverseAndCreate(String[] segments) {
	    int i;
	    CompositeProcedure parent = root;
	    for (i = 0; i < segments.length - 1; i++) {
	      Procedure c = parent.get(segments[i]);
	      if (c == null) {
	        DefaultCompositeProcedure composite = new DefaultCompositeProcedure();
	        parent.add(segments[i], composite);
	        parent = composite;
	      } else if (c instanceof CompositeProcedure) {
	        parent = (CompositeProcedure) c;
	      } else {
	        // Illegal.
	        throw new IllegalArgumentException("Unable to find the procedure `" + segments[i] + "`, `"
	          + segments[i] + "` is not a composite.");
	      }
	    }
	
	    return parent;
	  }

注销Procedure，只是从树形结构中删除了对应的节点和它的子孙节点

	  @Override
	  public HealthChecks unregister(String name) {
	    Objects.requireNonNull(name);
	    if (name.isEmpty()) {
	      throw new IllegalArgumentException("The name must not be empty");
	    }
	
	    String[] segments = name.split("/");
	    CompositeProcedure parent = findLastParent(segments);
	    if (parent != null) {
	      String lastSegment = segments[segments.length - 1];
	      parent.remove(lastSegment);
	    }
	    return this;
	  }

invoke的逻辑很简单，主要由两部分组成：

1. 根据名称找到对应的节点
2. 执行该节点的check方法即

	  @Override
	  public HealthChecks invoke(Handler<JsonObject> resultHandler) {
	    compute(root, resultHandler);
	    return this;
	  }
	
	  @Override
	  public HealthChecks invoke(String name, Handler<AsyncResult<JsonObject>> resultHandler) {
	    if (name == null || name.isEmpty() || name.equals("/")) {
	      return invoke(json -> resultHandler.handle(Future.succeededFuture(json)));
	    } else {
	      String[] segments = name.split("/");
	      Procedure check = root;
	      for (String segment : segments) {
	        if (segment.trim().isEmpty()) {
	          continue;
	        }
	        if (check instanceof CompositeProcedure) {
	          check = ((CompositeProcedure) check).get(segment);
	          if (check == null) {
	            // Not found
	            resultHandler.handle(Future.failedFuture("Not found"));
	            return this;
	          }
	          // Else continue...
	        } else {
	          // Not a composite
	          resultHandler.handle(Future.failedFuture("'" + segment + "' is not a composite"));
	          return this;
	        }
	      }
	
	      if (check == null) {
	        resultHandler.handle(null);
	        return this;
	      }
	      compute(check, json -> resultHandler.handle(Future.succeededFuture(json)));
	    }
	    return this;
	  }

## HealthCheckHandler
HealthCheckHandler用来接收一个HTTP请求，执行健康检查，并将结果响应。它的逻辑比较简单。

首先每个HealthCheckHandler都应该与一个HealthChecks绑定

	  public HealthCheckHandlerImpl(Vertx vertx, AuthProvider provider) {
	    this.healthChecks = new HealthChecksImpl(vertx);
	    this.authProvider = provider;
	  }
	
	  public HealthCheckHandlerImpl(HealthChecks hc, AuthProvider provider) {
	    this.healthChecks = Objects.requireNonNull(hc);
	    this.authProvider = provider;
	  }
	
	  @Override
	  public HealthCheckHandler register(String name, Handler<Future<Status>> procedure) {
	    healthChecks.register(name, procedure);
	    return this;
	  }

HTTP请求调用handle方法的时候会触发绑定的HealthChecks执行健康检查。在执行健康检查之前也可以用AuthProvider对请求的做安全校验。

	  @Override
	  public void handle(RoutingContext rc) {
	    String id = rc.request().path().substring(rc.currentRoute().getPath().length());
	    if (authProvider != null) {
	      // Copy all HTTP header in a json array and params
	      JsonObject authData = new JsonObject();
	      rc.request().headers().forEach(entry -> authData.put(entry.getKey(), entry.getValue()));
	      rc.request().params().forEach(entry -> authData.put(entry.getKey(), entry.getValue()));
	      if (rc.request().method() == HttpMethod.POST
	        && rc.request()
	        .getHeader(HttpHeaders.CONTENT_TYPE).contains("application/json")) {
	        authData.mergeIn(rc.getBodyAsJson());
	      }
	      authProvider.authenticate(authData, ar -> {
	        if (ar.failed()) {
	          rc.response().setStatusCode(403).end();
	        } else {
	          healthChecks.invoke(id, healthReportHandler(rc));
	        }
	      });
	    } else {
	      healthChecks.invoke(id, healthReportHandler(rc));
	    }
	  }

healthReportHandler方法主要是根据HealthChecks返回健康检查结果，输出不同的响应到调用方。

