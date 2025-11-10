---
title: grpc 学习
date: 2021-04-13
tags: [grpc,c,cpp]
---

# rpc
rpc 意为`远程过程调用`, http, grpc 广义上讲都是rpc。
而且还有个项目叫[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway), 可以将grpc通过http的方式暴露。

# grpc

grpc 是rpc的一种实现，由google开源，其他还有thrift, sogorpc 等等。 并且grpc使用的http/2协议

## http/1.1 与 http/2 的区别

* 2使用二进制，而1.1使用文本，提高效率
* 2将相同的tcp连接合并为一个请求，提高性能,而1.1则为每个请求创建tcp连接
* 2的客户端使用流，这样可以多次请求
* 2含有trailers，也就是尾部消息，可以用来发送body的checksume等, 当然也可以直接放到body里
...

而1.1中也已经实现服务端到客户端的流,使用'Transfer-Encoding=chunked'来替代'Content-Length'，详见[rfc](https://datatracker.ietf.org/doc/html/rfc7230#section-3.3.2)
```
 A sender MUST NOT send a Content-Length header field in any message
   that contains a Transfer-Encoding header field.
```

## 认识proto文件

### proto 文件中多个service和单个service 区别

在同一个service里的方法会codegen到同一个类，但这个类比较鸡肋。
由于RPC调用是RESTful的，所以多次调用或者多个rpc方法无法通过同一个service来共享数据，这需要使用者借助其他办法来解决。

service 还可以用以隔离相同名称的rpc， 如 
- service1/helloworld
- service2/helloworld

而方法和方法通过`RpcServiceMethod`来保存，而通过index来调用
```
::grpc::Service::RequestAsyncUnary(0, context, request, response, new_call_cq, notification_cq, tag);
::grpc::Service::RequestAsyncUnary(1, context, request, response, new_call_cq, notification_cq, tag);
```

### rpc 声明UnaryCall&StreamingCall

非流调用也称为`UnaryCall`，指发送或接受的消息大小是固定的。
流调用称为`StreamingCall`，可以多次发送或者接收，所以消息大小并不固定。

StreamCall 可以多次调用，直到发送`WriteDone/Finish`，所以在接受的一端总是
```
while(read stream){}
```

grpc支持客户端流服务端非流、客户端非流、服务端流以及双向流，而普通的就是客户端和服务端都不流NORMAL_RPC(unary call)
- grpc::internal::RpcMethod::NORMAL_RPC
- grpc::internal::RpcMethod::RpcType::SERVER_STREAMING
- grpc::internal::RpcMethod::RpcType::CLIENT_STREAMING
- grpc::internal::RpcMethod::RpcType::BIDI_STREAMING

### 认识pb.h和grpc.pb.h文件
protoc 调用`grpc_cpp_plugin` 插件生成grpc.pb.{h,cc}文件，生成rpc方法的实现  

pb.{h,cc}则是定义了protobuf消息的序列化和反序列化方法

#### 反射、序列化和反序列化的实现
pb.h 实现grpc的请求参数和返回参数的特定语言的解析，还有pb的通用方法，
例如: `has_xx`(版本3里只有自定义类型才支持), `class XXX_CPP_API`

生成的class都继承自`google::protobuf::Message`
```
class HelloRequest PROTOBUF_FINAL :
      public ::PROTOBUF_NAMESPACE_ID::Message

#define PROTOBUF_NAMESPACE "google::protobuf"
#define PROTOBUF_NAMESPACE_ID google::protobuf 
```
而在`message`中有注释说明, 关键函数是`SerializeToString`和`ParseFromString`，还有个array版本`SerializeToArray`,  
还有一个反射函数`GetDescriptor()`用来动态获取指定槽位的数据
```
// Example usage:
  //
  // Say you have a message defined as:
  //
  //   message Foo {
  //     optional string text = 1;
  //     repeated int32 numbers = 2;
  //   }
  //
  // Then, if you used the protocol compiler to generate a class from the above
  // definition, you could use it like so:
  //
  //   std::string data;  // Will store a serialized version of the message.
  //
  //   {
  //     // Create a message and serialize it.
  //     Foo foo;
  //     foo.set_text("Hello World!");
  //     foo.add_numbers(1);
  //     foo.add_numbers(5);
  //     foo.add_numbers(42);
  //
  //     foo.SerializeToString(&data);
  //   }
  //
  //   {
  //     // Parse the serialized message and check that it contains the
  //     // correct data.
  //     Foo foo;
  //     foo.ParseFromString(data);
  //
  //     assert(foo.text() == "Hello World!");
  //     assert(foo.numbers_size() == 3);
  //     assert(foo.numbers(0) == 1);
  //     assert(foo.numbers(1) == 5);
  //     assert(foo.numbers(2) == 42);
  //   }
```

如下可以将Message转换为基本类型
```
int size = reqMsg.ByteSizeLong();
char* array = new char[size];
reqMsg.SerializeToArray(array, size);
```

```
std::string bytes = reqMsg.SerializeAsString();
const char* array = bytes.data();
int size = bytes.size();
```

进一步看`protobuf::message`继承自`protobuf::message_lite`, 后者实现了`SerializeAsString`和`SerializeToArray`
```
inline uint8* SerializeToArrayImpl(const MessageLite& msg, uint8* target,
                                     int size) {
    constexpr bool debug = false;
    if (debug) {
      // Force serialization to a stream with a block size of 1, which forces
      // all writes to the stream to cross buffers triggering all fallback paths
      // in the unittests when serializing to string / array.
      io::ArrayOutputStream stream(target, size, 1);
      uint8* ptr;
      io::EpsCopyOutputStream out(
          &stream, io::CodedOutputStream::IsDefaultSerializationDeterministic(),
          &ptr);
      ptr = msg._InternalSerialize(ptr, &out);
      out.Trim(ptr);
      GOOGLE_DCHECK(!out.HadError() && stream.ByteCount() == size);
      return target + size;
    } else {
      io::EpsCopyOutputStream out(
          target, size,
          io::CodedOutputStream::IsDefaultSerializationDeterministic());
实际调用->    auto res = msg._InternalSerialize(target, &out);
      GOOGLE_DCHECK(target + size == res);
      return res;
    }
  }
```
可见，其实序列化最终调用的是pb.h文件里定义的`_InternalSerialize`, 举例官方例子`HelloRequest`
```
 ::PROTOBUF_NAMESPACE_ID::uint8* HelloRequest::_InternalSerialize(
      ::PROTOBUF_NAMESPACE_ID::uint8* target, ::PROTOBUF_NAMESPACE_ID::io::EpsCopyOutputStream*   stream) const {
    // @@protoc_insertion_point(serialize_to_array_start:helloworld.HelloRequest)
    ::PROTOBUF_NAMESPACE_ID::uint32 cached_has_bits = 0;
    (void) cached_has_bits;
  
    // string name = 1;
    if (this->name().size() > 0) {
      ::PROTOBUF_NAMESPACE_ID::internal::WireFormatLite::VerifyUtf8String(
        this->_internal_name().data(), static_cast<int>(this->_internal_name().length()),
        ::PROTOBUF_NAMESPACE_ID::internal::WireFormatLite::SERIALIZE,
        "helloworld.HelloRequest.name");
      target = stream->WriteStringMaybeAliased(
          1, this->_internal_name(), target);
    }
  
    if (PROTOBUF_PREDICT_FALSE(_internal_metadata_.have_unknown_fields())) {
      target = ::PROTOBUF_NAMESPACE_ID::internal::WireFormat::InternalSerializeUnknownFieldsToA  rray(
         _internal_metadata_.unknown_fields<::PROTOBUF_NAMESPACE_ID::UnknownFieldSet>(::PROTOB  UF_NAMESPACE_ID::UnknownFieldSet::default_instance), target, stream);
    }
    // @@protoc_insertion_point(serialize_to_array_end:helloworld.HelloRequest)
    return target;
  }
  
```

#### grpc.pb生成的代码实现rpc调用

生成的框架代码用来继承实现Service和获取stub来发起rpc call。实际上这些代码并不是必须的  
在下面讲了如何使用几个工厂类来创建Stub，还有直接`new`出Service

```
class XXXServer {
        // 客户端使用的桩
	class Stub
        // base 
	class Service
	// 各种版本的rpc包装，但都继承自base
        class WithAsyncMethod_XXX
        typedef WithAsyncMethod_XXX<Service > AsyncService;
	typedef ExperimentalWithCallbackMethod_XXX<Service > CallbackService;
	class WithGenericMethod_XXX
	class WithRawMethod_XXX
	typedef WithStreamedUnaryMethod_XXX<Service > StreamedUnaryService;
}
```

## 同步与异步

grpc 的异步即为使用cq事件驱动（cq-based），使用tag标记事件。另外还有callback方式

### 对于客户端
同步时，通过调用'::grpc::internal::BlockingUnaryCall'  
异步时，创建'ClientAsyncResponseReader'(非流), 然后通过调用'ClientAsyncResponseReader'的write和finish，并等待tag
当存在流时分别是
- ::grpc::ClientAsyncReader
- ::grpc::ClientAsyncWriter
- ::grpc::ClientAsyncReaderWriter

这些类型可用对应的工厂类来创建, 生成代码的stub也是这么用的
```
class ClientReaderFactory 
class ClientWriterFactory 
class ClientReaderWriterFactory 
```

### 对于服务端
同步时，通过'AddMethod'来注册，生成代码会在父类构造时执行。注册后由grpc调用
```
Greeter::Service::Service() {
    AddMethod(new ::grpc::internal::RpcServiceMethod(
        Greeter_method_names[0],
        ::grpc::internal::RpcMethod::NORMAL_RPC,
        new ::grpc::internal::RpcMethodHandler< Greeter::Service, ::helloworld::HelloRequest, ::helloworld::HelloReply>(
            [](Greeter::Service* service,
               ::grpc_impl::ServerContext* ctx,
               const ::helloworld::HelloRequest* req,
               ::helloworld::HelloReply* resp) {
                 return service->SayHello(ctx, req, resp);
               }, this)));
  }
```

异步时，类似客户端
- grpc::ServerAsyncReaderWriter
- grpc::ServerAsyncReader
- grpc::ServerAsyncWriter

可见服务端是直接new出来的，异步时这些io操作对象也是直接new出来的, 在调用以下时传入
```
RequestAsyncBidiStreaming
RequestAsyncClientStreaming
RequestAsyncServerStreaming
```

## grpc callback

只在客户端使用，callback方式的请求可以传入一个lambda, 在请求完成时调用
```
    stub_->async()->SayHello(&context, &request, &reply,
                             [&mu, &cv, &done, &status](Status s) {
                               status = std::move(s);
                               std::lock_guard<std::mutex> lock(mu);
                               done = true;
                               cv.notify_one();
                             });
```

新版本的grpc已经将实验性的标记去除，说明此方式成熟了
```
    #ifdef GRPC_CALLBACK_API_NONEXPERIMENTAL
      ::grpc::Service::
    #else
      ::grpc::Service::experimental().
    #endif
```

## grpc异步流

官方仓库的示例代码没有异步且流的, 在实际项目中用到异步流,使用大概方法
1. 手动创建writereader
2. 启动时，调用'grpc::Service::RequestAsyncBidiStreaming' 和 'grpc::Service::RequestAsyncClientStreaming' 以及'RequestAsyncServerStreaming', 向cq塞请求new_connection事件
3. 收到'new_connection'事件返回后，再调用read事件。

一共有5个类型
```
new_connection, read, write, finish, done
```
我写了一个demo [grpcstreamhelloworld](https://github.com/energygreek/grpcstreamhelloworld)

## grpc 消息大小

老版本的grpc中，发送端是支持无限大小的，但接受端只能是4M
```
#define GRPC_DEFAULT_MAX_SEND_MESSAGE_LENGTH -1
#define GRPC_DEFAULT_MAX_RECV_MESSAGE_LENGTH (4 * 1024 * 1024)
```
服务端代码
```
std::unique_ptr<Server> ServerBuilder::BuildAndStart() {
    if (max_receive_message_size_ >= 0) {
      args.SetInt(GRPC_ARG_MAX_RECEIVE_MESSAGE_LENGTH, max_receive_message_size_);
    }
```

但在新版grpc中变了

```
  std::unique_ptr<grpc::Server> ServerBuilder::BuildAndStart() {
    grpc::ChannelArguments args;
    if (max_receive_message_size_ >= -1) {
      args.SetInt(GRPC_ARG_MAX_RECEIVE_MESSAGE_LENGTH, max_receive_message_size_);
    }
    if (max_send_message_size_ >= -1) {
      args.SetInt(GRPC_ARG_MAX_SEND_MESSAGE_LENGTH, max_send_message_size_);
    }
```

## grpc 编译安装的问题

https://github.com/grpc/grpc/issues/13841

## grpc异步存在问题

因为异步服务端通过completionqueue来通知rpc执行结果和执行下次调用，通常使用多queue和多线程的方式提高处理效率
1. 通常情况是多queue, 即每个service对应一个queue, 而每个service又有多个rpc，线程去轮询这个complete_queue。这样导致高线程切换开销，而且complete_queue也占用大量内存
2. 多线程，queue可以用多个线程去轮询，但0.13版本可能出现bug

## grpc异步流存在的问题

grpc区别与其他框架很大一个优势是支持异步流，即可以多次请求和多次回复。异步是基于cq的事件驱动，所以必须等待tag回调, 连续两次发送会异常。
而真正的请求一般在业务模块处理, 不知道tag的状态即不知道是否正在发送， 那么如何在cq回调外发送消息呢？ 

办法是维护一个发送队列，消息先存队列里，等待cq回调时取出发送。 另外由于流同步需要显式发送结束标记(服务端调Stream::Finish, 客户端调用WriteDown和Finish), 
所以需要有一个特殊消息加以区分，通常用空指针，也可以设置结束标志。另外由于发送代码会同时被业务调用和cq回调，需要对发送代码加锁

## 调试grpc

通过设置环境变量，让grpc向控制台打印详细信息
```
export GRPC_VERBOSITY=DEBUG
bash-5.0# ./build/bin/hasync slave  stdin stdout @127.0.0.1:7615
D1026 08:27:44.142802149   24658 ev_posix.cc:174]            Using polling engine: epollex
D1026 08:27:44.143406685   24658 dns_resolver_ares.cc:490]   Using ares dns resolver
I1026 08:27:44.158115785   24658 server_builder.cc:332]      Synchronous server. Num CQs: 1, Min pollers: 1, Max Pollers: 2, CQ timeout (msec): 10000
```

## 项目实践

项目使用客户端异步/同步，服务端全异步， 可以兼容四种传输方式

## 引用
https://grpc.github.io/grpc/cpp/grpcpp_2impl_2codegen_2sync__stream_8h_source.html
https://grpc.github.io/grpc/cpp/grpcpp_2impl_2codegen_2byte__buffer_8h_source.html
https://grpc.github.io/grpc/cpp/call__op__set_8h_source.html

