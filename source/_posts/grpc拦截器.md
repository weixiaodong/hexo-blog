---
title: grpc拦截器
date: 2023-03-18 20:33:52
tags:
---

# 1. 概述

grpc提供了interceptor功能，包括客户端拦截器和服务端拦截器。 可以在接收请求或者发起请求之前优先对请求中的数据做一些处理后再转交给指定的服务处理并响应，很适合在这里处理验证、日志等流程。

<!--more-->

在grpc中，根据拦截的方法类型不同可以分为拦截unary方法的一元拦截器，和作用于stream方法的流拦截器。 同时还分为服务端拦截器和客户端拦截器，所以一共有以下4种类型：

- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor

# 2. 定义

## 2.1 客户端拦截器

>Unary Interceptor

使用客户端拦截器，只需要在Dial的时候指定相应的DialOption即可。

客户端一元拦截器类型为 grpc.UnaryClientInterceptor, 具体如下:

```golang
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

可以看到，所谓的拦截器其实就是一个函数，可以分为预处理（pre-processing）、调用rpc方法（invoking rpc method）、后处理（post-processing）三个阶段。

参数含义如下：
- ctx: go语言中的上下文，一般和goroutine配合使用，起到超时控制的效果
- method: 当前调用的rpc方法名
- req: 本次请求的参数，只有在处理前阶段修改才有效
- reply: 本次请求响应，需要在处理后阶段才能获取到
- cc: grpc连接信息
- invoker: 可以看做是当前rpc方法，一般在拦截器中调用invoker能达到调用rpc方法的效果，当然底层也是grpc在处理。
- opts: 本次调用指定的options信息

作为一个客户端拦截器，可以在处理前检查req看看本次请求带没带token之类的鉴权数据，没有的话就可以在拦截器中加上

>stream interceptor

```golang
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```

由于streamAPI和unaryAPI有所不同，因此拦截器方面也有所区别，比如req参数变成了streamer。同时其拦截过程也有所不同，不在是处理req resp，而是对streamer这个流对象进行包装，比如说实现自己的sendmsg和reqmsg方法。

然后在这些方法中的预处理、调用rpc方法、后处理各个阶段加入自己的逻辑。

## 2.2 服务端拦截器

服务端拦截器和客户端拦截器类似，就不做过多描述。使用客户端拦截器只需要在NewServer的时候指定相应的ServerOption即可。

> unary Interceptor

定义如下：

```golang
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHander) (resp interface{}, err error)
```

参数具体含义如下：
- ctx context.Context: 请求上下文
- req interface{}: rpc方法的请求参数
- info *unaryServerInfo: rpc方法的所有信息
- handler unaryHandler: rpc方法正在执行的逻辑

> stream Interceptor

```golang
type StreamServerInterceptor func(src interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

# unaryInterceptor

一元拦截器可以分为3个阶段:
- 1) 预处理
- 2) 调用rpc方法
- 3) 后处理

>client

```golang
// unaryInterceptor 一个简单的 unary interceptor 示例。
func unaryInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	// pre-processing
	start := time.Now()
	err := invoker(ctx, method, req, reply, cc, opts...) // invoking RPC method
	// post-processing
	end := time.Now()
	logger("RPC: %s, req:%v start time: %s, end time: %s, err: %v", method, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
	return err
}
```
invoker(ctx, method, req, reply, cc, opts...) 是真正调用 RPC 方法。因此我们可以在调用前后增加自己的逻辑：比如调用前检查一下参数之类的，调用后记录一下本次请求处理耗时等。

建立连接时通过 grpc.WithUnaryInterceptor 指定要加载的拦截器即可。

```golang
func main() {
	flag.Parse()

	creds, err := credentials.NewClientTLSFromFile(data.Path("x509/server.crt"), "www.lixueduan.com")
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}

	// 建立连接时指定要加载的拦截器
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(creds), grpc.WithUnaryInterceptor(unaryInterceptor))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := ecpb.NewEchoClient(conn)
	callUnaryEcho(client, "hello world")
}
```

> server

服务端的一元拦截器和客户端类似：

```golang
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	start := time.Now()
	m, err := handler(ctx, req)
	end := time.Now()
	// 记录请求参数 耗时 错误信息等数据
	logger("RPC: %s,req:%v start time: %s, end time: %s, err: %v", info.FullMethod, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
	return m, err
}
```

服务端则是在 NewServer 时指定拦截器：

```golang
func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	creds, err := credentials.NewServerTLSFromFile(data.Path("x509/server.crt"), data.Path("x509/server.key"))
	if err != nil {
		log.Fatalf("failed to create credentials: %v", err)
	}

	s := grpc.NewServer(grpc.Creds(creds), grpc.UnaryInterceptor(unaryInterceptor))

	pb.RegisterEchoServer(s, &server{})
	log.Println("Server gRPC on 0.0.0.0" + fmt.Sprintf(":%d", *port))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

# 4. StreamInterceptor

流拦截器过程和一元拦截器有所不同，不过同样可以分为3个阶段：

1）预处理(pre-processing)
2）调用RPC方法(invoking RPC method)
3）后处理(post-processing)
预处理阶段和一元拦截器类似，但是调用RPC方法和后处理这两个阶段则完全不同。

StreamAPI 的请求和响应都是通过 Stream 进行传递的，更进一步是通过 Streamer 调用 SendMsg 和 RecvMsg 这两个方法获取的。

然后 Streamer 又是调用RPC方法来获取的，所以在流拦截器中我们可以对 Streamer 进行包装，然后实现 SendMsg 和 RecvMsg 这两个方法。

> client

本例中通过结构体嵌入的方式，对 Streamer 进行包装，在 SendMsg 和 RecvMsg 之前打印出具体的值。

```golang
// wrappedStream  用于包装 grpc.ClientStream 结构体并拦截其对应的方法。
type wrappedStream struct {
	grpc.ClientStream
}

func newWrappedStream(s grpc.ClientStream) grpc.ClientStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	logger("Receive a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.SendMsg(m)
}

// streamInterceptor 一个简单的 stream interceptor 示例。
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
	s, err := streamer(ctx, desc, cc, method, opts...)
	if err != nil {
		return nil, err
	}
    // 返回的是自定义的封装过的 stream
	return newWrappedStream(s), nil
}
```

连接时则通过 grpc.WithStreamInterceptor 指定要加载的拦截器。

```golang
func main() {
	flag.Parse()

	creds, err := credentials.NewClientTLSFromFile(data.Path("x509/server.crt"), "www.lixueduan.com")
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}

	// 建立连接时指定要加载的拦截器
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(creds), grpc.WithStreamInterceptor(streamInterceptor))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := ecpb.NewEchoClient(conn)
	// callUnaryEcho(client, "hello world")
	callBidiStreamingEcho(client)
}
```

> server

和客户端类似。

```golang
type wrappedStream struct {
	grpc.ServerStream
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	logger("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.SendMsg(m)
}

func streamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	// 包装 grpc.ServerStream 以替换 RecvMsg SendMsg这两个方法。
	err := handler(srv, newWrappedStream(ss))
	if err != nil {
		logger("RPC failed with error %v", err)
	}
	return err
}
```

相似的，通过grpc.StreamInterceptor指定要加载的拦截器。

```golang
func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	creds, err := credentials.NewServerTLSFromFile(data.Path("x509/server.crt"), data.Path("x509/server.key"))
	if err != nil {
		log.Fatalf("failed to create credentials: %v", err)
	}

	s := grpc.NewServer(grpc.Creds(creds), grpc.StreamInterceptor(streamInterceptor))

	pb.RegisterEchoServer(s, &server{})
	log.Println("Server gRPC on 0.0.0.0" + fmt.Sprintf(":%d", *port))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

# 5. 小结

1）拦截器分类与定义 gRPC 拦截器可以分为：一元拦截器和流拦截器，服务端拦截器和客户端拦截器。

一共有以下4种类型:

- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor

拦截器本质上就是一个特定类型的函数，所以实现拦截器只需要实现对应类型方法（方法签名相同）即可。

2）拦截器执行过程

一元拦截器
- 1）预处理
- 2）调用RPC方法
- 3）后处理

流拦截器
- 1）预处理
- 2）调用RPC方法 获取 Streamer
- 3）后处理
	调用 SendMsg 、RecvMsg 之前
	调用 SendMsg 、RecvMsg
	调用 SendMsg 、RecvMsg 之后

3）拦截器使用及执行顺序

配置多个拦截器时，会按照参数传入顺序依次执行

所以，如果想配置一个 Recovery 拦截器则必须放在第一个，放在最后则无法捕获前面执行的拦截器中触发的 panic。

同时也可以将 一元和流拦截器一起配置，gRPC 会根据不同方法选择对应类型的拦截器执行。

