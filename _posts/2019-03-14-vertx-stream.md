---
layout: post
title: Vert.x Stream
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-stream.html
---

# Stream

Stream是vert.x提供的表明对象支持写入和读取的接口

在Vert.x里，写是立即写入，并且再内部队列中排队．因此如果对对象的写入速度比它将数据实际写到底层资源对速度快，那么写队列可以无限增长，最终会导致内存耗尽．可以通过Vert.x提供的某些简单的流量控制来解决这个问题．

所有能够写入的对象都可以实现WriteStream，所有能够读取的对象都可以实现ReadStream．

# 示例　NetSocket 
ＮetSocket实现了ReadStream和WriteStream接口

	interface NetSocket extends ReadStream<Buffer>, WriteStream<Buffer>

下面来演示一个从socket读取和写入数据的功能

## 最简单的实现

	    vertx.createNetServer().connectHandler(socket -> {
	      socket.handler(buffer -> {
		socket.write(buffer);
	      });
	    }).listen(1234, ar -> {
	      if (ar.succeeded()) {
		System.out.println("Echo server is now listening");
	      } else {

	      }
	    });
	    
上述例子有一个问题，如果从socket中读取数据速度比写回socket的速度要快，它会在NetSocket上建立写入队列，总在耗尽内存．这种情况是可能发生的，例如如果socket另一端的client读取数据对速度不够快，将会将连接上对压力往后推．

因为NetSocket实现了WriteStream，我们可以再写入数据之前判断WriteStream是否已经满了．

      socket.handler(buffer -> {
        if (!socket.writeQueueFull()) {
          socket.write(buffer);
        }
      });

上面例子不会写入内存，但是如果写队列已经满类，最终会丢失数据．而我们真正想做的是在写队列满对时候暂停写入．

      socket.handler(buffer -> {
        socket.write(buffer);
        if (socket.writeQueueFull()) {
          socket.pause();
        }
      });

之后，我们还需要取消暂停

      socket.handler(buffer -> {
        socket.write(buffer);
        if (socket.writeQueueFull()) {
          socket.pause();
          socket.drainHandler(v -> socket.resume());
        }
      });
      
在写队列可以写入数据的时候，drainHandler会恢复NetSocket，来允许新数据的读取．

Vert.x提供了一个辅助函数用来完成上诉功能

      socket.handler(buffer -> {
        Pump.pump(socket, socket).start();
      });

## ReadStream接口

- handler方法： 从ReadStream读取数据
- pause方法： 暂停handler，暂停时不会再handler中接收任何数据
- resume方法： 恢复handler,如果有数据到达，handler会被执行
- exceptionHandler ：RealStream发生异常时会被调用
- endHandler：在整个流结束时会被调用．例如从一个文件对ReadStream里读取到一个EOF，或者是HTTP请求,TCP Socket的关闭

## WriteStream接口

- write方法：向WriteStream写入数据．这个方法永远不会阻塞．它会立即写入队列，然后异步对写入底层资源．
- setWriteQueueMaxSize方法：设置写队列的大小，如果写队列已经满了，则`writeQueueFull`方法会返回true. **即使写队列已经满了，调用write方法依然会接收数据并入队．实际的数字取决于流对实现．例如Buffer的大小表示实际写入的字节数，而不是bufer的数量**
- writeQueueFull方法：如果写队列已满，返回true
- exceptionHandler WriteStream发生异常时会被调用
- drainHandler 如果WriteStream认为写队列不再是满的时候，会调用这个方法

## Pump
Pump是一个汞，它主要对数据从readstream到writestream进行流量控制，防止向写入流的缓冲区写入过多数据．

这个类从ReadStream读取数据并将它们写入到WriteStream．如果读取数据的速度要比写入数据对速度块，可能会导致WriteStream写队列没有约束，最终耗尽内存．

为了防止发生上面的问题，在每次写入之后都需要检查写队列是否已经满类．如果满了，将会暂停ReadStream，并且设置一个drainHandler.当WriteSteram已经处理了一半积压在写队列中的数据．drainHandler将会被调用，重新恢复ReadStream的读取．

Pump可以在任意的ReadStream或WriteStream之间使用．例如：从HttpServerRequest写入到AsyncFIle，或者从NetSocket写入到WebSocket.

# 原理
## Pump

Pump的构造方法很简单，就是创建前面例子中的drainHandler和dataHandler

	  PumpImpl(ReadStream<T> rs, WriteStream<T> ws, int maxWriteQueueSize) {
	    this(rs, ws);
	    this.writeStream.setWriteQueueMaxSize(maxWriteQueueSize);
	  }

	  PumpImpl(ReadStream<T> rs, WriteStream<T> ws) {
	    this.readStream = rs;
	    this.writeStream = ws;
	    drainHandler = v-> readStream.resume();
	    dataHandler = data -> {
	      writeStream.write(data);
	      incPumped();
	      if (writeStream.writeQueueFull()) {
		readStream.pause();
		writeStream.drainHandler(drainHandler);
	      }
	    };
	  }

start方法

	  @Override
	  public PumpImpl start() {
	    readStream.handler(dataHandler);
	    return this;
	  }

为ReadStream设置读取数据的Handler，这个handler已经在创建Pump对构造函数中声明．

	    dataHandler = data -> {
	      writeStream.write(data);
	      incPumped();
	      if (writeStream.writeQueueFull()) {
		readStream.pause();
		writeStream.drainHandler(drainHandler);
	      }
	    };

stop方法直接将WriteStream和ReadStream中注册的drainHandler和dataHandler删除．

	  /**
	   * Stop the Pump. The Pump can be started and stopped multiple times.
	   */
	  @Override
	  public PumpImpl stop() {
	    writeStream.drainHandler(null);
	    readStream.handler(null);
	    return this;
	  }
  
## ReadStream，WriteStream
借助AsyncFile来看下ReadStream，WriteStream的实现

handler方法再设置handler之后，调用doRead方法

	  @Override
	  public synchronized AsyncFile handler(Handler<Buffer> handler) {
	    check();
	    this.dataHandler = handler;
	    if (dataHandler != null && !paused && !closed) {
	      doRead();
	    }
	    return this;
	  }

doRead方法在读取一次数据之后，就会通过`handleData(buffer);`方法调用上面方法注册的handler

	  private synchronized void doRead() {
	    if (!readInProgress) {
	      readInProgress = true;
	      Buffer buff = Buffer.buffer(readBufferSize);
	      read(buff, 0, readPos, readBufferSize, ar -> {
		if (ar.succeeded()) {
		  readInProgress = false;
		  Buffer buffer = ar.result();
		  if (buffer.length() == 0) {
		    // Empty buffer represents end of file
		    handleEnd();
		  } else {
		    readPos += buffer.length();
		    handleData(buffer);
		    if (!paused && dataHandler != null) {
		      doRead();
		    }
		  }
		} else {
		  handleException(ar.cause());
		}
	      });
	    }
	  }

	  private synchronized void handleData(Buffer buffer) {
	    if (dataHandler != null) {
	      checkContext();
	      dataHandler.handle(buffer);
	    }
	  }

pause方法将`paused`标识位设置为true，然后doRead方法会暂停文件的读取

	  @Override
	  public synchronized AsyncFile pause() {
	    check();
	    paused = true;
	    return this;
	  }
  	
  	  if (!paused && dataHandler != null) {
	      doRead();
	    }

resume方法将`paused`标识位设置为false，然后通过doRead方法重新读取数据

	  @Override
	  public synchronized AsyncFile resume() {
	    check();
	    if (paused && !closed) {
	      paused = false;
	      if (dataHandler != null) {
		doRead();
	      }
	    }
	    return this;
	  }
  
在doRead方法中，如果判断文件已经读完，会通过handleEnd方法来调用endHandler

	 if (buffer.length() == 0) {
	    // Empty buffer represents end of file
	    handleEnd();
	  } 
	  
	  private synchronized void handleEnd() {
	    if (endHandler != null) {
	      checkContext();
	      endHandler.handle(null);
	    }
	  }

setWriteQueueMaxSize方法设置了两个参数，写队列的大小`maxWrites`，drainHandler被触发时候时的队列大小`lwm`

	  @Override
	  public synchronized AsyncFile setWriteQueueMaxSize(int maxSize) {
	    Arguments.require(maxSize >= 2, "maxSize must be >= 2");
	    check();
	    this.maxWrites = maxSize;
	    this.lwm = maxWrites / 2;
	    return this;
	  }

在write方法会通过doWrite方法向写队列写入数据，并更新writesOutstanding．

  private void doWrite(ByteBuffer buff, long position, long toWrite, Handler<AsyncResult<Void>> handler) {
    if (toWrite == 0) {
      throw new IllegalStateException("Cannot save zero bytes");
    }
    writesOutstanding += toWrite;
    writeInternal(buff, position, handler);
  }
  
  在数据写入底层资源之后会重新更新writesOutstanding．
  
	   private void writeInternal(ByteBuffer buff, long position, Handler<AsyncResult<Void>> handler) {

	    ch.write(buff, position, null, new java.nio.channels.CompletionHandler<Integer, Object>() {

	      public void completed(Integer bytesWritten, Object attachment) {

		long pos = position;

		if (buff.hasRemaining()) {
		  // partial write
		  pos += bytesWritten;
		  // resubmit
		  writeInternal(buff, pos, handler);
		} else {
		  // It's been fully written
		  context.runOnContext((v) -> {
		    writesOutstanding -= buff.limit();
		    handler.handle(Future.succeededFuture());
		  });
		}
	      }

	      public void failed(Throwable exc, Object attachment) {
		if (exc instanceof Exception) {
		  context.runOnContext((v) -> handler.handle(Future.succeededFuture()));
		} else {
		  log.error("Error occurred", exc);
		}
	      }
	    });
	  }
  
在每次写入数据之后都会触发回调handler，用来检查是否需要调用checkDrained方法;

	  private synchronized void checkDrained() {
	    if (drainHandler != null && writesOutstanding <= lwm) {
	      Handler<Void> handler = drainHandler;
	      drainHandler = null;
	      handler.handle(null);
	    }
	  }
	  
回调handler是在doWrite方法中指定`Handler<AsyncResult<Void>> wrapped `

	  private synchronized AsyncFile doWrite(Buffer buffer, long position, Handler<AsyncResult<Void>> handler) {
	    Objects.requireNonNull(buffer, "buffer");
	    Arguments.require(position >= 0, "position must be >= 0");
	    check();
	    Handler<AsyncResult<Void>> wrapped = ar -> {
	      if (ar.succeeded()) {
		checkContext();
		if (writesOutstanding == 0 && closedDeferred != null) {
		  closedDeferred.run();
		} else {
		  checkDrained();
		}
		if (handler != null) {
		  handler.handle(ar);
		}
	      } else {
		if (handler != null) {
		  handler.handle(ar);
		} else {
		  handleException(ar.cause());
		}
	      }
	    };
	    ByteBuf buf = buffer.getByteBuf();
	    if (buf.nioBufferCount() > 1) {
	      doWrite(buf.nioBuffers(), position, wrapped);
	    } else {
	      ByteBuffer bb = buf.nioBuffer();
	      doWrite(bb, position, bb.limit(),  wrapped);
	    }
	    return this;
	  }