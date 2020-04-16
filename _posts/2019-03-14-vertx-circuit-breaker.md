---
layout: post
title: Vert.x的断路器
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-circuit-breaker.html
---

# Vert.x的断路器

## 创建断路器

        CircuitBreaker breaker = CircuitBreaker.create("my-circuit-breaker", vertx,
                new CircuitBreakerOptions()
                        .setMaxFailures(5) // number of failure before opening the circuit
                        .setTimeout(1000) // consider a failure if the operation does not succeed in time
                        .setResetTimeout(3000) // time spent in open state before attempting to re-try
        )
                .openHandler(v -> {
                    System.out.println("Circuit opened");
                }).closeHandler(v -> {
                    System.out.println("Circuit closed");
                }).halfOpenHandler(v -> {
                    System.out.println("reset (half-open state)");
                });;

- setMaxFailures方法用来定义一个错误次数，断路器execute方法的错误次数到达这个错误次数之后，断路器会被开启
- setTimeout方法用来定义超时时间，如果断路器execute方法超过这个时间仍未完成，断路器会认为这个方法超时
- setResetTimeout方法用来定义断路器从开启状态到半开启状态的时间

我们通过一个定时器来测试一下上面创建的断路器

    AtomicInteger seq = new AtomicInteger();
    vertx.setPeriodic(1000, l -> {
      final long time = System.currentTimeMillis();
      final int i = seq.incrementAndGet();
      breaker.<String>execute(future -> {
        //do nothing
      }).setHandler(ar -> {
        if (ar.succeeded()) {
          System.out.println(time + " : OK: "
                             + ar.result() + " " + i);
        } else {
          System.out.println(
                  time + " : ERROR: " + ar.cause() + " " + i);
        }
      });
    });

输出

	1493714248939 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 1
	1493714249939 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 2
	1493714250939 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 3
	1493714251939 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 4
	1493714253939 : Circuit opened
	1493714252939 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 5
	1493714254939 : ERROR: java.lang.RuntimeException: open circuit 7
	1493714253938 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 6
	1493714255938 : ERROR: java.lang.RuntimeException: open circuit 8
	1493714256938 : ERROR: java.lang.RuntimeException: open circuit 9
	1493714256941 : reset (half-open state)
	1493714258939 : ERROR: java.lang.RuntimeException: open circuit 11
	1493714258941 : Circuit opened
	1493714257938 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 10
	1493714259939 : ERROR: java.lang.RuntimeException: open circuit 12
	1493714260939 : ERROR: java.lang.RuntimeException: open circuit 13
	1493714261939 : ERROR: java.lang.RuntimeException: open circuit 14
	1493714261942 : reset (half-open state)
	1493714263939 : ERROR: java.lang.RuntimeException: open circuit 16
	1493714263939 : Circuit opened

观察输出，可以看到，在第五个请求超时后，断路器被打开；

	1493714253939 : Circuit opened

在断路器打开之后的请求，全部直接返回的RuntimeException，而不再是超时的错误

在3秒后，断路器处于半开状态，

	1493714256941 : reset (half-open state)

这之后的第一个请求通过了断路器

	1493714257938 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 10

他在超时之后又重新将断路器打开

我们将断路器的逻辑做一点修改

	  breaker.<String>execute(future -> {
	    //do nothing
	    if (i >= 12) {
	      future.complete();
	    }
	  })

输出

	1493714640143 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 1
	1493714641145 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 2
	1493714642143 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 3
	1493714643145 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 4
	1493714645144 : Circuit opened
	1493714644144 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 5
	1493714646144 : ERROR: java.lang.RuntimeException: open circuit 7
	1493714645144 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 6
	1493714647144 : ERROR: java.lang.RuntimeException: open circuit 8
	1493714648144 : ERROR: java.lang.RuntimeException: open circuit 9
	1493714648145 : reset (half-open state)
	1493714650144 : ERROR: java.lang.RuntimeException: open circuit 11
	1493714650146 : Circuit opened
	1493714649144 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 10
	1493714651144 : ERROR: java.lang.RuntimeException: open circuit 12
	1493714652144 : ERROR: java.lang.RuntimeException: open circuit 13
	1493714653143 : ERROR: java.lang.RuntimeException: open circuit 14
	1493714653147 : reset (half-open state)
	1493714654145 : Circuit closed
	1493714654144 : OK: null 15
	1493714655143 : OK: null 16
	1493714656144 : OK: null 17
	1493714657144 : OK: null 18
	1493714658143 : OK: null 19

虽然我们设置了从第12个请求开始，所有的请求都成功，但是由于在第10个请求时，断路器被重新开启，所以第12，13，14个请求都直接由断路器返回了失败，直到第15个请求，断路器才关闭

## 断路器开启时的默认值

     breaker.<String>executeWithFallback(future -> {
        vertx.createHttpClient().getNow(8080, "localhost", "/", response -> {
          if (response.statusCode() != 200) {
            future.fail("HTTP error");
          } else {
            response
                    .exceptionHandler(future::fail)
                    .bodyHandler(buffer -> {
                      future.complete(buffer.toString());
                    });
          }
        });
      }, r -> "Hello").setHandler(ar -> {
        if (ar.succeeded()) {
          System.out.println(ar.result());
        } else {
          System.out.println(ar.cause());
        }
      });

在断路器开启之后，会直接通过`r -> "Hello"`返回

输出

	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	Circuit opened
	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	Hello
	Hello
	Hello
	reset (half-open state)
	Circuit opened
	io.vertx.core.impl.NoStackTraceThrowable: HTTP error
	Hello

也可以在创建断路器时指定fallback

    CircuitBreaker breaker = CircuitBreaker.create("my-circuit-breaker", vertx,
                                                   new CircuitBreakerOptions()
                                                           .setMaxFailures(5)
                                                           .setTimeout(1000)
                                                           .setResetTimeout(3000)
    ).fallback(t -> "HELLO")

# Retry
通过setMaxRetries方法设置断路器的重试次数

    CircuitBreaker breaker = CircuitBreaker.create("my-circuit-breaker", vertx,
                                                   new CircuitBreakerOptions()
                                                           .setMaxFailures(5)
                                                           .setTimeout(1000)
                                                           .setResetTimeout(3000)
                                                           .setMaxRetries(3)
    )

输出

	1493716673245 execute:1
	1493716674234 execute:2
	1493716674248 execute:1
	1493716675232 execute:3
	1493716675235 execute:2
	1493716675249 execute:1
	1493716676238 execute:4
	1493716676238 execute:3
	1493716676238 execute:2
	1493716673232 : ERROR: io.vertx.core.impl.NoStackTraceThrowable: operation timeout 1

# Eventbus通知

	  CircuitBreaker breaker = CircuitBreaker.create("my-circuit-breaker", vertx,
	                                                   new CircuitBreakerOptions()
	                                                           .setMaxFailures(5)
	                                                           .setTimeout(2000)
	                                                           .setNotificationAddress(
	                                                                   "vertx.circuit-breaker")
	    );
	
	    vertx.eventBus().consumer("vertx.circuit-breaker", msg -> {
	      System.out.println(msg.body());
	    });

每个通知的事件包括下列属性

- state : 断路器的状态 (OPEN, CLOSED, HALF_OPEN)
- name : 断路器的名称
- failures : 失败次数
- node : 节点标识符 (单机模式下为local)
- resetTimeout、timeout、metricRollingWindow等属性

# Hystrix集成
通过HystrixMetricHandler可以将断路器的状态上报到Hystrix中

如果想使用Hystrix的断路器，可以通过executeBlocking来实现

	HystrixCommand<String> someCommand = getSomeCommandInstance();
	vertx.<String>executeBlocking(
	future -> future.complete(someCommand.execute()),
	ar -> {
	// back on the event loop
	String result = ar.result();
	}
	);

或者

	vertx.runOnContext(v -> {
	Context context = vertx.getOrCreateContext();
	HystrixCommand<String> command = getSomeCommandInstance();
	command.observe().subscribe(result -> {
	context.runOnContext(v2 -> {
	// Back on context (event loop or worker)
	String r = result;
	});
	});
	});

# 实现

CircuitBreaker的创建很简单，就是做了一下基本的参数设置，并用了一个定时器想eventbus广播消息

	  public CircuitBreakerImpl(String name, Vertx vertx, CircuitBreakerOptions options) {
	    Objects.requireNonNull(name);
	    Objects.requireNonNull(vertx);
	    this.vertx = vertx;
	    this.name = name;
	
	    if (options == null) {
	      this.options = new CircuitBreakerOptions();
	    } else {
	      this.options = new CircuitBreakerOptions(options);
	    }
	
	    this.metrics = new CircuitBreakerMetrics(vertx, this, options);
	
	    sendUpdateOnEventBus();
	
	    if (this.options.getNotificationPeriod() > 0) {
	      this.periodicUpdateTask = vertx.setPeriodic(this.options.getNotificationPeriod(), l -> sendUpdateOnEventBus());
	    } else {
	      this.periodicUpdateTask = -1;
	    }
	  }
CircuitBreaker会有一个状态属性，两个计数器：

	  private CircuitBreakerState state = CircuitBreakerState.CLOSED;
	  private long failures = 0;//失败次数
	  private final AtomicInteger passed = new AtomicInteger(); //半开状态下的通过次数

断路器的execute和executeAndReport都是通过executeAndReportWithFallback来实现，这个方法首先会查询断路器的状态：

    CircuitBreakerState currentState;
    synchronized (this) {
      currentState = state;
    }

然后根据状态来执行不同的逻辑

## 关闭

如果断路器是关闭状态，则执行用户的请求

    if (currentState == CircuitBreakerState.CLOSED) {
      if (options.getMaxRetries() > 0) {
        executeOperation(context, command, retryFuture(context, 1, command, operationResult, call), call);
      } else {
        executeOperation(context, command, operationResult, call);
      }
    } 

executeOperation方法，会直接执行用户的方法，并在方法执行完成后修改operationResult的完成状态

    try {
      // We use an intermediate future to avoid the passed future to complete or fail after a timeout.
      Future<T> passedFuture = Future.future();
      passedFuture.setHandler(ar -> {
        context.runOnContext(v -> {
          if (ar.failed()) {
            if (!operationResult.isComplete()) {
              operationResult.fail(ar.cause());
            }
          } else {
            if (!operationResult.isComplete()) {
              operationResult.complete(ar.result());
            }
          }
        });
      });
    
      operation.handle(passedFuture);
    } catch (Throwable e) {
      context.runOnContext(v -> {
        if (!operationResult.isComplete()) {
          if (call != null) {
            call.error();
          }
          operationResult.fail(e);
        }
      });
    }

同时，executeOperation会开启一个定时器，如果在规定的时间operationResult并没有完成，则经operationResult设为超时失败

    if (options.getTimeout() != -1) {
      vertx.setTimer(options.getTimeout(), (l) -> {
        context.runOnContext(v -> {
          // Check if the operation has not already been completed
          if (!operationResult.isComplete()) {
            if (call != null) {
              call.timeout();
            }
            operationResult.fail("operation timeout");
          }
          // Else  Operation has completed
        });
      });
    }

如果断路器开启了重试机制，会将operationResult封装为一个新的Future，只有在断路器的状态是关闭时才会执行重试，其他状态直接返回失败

在operationResult标记为完成之后,会根据返回的结果更新断路器的状态

    operationResult.setHandler(event -> {
      context.runOnContext(v -> {
        if (event.failed()) {
          incrementFailures();
          call.failed();
          if (options.isFallbackOnFailure()) {
            invokeFallback(event.cause(), userFuture, fallback, call);
          } else {
            userFuture.fail(event.cause());
          }
        } else {
          call.complete();
          reset();
          userFuture.complete(event.result());
        }
        // Else the operation has been canceled because of a time out.
      });
    
    });

我们先看成功的操作，成功之后会将userFuture标记为成功

	call.complete();
	reset();
	userFuture.complete(event.result());

并通过reset方法重置断路器的状态：将failures重置为0，断路器状态重置为关闭.

	public synchronized CircuitBreaker reset() {
		failures = 0;
		
		if (state == CircuitBreakerState.CLOSED) {
		  // Do nothing else.
		  return this;
		}
		
		state = CircuitBreakerState.CLOSED;
		closeHandler.handle(null);
		sendUpdateOnEventBus();
		return this;
	}

再来看失败后的操作，失败之后会通过检查用户是否设置了isFallbackOnFailure，如果没有设置，直接返回将userFuture标记为失败，如果有设置，通过invokeFallback方法重新标记userFuture

	incrementFailures();
	call.failed();
	if (options.isFallbackOnFailure()) {
	invokeFallback(event.cause(), userFuture, fallback, call);
	} else {
	userFuture.fail(event.cause());
	}

通过incrementFailures方法，计算断路器的状态：失败次数加1，如果失败次数大于等于最大次数，将断路器的状态设置为开启(open方法)

	private synchronized void incrementFailures() {
		failures++;
		if (failures >= options.getMaxFailures()) {
		  if (state != CircuitBreakerState.OPEN) {
		    open();
		  } else {
		    // No need to do it in the previous case, open() do it.
		    // If open has been called, no need to send update, it will be done by the `open` method.
		    sendUpdateOnEventBus();
		  }
		} else {
		  // Number of failure has changed, send update.
		  sendUpdateOnEventBus();
		}
	}

再来看一下open方法：除了将状态修改为开启外，它还会开启一个定时器，通过attemptReset方法尝试将状态修改为半开状态

	public synchronized CircuitBreaker open() {
		state = CircuitBreakerState.OPEN;
		openHandler.handle(null);
		sendUpdateOnEventBus();
		
		// Set up the attempt reset timer
		long period = options.getResetTimeout();
		if (period != -1) {
		  vertx.setTimer(period, l -> attemptReset());
		}
		
		return this;
	}

attemptReset方法尝试将状态修改为半开状态,在修改为半开状态后，会重置passed计数器为0，用于半开状态的判断

	private synchronized CircuitBreaker attemptReset() {
		if (state == CircuitBreakerState.OPEN) {
		  passed.set(0);
		  state = CircuitBreakerState.HALF_OPEN;
		  halfOpenHandler.handle(null);
		  sendUpdateOnEventBus();
		}
		return this;
	}


## 开启

如果是开启状态，则执行fallback或者抛出异常

	else if (currentState == CircuitBreakerState.OPEN) {
	  // Fallback immediately
	  call.shortCircuited();
	  invokeFallback(new RuntimeException("open circuit"), userFuture, fallback, call);
	} 

如果是半开状态，断路器会检查passed属性，如果passed不是0，那么直接执行fallback或者抛出异常.
如果passed是0,那么会自增加1，然后让程序通过。如果这个程序依然返回失败，就会重新将断路器的状态设置为打开；如果程序返回成功，这会通过reset方法重置断路器状态

	 else if (currentState == CircuitBreakerState.HALF_OPEN) {
	  if (passed.incrementAndGet() == 1) {
	    operationResult.setHandler(event -> {
	      if (event.failed()) {
	        open();
	        if (options.isFallbackOnFailure()) {
	          invokeFallback(event.cause(), userFuture, fallback, call);
	        } else {
	          userFuture.fail(event.cause());
	        }
	      } else {
	        reset();
	        userFuture.complete(event.result());
	      }
	    });
	    // Execute the operation
	    executeOperation(context, command, operationResult, call);
	  } else {
	    // Not selected, fallback.
	    call.shortCircuited();
	    invokeFallback(new RuntimeException("open circuit"), userFuture, fallback, call);
	  }
	}