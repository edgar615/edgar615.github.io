---
layout: post
title: Vert.x service proxy
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-service-proxy.html
---

# Vert.x Service Proxy
Vert.x Service Proxy是Vert.x用于进行异步RPC通信的组件．它基于Event Bus实现，它会自动生成代理类进行消息的包装与解码、发送与接收以及超时处理.

# 接口定义

	@ProxyGen
	public interface UserService {

	  void save(JsonObject jsonObject, Handler<AsyncResult<JsonObject>> handler);

	  void get(Integer userId, Handler<AsyncResult<JsonObject>> handler);
	}

在服务接口上加上@ProxyGen注解，Vert.x就会自动为生成相应的代理类用于在底层处理RPC.
接口的方法上都有一个回调函数<code>Handler<AsyncResult<JsonObject>> handler</code>，在调用结果发送过来的时候会自动调用绑定的回调函数进行相关的处理.

# 自动生成代理类
**package-info.java**

	@ModuleGen(name = "vertx-rpc-userservice", groupPackage = "com.edgar.rpc.vertx")
	package com.edgar.rpc.vertx.core;

	import io.vertx.codegen.annotations.ModuleGen;

自动生成的代码必须属于某个模块：一个java包注解＠ＭoduleＧen用来定义模块。这些文件是在文件中创建package-info.java.

- name属性定义了在生成其他不遵循java包命名规则的语言时所使用的包名，如JavaScript和Ruby.
- groupPackage属性定义了用于生成的包名，它必须和接口所在包相同或者是它的父节点

**在pom.xml中增加插件**

	<plugin>
		<artifactId>maven-compiler-plugin</artifactId>
		<version>3.1</version>
		<configuration>
		    <source>1.8</source>
		    <target>1.8</target>
		    <annotationProcessors>
		        <annotationProcessor>io.vertx.codegen.CodeGenProcessor</annotationProcessor>
		    </annotationProcessors>
		    <generatedSourcesDirectory>
		        ${project.basedir}/src/main/generated
		    </generatedSourcesDirectory>
		    <compilerArgs>
		        <arg>-AoutputDirectory=${project.basedir}/src/main</arg>
		    </compilerArgs>
		</configuration>
	 </plugin>

运行mvn complie就会在src/main/generated生成两个文件：
- 服务代理类UserServiceVertxEBProxy，通过eventbus与服务提供者进行通信.
- 服务处理程序UserServiceVertxProxyHandler，通过eventbus与服务代理类进行通信，它继承ProxyHandler，并实现了<code>MessageConsumer<JsonObject> registerHandler(String address)</code>用于将自己注册到eventbus上．

# 服务提供者
**服务实现类**

	public class UserServiceImpl implements UserService {
	  @Override
	  public void save(JsonObject jsonObject, Handler<AsyncResult<JsonObject>> handler) {
	    if (!jsonObject.containsKey("username")) {
	      handler.handle(ServiceException.fail(400, "invalid arguments"));
	    } else {
	      handler.handle(Future.succeededFuture(jsonObject.copy()));
	    }
	  }

	  @Override
	  public void get(Integer userId, Handler<AsyncResult<JsonObject>> handler) {
	    if (userId <= 0) {
	      handler.handle(Future.failedFuture("404 not found"));
	    } else {
	      handler.handle(Future.succeededFuture(new JsonObject()
		  .put("userId", userId)
		  .put("username", "edgar")));
	    }
	  }
	}

**注册服务提供者**

	public class ProducerVerticle extends AbstractVerticle {

	  public static void main(String[] args) {
	    new Launcher().execute("run", ProducerVerticle.class.getName(), "--cluster");
	  }

	  @Override
	  public void start() throws Exception {
	    UserService userService = new UserServiceImpl();
	    ProxyHelper.registerService(UserService.class, vertx, userService, "rpc.demo.userservice");
	  }
	}

ProxyHelper.registerService可以将服务接口注册到eventbus上，它的实现很简单，它会先通过反射创建服务处理程序UserServiceVertxProxyHandler的一个实例，然后将这个示例绑定到指定对地址上，源码如下：

	  public static <T> MessageConsumer<JsonObject> registerService(Class<T> clazz, Vertx vertx, T service, String address,
		                                                        boolean topLevel,
		                                                        long timeoutSeconds) {
	    String handlerClassName = clazz.getName() + "VertxProxyHandler";
	    Class<?> handlerClass = loadClass(handlerClassName, clazz);
	    Constructor constructor = getConstructor(handlerClass, Vertx.class, clazz, boolean.class, long.class);
	    Object instance = createInstance(constructor, vertx, service, topLevel, timeoutSeconds);
	    ProxyHandler handler = (ProxyHandler) instance;
	    return handler.registerHandler(address);
	  }
 
# 服务调用者
  
	  public class ConsumerVerticle extends AbstractVerticle {

	  public static void main(String[] args) {
	    new Launcher().execute("run", ConsumerVerticle.class.getName(), "--cluster");
	  }

	  @Override
	  public void start() throws Exception {
	    UserService userService = ProxyHelper.createProxy(UserService.class, vertx, "rpc.demo.userservice");
	    userService.get(1, ar -> {
	      if (ar.succeeded()) {
		System.out.println(ar.result());
	      } else {
		System.err.println(ar.cause().getMessage());
	      }
	    });
	  }
	}
ProxyHelper.createProxy  方法可以创建一个服务代理类，它的实现也很简单，它会通过反射创建服务代理类UserServiceVertxEBProxy的一个实例，源码如下：

	  public static <T> T createProxy(Class<T> clazz, Vertx vertx, String address, DeliveryOptions options) {
	    String proxyClassName = clazz.getName() + "VertxEBProxy";
	    Class<?> proxyClass = loadClass(proxyClassName, clazz);
	    Constructor constructor;
	    Object instance;
	    if (options == null) {
	      constructor = getConstructor(proxyClass, Vertx.class, String.class);
	      instance = createInstance(constructor, vertx, address);
	    } else {
	      constructor = getConstructor(proxyClass, Vertx.class, String.class, DeliveryOptions.class);
	      instance = createInstance(constructor, vertx, address, options);
	    }
	    return (T) instance;
	  }

# 调用逻辑
当服务调用者发送userService.get请求时，服务代理类会将参数封装在JsonObject中

	JsonObject _json = new JsonObject();
	    _json.put("userId", userId);
同时将方法名放在Eventbus对请求头里

	_deliveryOptions.addHeader("action", "get")
然后发起一个请求/响应类型的事件.

服务处理程序再收到事件之后会解析参数和方法

	JsonObject json = msg.body();
	String action = msg.headers().get("action");
然后再调用真正的服务方法，并调用对应的回调函数.

      switch (action) {
        case "save": {
          service.save((io.vertx.core.json.JsonObject)json.getValue("jsonObject"), createHandler(msg));
          break;
        }
        case "get": {
          service.get(json.getValue("userId") == null ? null : (json.getLong("userId").intValue()), createHandler(msg));
          break;
        }
        default: {
          throw new IllegalStateException("Invalid action: " + action);
        }
      }

服务代理类会封装对消息