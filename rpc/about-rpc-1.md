---
title: 关于 rpc 的那些事（一）
comments: true
---

该文是我学习 `rpc` 过程中的总结，初步了解到一个 `rpc ` 框架是做什么的，以及为什么我们要使用 `rpc ` 框架。结合当前工作中遇到的场景，对比思考。

![](https://s2.loli.net/2022/06/22/8kZFXOu7yo6nfLJ.png)

<!--more-->

## 我遇到的

我们现在的后端服务，跑的是一套开源的云管代码 [[在这里](https://github.com/yunionio/cloudpods)]，据不可靠消息透露，这套框架的源头是美团搞得，一开始貌似用的 Java 代码，后来用 Go 重写了。



八卦了一下，言归正传，这套代码主要由 keystone / apigateway / region / scheduler 组成，简单概述下他们的职责。

1. keystone 负责认证。
2. apigateway 负责路由转发。
3. region 负责实际的业务逻辑。
4. scheduler 负责调度，根据库里数据选择某个最优解。

**网络请求是怎么进行流转的呢？**

认证我们抛开不说，假设以下内容都是认证完成后的。



认证完成后，用户需要创建云主机，从页面点击创建后，请求走到 `apigateway`，`apigateway` 分析请求路径，找到对应的 `model`，进行路由转发，请求 `region`，`region` 处理完成后，返回给 `apigateway`，`apigateway` 拿到响应后返回给客户结果。



**这里是怎么拿到另一个服务的地址呢？**

将所有服务信息存储在 endpoint 表中，然后根据请求的相关信息拿到对应的 url，再把 path 拼上。



这个其实不重要，我最好奇的就是，新加了一些路由之后，`apigateway` 怎么感知呢？



调试代码后发现，`region` 服务和 `apigateway` 服务存在着某些程度上的耦合，我们在 `region` 中添加路由后，必须要“同步”给网关。在网关中添加个文件，如下：

```go
var (
    Disks modulebase.ResourceManager
)

func init() {
    Disks = modules.NewComputeManager(
        "disk",
        "disks",
        []string{"ID", "Name", "Billing_type",
                 "Guest_id", "Created_at"},
        []string{"Storage", "Tenant"})

    modules.RegisterCompute(&Disks)
}
```

通过这种方式，注册到全局变量中，后续根据请求的路径查询对应的模块。

> 值得一提的是，`apigateway` 中使用的是正则路由。这样就可以根据请求方法进行抽象，所有的 Get 请求走的都是同一个 handler，其他也类似。

所以，我们每次写新功能的时候都需要额外重启网关服务，个人感觉不是那么的优雅。



那么问题来了，rpc 能否解决这个问题？我写这篇文章的时候并不知道答案。



## rpc 是什么呢？

虽然没用过微服务，但是听的耳朵都要起茧子了。这玩意应该没那么普及吧..

我认为 rpc 框架主要解决的是服务间调用的问题，如果每个接口都使用注册 handler 的方式实现，这个工作量很大并且很难维护。正如 rpc 名字的意思，调用远程接口和调用本地函数一样，rpc 框架封装了这些细节。



另一方面，grpc 通信摒弃了常规的序列化方式，《DDIA》 中有讲到几种压缩方式的对比。



看一个简单的例子就知道这个调用过程大致长什么样了，例子来自 [[grpc-go](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) ]

```go
// client
func main() {
    flag.Parse()
    // Set up a connection to the server.
    conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // Contact the server and print out its response.
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    log.Printf("Greeting: %s", r.GetMessage())
}
```

```go
// server
// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

可以看到，我们在客户端通过调用函数的方式就实现了对服务端的调用。但是，重点也不是这些，这些内容大部分是通过 `.proto` 文件生成的，我们只是实现了相应的接口。



## .proto 文件

生成 `.pb.go | _grpc.pb.go` 命令：

` protoc --go_out=. --go_opt=paths=source_relative  --go-grpc_out=.  --go-grpc_opt=paths=source_relative .\forGen.proto`



Google 官网的解释 [[在这里](https://developers.google.com/protocol-buffers/docs/gotutorial)]，常用的几种数据类型：

- repeated
- map<string, int>
- oneof 结构体里面套了一个接口
- 单个结构体类型，`message Foo {}` 

```go
message FooRepeated {
    // []string
    repeated string Address = 1;
}

message FooMap {
    // map[string]int32
    map<string, int32> info = 1;
}

message FooOneof {
    // struct {interface{}}
    oneof avatar {
        string image_url = 1;
        bytes image_data = 2;
    }
}
```



## grpc 流

从文档中不难发现，grpc 支持流式数据传输，流式传输应对的是上传和下载大量数据的场景。

```go
// 双向流，stream指定启用流特性
service HelloService {
    rpc Channel (stream Foo) returns (stream Foo);
}
```

和普通的差别也不大，只是从直接接收参数变成了，读取连接里的内容。我们实现的 Channel 方法如下：

```go
type HelloServiceImpl struct {
    // 这就好像，pb 插件给我们提供一个抽象类
    // 我们自己去实现具体的内容
    pb.UnimplementedHelloServiceServer
}

func (p *HelloServiceImpl) Channel(stream pb.HelloService_ChannelServer) error {
    for {
        // 读取连接中的内容，
        // 读出来的最小单位就是我们在 service 中定义的
        args, err := stream.Recv()
        if err != nil {
            if err == io.EOF {
                return nil
            }
            return err
        }
        // 输出从客户端都到的东西
        fmt.Println(args.GetName())
        reply := &pb.Foo{Name: args.GetName()}

        err = stream.Send(reply)
        if err != nil {
            return err
        }
    }
}
```

其余 server 端的代码类似，注册函数，启动服务。这里看下 client 实现：

```go
conn, err := grpc.Dial(":1234", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

client := pb.NewHelloServiceClient(conn)
s, err := client.Channel(context.Background())
if err != nil {
    log.Fatal(err)
}

for {
    // 每隔一秒给 server 发个消息
    err := s.Send(&pb.Foo{Name: "bqyang-test"})
    if err != nil {
        log.Println(err)
    }

    time.Sleep(time.Second)
}
```

这样既可实现 grpc 以流的方式通信。



## 思考

那么我们是否可以通过 rpc 的方式重写那个框架中的内容呢？答案是肯定的，可以通过基于HTTP的 rpc 服务，但是貌似仍然解决不了两个服务都重启的问题，而且维护路由的成本更高了。

```go
func AServer() {
    rpc.RegisterName("HelloService", new(HelloService))
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request){
        var conn io.ReadWriteCloser = struct {
            io.Writer
            io.ReadCloser
        } {
            Writer: w, 
            ReadCloser: r.Body,
        }
        
        rpc.ServeRequest(jsonrpc.NewServerCodec(conn))
    })
    http.ListenAndServe(":1234", nil)
}
```



有没有更优解呢？或许可以通过服务发现进行实现，下篇文章的内容有了，了解下服务发现。
