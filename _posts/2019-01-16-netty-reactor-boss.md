---
layout: post
title: Netty - 主Reactor执行主流程
date: 2019-01-15
categories:
    - netty
comments: true
permalink: netty-reactor-boss.html
---

ServerBootstrap在注册Channel时，会通过AbstractChannel#register方法注册channel

```
if (eventLoop.inEventLoop()) {
    register0(promise);
} else {
    try {
        eventLoop.execute(new Runnable() {
            @Override
            public void run() {
                register0(promise);
            }
        });
    } catch (Throwable t) {
		...
    }
}
```

此时我们的线程是Main线程，所以它会执行eventLoop#execute方法来注册Channel，而这个方法的实现是在SingleThreadEventExecutor。

```
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
		...
    }
}

private void startThread() {
	if (state == ST_NOT_STARTED) {
		if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
			...
			doStartThread();
			...
		}
	}
}

private void doStartThread() {
	...
	executor.execute(new Runnable() {
		@Override
		public void run() {
			...
			try {
				// 重要的地方
				SingleThreadEventExecutor.this.run();
				success = true;
			} catch (Throwable t) {
				logger.warn("Unexpected exception from an event executor: ", t);
			} finally {
				...
			}
		}
	});
}
```

可以看到注册Channel的时候，SingleThreadEventExecutor才正式启动了线程池，当我们进入NioEventLoop run()方法后，可以发现它是一个无限循环，没有任何退出条件。

```
@Override
protected void run() {
	int selectCnt = 0;
	for (;;) {
		try {
			int strategy;
			try {
				strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
				switch (strategy) {
				case SelectStrategy.CONTINUE:
					continue;
				case SelectStrategy.BUSY_WAIT:
				case SelectStrategy.SELECT: // 轮询 I/O 事件
					long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
					if (curDeadlineNanos == -1L) {
						curDeadlineNanos = NONE; // nothing on the calendar
					}
					nextWakeupNanos.set(curDeadlineNanos);
					try {
						if (!hasTasks()) {
							strategy = select(curDeadlineNanos);
						}
					} finally {
						nextWakeupNanos.lazySet(AWAKE);
					}
					// fall through
				default:
				}
			} catch (IOException e) {
				...
			}

			selectCnt++;
			cancelledKeys = 0;
			needsToSelectAgain = false;
			final int ioRatio = this.ioRatio;
			boolean ranTasks;
			if (ioRatio == 100) {
				try {
					if (strategy > 0) {
						// 处理 I/O 事件
						processSelectedKeys();
					}
				} finally {
					// 处理所有任务
					ranTasks = runAllTasks();
				}
			} else if (strategy > 0) {
				final long ioStartTime = System.nanoTime();
				try {
					// 处理 I/O 事件
					processSelectedKeys();
				} finally {
					// // 处理完 I/O 事件，再处理异步任务队列
					final long ioTime = System.nanoTime() - ioStartTime;
					ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
				}
			} else {
				ranTasks = runAllTasks(0); // This will run the minimum number of tasks
			}

			...
		} catch (e) {
			...
		} finally {
			...
		}
	}
}

```

上述的代码在不间断循环执行以下三件事情，

- 轮询 I/O 事件（select）：轮询 Selector 选择器中已经注册的所有 Channel 的 I/O 事件。

- 处理 I/O 事件（processSelectedKeys）：处理已经准备就绪的 I/O 事件。

- 处理异步任务队列（runAllTasks）：Reactor 线程还有一个非常重要的职责，就是处理任务队列中的非 I/O 任务。Netty 提供了 ioRatio 参数用于调整 I/O 事件处理和任务处理的时间比例。

![](/assets/images/posts/netty-eventloop/netty-eventloop-4.png)

# 轮询 I/O 事件

```
switch (strategy) {
case SelectStrategy.CONTINUE:
	continue;
case SelectStrategy.BUSY_WAIT:
case SelectStrategy.SELECT: // 轮询 I/O 事件
	long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
	if (curDeadlineNanos == -1L) {
		curDeadlineNanos = NONE; // nothing on the calendar
	}
	nextWakeupNanos.set(curDeadlineNanos);
	try {
		if (!hasTasks()) {
			strategy = select(curDeadlineNanos);
		}
	} finally {
		// This update is just to help block unnecessary selector wakeups
        // so use of lazySet is ok (no race condition)
		nextWakeupNanos.lazySet(AWAKE);
	}
	// fall through
default:
}
```

NioEventLoop 通过核心方法 select() 不断轮询注册的 I/O 事件。当没有 I/O 事件产生时，为了避免 NioEventLoop 线程一直循环空转，在获取 I/O 事件或者异步任务时需要阻塞线程，等待 I/O 事件就绪或者异步任务产生后才唤醒线程。NioEventLoop通过nextWakeupNanos来判断唤醒逻辑

```
// nextWakeupNanos is:
//    AWAKE            when EL is awake
//    NONE             when EL is waiting with no wakeup scheduled
//    other value T    when EL is waiting with wakeup scheduled at time T
private final AtomicLong nextWakeupNanos = new AtomicLong(AWAKE);
```

Netty 提供了选择策略 SelectStrategy 对象，它用于控制 select 循环行为，包含 CONTINUE、SELECT、BUSY_WAIT 三种策略，因为 NIO 并不支持 BUSY_WAIT，所以 BUSY_WAIT 与 SELECT 的执行逻辑是一样的。

SelectStrategy是如何判断的？

```
final class DefaultSelectStrategy implements SelectStrategy {
    @Override
    public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
        return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
    }
}

// NioEventLoop 
private final IntSupplier selectNowSupplier = new IntSupplier() {
	public int get() throws Exception {
		return NioEventLoop.this.selectNow();
	}
};

int selectNow() throws IOException {
	return this.selector.selectNow();
}
```

如果当前 NioEventLoop 线程存在异步任务，会通过 selectSupplier.get() 最终调用到 selectNow() 方法，selectNow() 是非阻塞，执行后立即返回。如果存在就绪的 I/O 事件，那么会走到 default 分支后直接跳出，然后执行 I/O 事件处理 processSelectedKeys 和异步任务队列处理 runAllTasks 的逻辑。**所以在存在异步任务的场景，NioEventLoop 会优先保证 CPU 能够及时处理异步任务。**

当 NioEventLoop 线程的不存在异步任务，即任务队列为空，返回的是 SELECT 策略, 就会调用select方法

```
private int select(long deadlineNanos) throws IOException {
    if (deadlineNanos == NONE) {
        return selector.select();
    }
    // Timeout will only be 0 if deadline is within 5 microsecs
    // 判断是否超过5微妙，如果超过说明当前任务队列中有定时任务需要立刻执行，执行延时select，否则就执行selectNow()
    long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
    return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
}
```

# 处理 I/O 事件

通过 select 过程我们已经获取到准备就绪的 I/O 事件，接下来就需要调用 processSelectedKeys() 方法处理 I/O 事件。在开始处理 I/O 事件之前，**Netty 通过 ioRatio 参数控制 I/O 事件处理和任务处理的时间比例**，默认为 ioRatio = 50。如果 ioRatio = 100，表示每次都处理完 I/O 事件后，会执行所有的 task。如果 ioRatio < 100，也会优先处理完 I/O 事件，再处理异步任务队列。所以不论如何 processSelectedKeys() 都是先执行的：

```
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

处理 I/O 事件时有两种选择，一种是处理 Netty 优化过的 selectedKeys，另外一种是正常的处理逻辑。根据是否设置了 selectedKeys 来判断使用哪种策略，这两种策略使用的 selectedKeys 集合是不一样的。Netty 优化过的 selectedKeys 是 SelectedSelectionKeySet 类型，而正常逻辑使用的是 JDK HashSet 类型。

**processSelectedKeysPlain**

```
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
	...
    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        final SelectionKey k = i.next();
        // SelectionKey 上面可以挂载 attachment
        final Object a = k.attachment();
        i.remove();
		//根据 attachment 属性可以判断 SelectionKey 的类型,AbstractNioChannel 类型由 Netty 框架负责处理
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            // NioTask 是用户自定义的 task
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
		...
    }
}
```

**processSelectedKey**

```
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    // 检查 Key 是否合法
    if (!k.isValid()) {
        final EventLoop eventLoop;
		...
		
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            // key不合法，关闭连接
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // 处理连接事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // 处理可写事件
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.unsafe().forceFlush();
        }

        // 处理可读事件
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

- **OP_CONNECT 连接建立事件：**表示 TCP 连接建立成功, Channel 处于 Active 状态。处理 OP_CONNECT 事件首先将该事件从事件集合中清除，避免事件集合中一直存在连接建立事件，然后调用 unsafe.finishConnect() 方法通知上层连接已经建立。最终会调用的 pipeline().fireChannelActive() 方法，这时会产生一个 Inbound 事件，然后会在 Pipeline 中进行传播，依次调用 ChannelHandler 的 channelActive() 方法，通知各个 ChannelHandler 连接建立成功。
- **OP_WRITE，可写事件：**表示上层可以向 Channel 写入数据，通过执行 ch.unsafe().forceFlush() 操作，将数据冲刷到客户端，最终会调用 javaChannel 的 write() 方法执行底层写操作。
- **OP_READ，可读事件：**表示 Channel 收到了可以被读取的新数据。Netty 将 READ 和 Accept 事件进行了统一的封装，都通过 unsafe.read() 进行处理。unsafe.read() 的逻辑可以归纳为几个步骤：从 Channel 中读取数据并存储到分配的 ByteBuf；调用 pipeline.fireChannelRead() 方法产生 Inbound 事件，然后依次调用 ChannelHandler 的 channelRead() 方法处理数据；调用 pipeline.fireChannelReadComplete() 方法完成读操作；最终执行 removeReadOp() 清除 OP_READ 事件。

**processSelectedKeysOptimized**

processSelectedKeysOptimized 与 processSelectedKeysPlain 的代码结构非常相似，其中最重要的一点就是 selectedKeys 的遍历方式是不同的，

因为 SelectedSelectionKeySet 内部使用的是 SelectionKey 数组，所以 processSelectedKeysOptimized 可以直接通过遍历数组取出 I/O 事件，相比 JDK HashSet 的遍历效率更高。SelectedSelectionKeySet 内部通过 size 变量记录数据的逻辑长度，每次执行 add 操作时，会把对象添加到 SelectionKey[] 尾部。当 size 等于 SelectionKey[] 的真实长度时，SelectionKey[] 会进行扩容。相比于 HashSet，SelectionKey[] 不需要考虑哈希冲突的问题，所以可以实现 O(1) 时间复杂度的 add 操作。

这个SelectedSelectionKeySet是NioEventLoop#openSelector 方法通过反射的方式，将 Selector 对象内部的 selectedKeys 和 publicSelectedKeys 替换为 SelectedSelectionKeySet

# 处理异步任务队列

NioEventLoop 内部有两个非常重要的异步任务队列，分别为普通任务队列和定时任务队列。NioEventLoop 提供了 execute() 和 schedule() 方法用于向不同的队列中添加任务，execute() 用于添加普通任务，schedule() 方法用于添加定时任务。

```
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    // 将任务添加到了 taskQueue
    addTask(task);
	...

    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}
```

在任务处理的场景下，inEventLoop() 始终是返回 true，始终都是在 Reactor 线程内串行执行。处理异步任务队列有 runAllTasks() 和 runAllTasks(long timeoutNanos) 两种实现，第一种会处理所有任务，第二种是带有超时时间来处理任务。之所以设置超时时间是为了防止 Reactor 线程处理任务时间过长而导致 I/O 事件阻塞，

```
protected boolean runAllTasks(long timeoutNanos) {
	// 合并定时任务到普通任务队列
    fetchFromScheduledTaskQueue();
    // 从普通任务队列中取出任务并处理
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }
	// 计算任务处理的超时时间
    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
    	// 执行任务
        safeExecute(task);

        runTasks ++;

        // 每执行 64 个任务检查一下是否超时
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
		// 继续取出下一个任务
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
	// 收尾工作，把尾部队列 tailTasks 里的任务以此取出执行一遍
    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

异步任务处理 runAllTasks 的过程可以分为三步：合并定时任务到普通任务队列，然后从普通任务队列中取出任务并处理，最后进行收尾工作

如果想对 Netty 的运行状态做一些统计数据，例如任务循环的耗时、占用物理内存的大小等等，都可以向尾部队列添加一个收尾任务完成统计数据的实时更新。