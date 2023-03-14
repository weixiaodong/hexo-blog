---
title: go-每日一库之-grpc使用
date: 2023-03-14 10:36:00
tags:
---

## grpc基础：

本教程提供了go程序员如何使用grpc的指南。

通过学习教程中的例子，你可以学会如何：

- 在一个.proto文件内定义服务
- 使用grpc的go api为你的服务实现一个简单的客户端和服务端

<!--more-->

## 为什么使用grpc

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了grpc，我们可以一次性的在一个.proto文件中定义服务并使用任何支持她的语言去实现客户端和服务端 -- grpc帮你解决了不同语言及环境间通信的复杂性，使用protocol buffer还能获得其他好处，包括高效的序列号，简单的IDL以及容易进行接口更新。

## 定义服务

要定义一个服务，你必须在你的.proto文件中指定service:

```golang
service RouteGuide {
	...
}
```

然后在你的服务中定义rpc方法，指定请求和响应类型。grpc允许你定义4中类型的service方法，这些都在routeguide服务中使用：

- 一个简单rpc，客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```golang
rpc GetFeature(Point) returns (Feature) {}
```

- 一个服务端流式rpc，客户端发送请求到服务端，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在响应类型前插入stream 关键字，可以指定一个服务器端的流方法。

```golang
rpc ListFeatures(Rectanle) returns (stream Feature) {}
```

- 一个客户端流式rpc，客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一单客户端完成写入消息，它等待服务器完成读取返回它的响应。 通过在请求类型前指定stram关键字来指定一个客户端流方法。

```golang
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

- 一个双向流式rpc是双发使用读写流去发送一个消息序列。 两个流独立操作，因此客户端和服务端可以以任意喜欢的顺序读写：比如， 服务器可以在写入响应前等待接收所有的客户端消息，或者可以交替的读取和写入消息，或者其他读写的组合。 每个流中的消息顺序被预留。 你可以通过在请求和响应前加stream关键字指定方法的类型

```golang
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的.proto文件也包含了所有请求的protocol buffer消息类型定义以及在服务方法中使用的响应类型，比如，下面的Point消息类型:

```golang
message Point {
	int32 latitude = 1;
	int32 longitude = 2;
}
```

## 生成客户端和服务器代码

接下来我们需要从.proto的服务定义中生成grpc客户端和服务器的接口。 我们通过protocol buffer的编译器protoc以及一个特殊的grpc go插件来完成。

```bash
protoc --go_out=plugins=grpc:. route_guide.proto
```

运行这个命令可以在当前目录中生成下面的文件:

- route_guide.pb.go

这些包括：

- 所有用于填充，序列化和获取我们请求和响应消息类型的protocol buffer代码
- 一个为客户端调用定义在RouteGuide服务的方法的接口类型（或者存根）
- 一个为服务器使用定义在RouteGuide服务的方法去实现的接口类型（或者存根）

## 创建服务器

首先来看看我们如何创建一个RouteGuide服务器。 

让RouteGuide服务工作有两个部分：

- 实现我们服务定义的生成的服务接口： 做我们的服务的实际的工作
- 运行一个grpc服务器， 监听来自客户单的请求并返回服务的响应。

## 实现RouteGuide

我们可以看出，服务器有一个实现了生成的RouteGuideServer接口的routeGuideServer结构类型：

```golang
type routeGuideServer struct {
	...
}

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
```

## 简单rpc

routeGuideServer实现了我们所有的服务方法。 首先让我们看看最简单的类型GetFeature

```golang

func (s *routeGuideServer) GetFeature(ctx context.Context, point *bp.Point) (*pb.Feature, error) {
	for _, feature := range s.savedFeatures {
		if proto.Equal(feature.Location, point) {
			return feature, nil
		}
	}

	return &pb.Feature{"", point}, nil
}
```

## 服务端流式rpc

现在让我们来看看我们的一种流式rpc。 ListFeatures是一个服务器端的流式RPC，所以我们需要将多个Feature发回给客户端。

```golang

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
	for _, feature := range s.savedFeatures {
		if inRange(feature.Location, rect) {
			if err := stream.Send(feature); err != nil {
				return err
			}
		}
	}
	return nil
}
```

如你所见，这里的请求对象是一个 Rectangle，客户端期望从中找到 Feature，这次我们得到了一个请求对象和一个特殊的RouteGuide_ListFeaturesServer来写入我们的响应，而不是得到方法参数中的简单请求和响应对象。

在这个方法中，我们填充了尽可能多的 Feature 对象去返回，用它们的 Send() 方法把它们写入 RouteGuide_ListFeaturesServer。最后，在我们的简单 RPC中，我们返回了一个 nil 错误告诉 gRPC 响应的写入已经完成。如果在调用过程中发生任何错误，我们会返回一个非 nil 的错误；gRPC 层会将其转化为合适的 RPC 状态通过线路发送。

## 客户端流式 RPC

现在让我们看看稍微复杂点的东西：客户端流方法 RecordRoute，我们通过它可以从客户端拿到一个 Point 的流，其中包括它们路径的信息。如你所见，这次这个方法没有请求参数。相反的，它拿到了一个 RouteGuide_RecordRouteServer 流，服务器可以用它来同时读 和 写消息——它可以用自己的 Recv() 方法接收客户端消息并且用 SendAndClose() 方法返回它的单个响应。

```golang
func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
    var pointCount, featureCount, distance int32
    var lastPoint *pb.Point
    startTime := time.Now()
    for {
        point, err := stream.Recv()
        if err == io.EOF {
            endTime := time.Now()
            return stream.SendAndClose(&pb.RouteSummary{
                PointCount:   pointCount,
                FeatureCount: featureCount,
                Distance:     distance,
                ElapsedTime:  int32(endTime.Sub(startTime).Seconds()),
            })
        }
        if err != nil {
            return err
        }
        pointCount++
        for _, feature := range s.savedFeatures {
            if proto.Equal(feature.Location, point) {
                featureCount++
            }
        }
        if lastPoint != nil {
            distance += calcDistance(lastPoint, point)
        }
        lastPoint = point
    }
}
```

在方法体中，我们使用 RouteGuide_RecordRouteServer 的 Recv() 方法去反复读取客户端的请求到一个请求对象（在这个场景下是 Point），直到没有更多的消息：服务器需要在每次调用后检查 Read() 返回的错误。如果返回值为 nil，流依然完好，可以继续读取；如果返回值为 io.EOF，消息流结束，服务器可以返回它的 RouteSummary。如果它还有其它值，我们原样返回错误，gRPC 层会把它转换为 RPC 状态。

## 双向流式 RPC

最后，让我们看看双向流式 RPC RouteChat()。

```golang
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        key := serialize(in.Location)
                ... // look for notes to be sent to client
        for _, note := range s.routeNotes[key] {
            if err := stream.Send(note); err != nil {
                return err
            }
        }
    }
}
```

这次我们得到了一个 RouteGuide_RouteChatServer 流，和我们的客户端流的例子一样，它可以用来读写消息。但是，这次当客户端还在往 它们 的消息流中写入消息时，我们通过方法的流返回值。

这里读写的语法和客户端流方法相似，除了服务器会使用流的 Send() 方法而不是 SendAndClose()，因为它需要写多个响应。虽然客户端和服务器端总是会拿到对方写入时顺序的消息，它们可以以任意顺序读写——流的操作是完全独立的


## 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们RouteGuide服务中实现的过程：

```golang
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
grpcServer := grpc.NewServer()
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
... // determine whether to use TLS
grpcServer.Serve(lis)
```

1. 使用 lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port)) 指定我们期望客户端请求的监听端口。
2. 使用grpc.NewServer()创建 gRPC 服务器的一个实例。
3. 在 gRPC 服务器注册我们的服务实现。
4. 用服务器 Serve() 方法以及我们的端口信息区实现阻塞等待，直到进程被杀死或者 Stop() 被调用。


## 创建客户端

在这部分，我们将尝试为 RouteGuide 服务创建一个 Go 的客户端。你可以从grpc-go/examples/route_guide/client/client.go看到我们完整的客户端例子代码.


### 创建存根

为了调用服务方法，我们首先创建一个 gRPC conn 和服务器交互。我们通过给 grpc.Dial() 传入服务器地址和端口号做到这点，如下：

```golang
conn, err := grpc.Dial(*serverAddr)
if err != nil {
    ...
}
defer conn.Close()
```

你可以使用 DialOptions 在 grpc.Dial 中设置授权认证（如， TLS，GCE认证，JWT认证），如果服务有这样的要求的话 —— 但是对于 RouteGuide 服务，我们不用这么做。

一旦 gRPC conn 建立起来，我们需要一个客户端 存根 去执行 RPC。我们通过 .proto 生成的 pb 包提供的 NewRouteGuideClient 方法来完成。

```golang
client := pb.NewRouteGuideClient(conn)
```

### 调用服务方法

现在让我们看看如何调用服务方法。注意，在 gRPC-Go 中，RPC以阻塞/同步模式操作，这意味着 RPC 调用等待服务器响应，同时要么返回响应，要么返回错误。

#### 简单 RPC

调用简单 RPC GetFeature 几乎是和调用一个本地方法一样直观。

```golang
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
        ...
}
```

如你所见，我们调用了前面创建的存根上的方法。在我们的方法参数中，我们创建并且填充了一个请求的 protocol buffer 对象（例子中为 Point）。我们同时传入了一个 context.Context ，在有需要时可以让我们改变 RPC 的行为，比如超时/取消一个正在运行的 RPC。 如果调用没有返回错误，那么我们就可以从服务器返回的第一个返回值中读到响应信息。

```golang
log.Println(feature)
```

#### 服务器端流式 RPC

ListFeatures 就是我们说的服务器端流方法，它会返回地理的Feature 流。 如果你已经读过创建服务器，本节的一些内容也许看上去会很熟悉——流式 RPC 是在客户端和服务器两端以一种类似的方式实现的。

```golang
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
stream, err := client.ListFeatures(context.Background(), rect)
if err != nil {
    ...
}
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}
```

在简单 RPC 的例子中，我们给方法传入一个上下文和请求。然而，我们得到返回的是一个 RouteGuide_ListFeaturesClient 实例，而不是一个应答对象。客户端可以使用 RouteGuide_ListFeaturesClient 流去读取服务器的响应。

我们使用 RouteGuide_ListFeaturesClient 的 Recv() 方法去反复读取服务器的响应到一个响应 protocol buffer 对象（在这个场景下是Feature）直到消息读取完毕：每次调用完成时，客户端都要检查从 Recv() 返回的错误 err。如果返回为 nil，流依然完好并且可以继续读取；如果返回为 io.EOF，则说明消息流已经结束；否则就一定是一个通过 err 传过来的 RPC 错误。


#### 客户端流式 RPC

除了我们需要给方法传入一个上下文而后返回 RouteGuide_RecordRouteClient 流以外，客户端流方法 RecordRoute 和服务器端方法类似，它可以用来读 和 写消息。

```golang
// Create a random number of random points
r := rand.New(rand.NewSource(time.Now().UnixNano()))
pointCount := int(r.Int31n(100)) + 2 // Traverse at least two points
var points []*pb.Point
for i := 0; i < pointCount; i++ {
    points = append(points, randomPoint(r))
}
log.Printf("Traversing %d points.", len(points))
stream, err := client.RecordRoute(context.Background())
if err != nil {
    log.Fatalf("%v.RecordRoute(_) = _, %v", client, err)
}
for _, point := range points {
    if err := stream.Send(point); err != nil {
        log.Fatalf("%v.Send(%v) = %v", stream, point, err)
    }
}
reply, err := stream.CloseAndRecv()
if err != nil {
    log.Fatalf("%v.CloseAndRecv() got error %v, want %v", stream, err, nil)
}
log.Printf("Route summary: %v", reply)
```

RouteGuide_RecordRouteClient 有一个 Send() 方法，我们可以用它来给服务器发送请求。一旦我们完成使用 Send() 方法将客户端请求写入流，就需要调用流的 CloseAndRecv()方法，让 gRPC 知道我们已经完成了写入同时期待返回应答。我们从 CloseAndRecv() 返回的 err 中获得 RPC 的状态。如果状态为nil，那么CloseAndRecv()的第一个返回值将会是合法的服务器应答。


#### 双向流式 RPC

最后，让我们看看双向流式 RPC RouteChat()。 和 RecordRoute 的场景类似，我们只给函数传
入一个上下文对象，拿到可以用来读写的流。但是，当服务器依然在往 他们 的消息流写入消息时，我们
通过方法流返回值。

```golang
stream, err := client.RouteChat(context.Background())
waitc := make(chan struct{})
go func() {
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            // read done.
            close(waitc)
            return
        }
        if err != nil {
            log.Fatalf("Failed to receive a note : %v", err)
        }
        log.Printf("Got message %s at point(%d, %d)", in.Message, in.Location.Latitude, in.Location.Longitude)
    }
}()
for _, note := range notes {
    if err := stream.Send(note); err != nil {
        log.Fatalf("Failed to send a note: %v", err)
    }
}
stream.CloseSend()
<-waitc
```

这里读写的语法和我们的客户端流方法很像，除了在完成调用时，我们会使用流的 CloseSend() 方法。
虽然每一端获取对方信息的顺序和信息被写入的顺序一致，客户端和服务器都可以以任意顺序读写——流的操作是完全独立的。
