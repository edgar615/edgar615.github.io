---
layout: post
title: GRPC（10）- 反射创建对象
date: 2020-06-10
categories:
    - rpc
comments: true
permalink: grpc-reflection.html
---

因为项目需要做rest到gRPC的动态转换，所以尝试了一下通过反射创建对象

- 构建Message

```
        Class clazz = GrpcReflectExample.class.getClassLoader()
                .loadClass("com.github.edgar615.grpc.HelloRequest");
        Method method = clazz.getMethod("newBuilder");
//        newBuilder 为静态变量，即使没有 message 的具体实例也可以
//        测试反射代码
//        HelloRequest.Builder builder = (HelloRequest.Builder) method.invoke(null, new Object[]{});
//        System.out.println(builder.setName("edgar").build());
        Message.Builder builder = (Message.Builder) method.invoke(null, new Object[]{});
        // 反射构建HelloRequest
        Descriptors.Descriptor descriptor = builder.getDescriptorForType();
        // 找到对应的字段
        Descriptors.FieldDescriptor field = descriptor.findFieldByName("name");
//         根据类型对字段做处理
//        if (field.getType() == Descriptors.FieldDescriptor.Type.MESSAGE) {
//            if (field.isRepeated()) {}
//        } else if (field.getType() == Descriptors.FieldDescriptor.Type.ENUM) {
//        } else {
//            switch (field.getJavaType()) {
//                case FLOAT: // float is a special case
//                case INT:
//                case LONG:
//                case DOUBLE:
//                default:
//            }
//        }
        builder.setField(field, "edgar");
        System.out.println(builder.build());
```

- 发送请求

```
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost", 9998)
                .usePlaintext()
                .build();
        Class grpcClass = GrpcReflectExample.class.getClassLoader()
                .loadClass("com.github.edgar615.grpc.GreeterGrpc");
        Class stubClass = GrpcReflectExample.class.getClassLoader()
                .loadClass("com.github.edgar615.grpc.GreeterGrpc$GreeterBlockingStub");
        Method stubMethod = grpcClass.getMethod("newBlockingStub", Channel.class);
        // 或者
        // Method stubMethod = null;
        // for (Method method1 : grpcClass.getMethods()) {
        //     if (method1.getName().equals("newBlockingStub")) {
        //         stubMethod = method1;
        //     }
        // }

        AbstractStub stub = (AbstractStub) stubMethod.invoke(null, new Object[]{managedChannel});
        System.out.println(stub);
        Method rpcMethod = stubClass.getMethod("sayHello", clazz);
        Message reply = (Message) rpcMethod.invoke(stub, message);
        System.out.println(reply);
```

- 基于JSON转换

有时候我们要将对象转换为JSON，如果直接用protobuf的对象返回的话会抛出异常。原因是protobuf生成对象的get方法，返回的类型有byte[]，而只有String类型可以作为json的key。

不过google有提供专门的架包，方便protobuf与json之间相互转换

```
实际上已经集成在了grpc-all中，不用再次引入
<dependency>
	<groupId>com.googlecode.protobuf-java-format</groupId>
	<artifactId>protobuf-java-format</artifactId>
	<version>1.4</version>
</dependency>
```

```
// protobuf -> json
String json = JsonFormat.printer().print(reply);
System.out.println(json);

// json -> protobuf
Message.Builder messageBuilder = HelloReply.newBuilder();
JsonFormat.parser()
		.merge("{ \"message\": \"hello edgar\"}", messageBuilder);
HelloReply helloReply = (HelloReply) messageBuilder.build();
System.out.println(helloReply);
```

