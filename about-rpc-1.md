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



这个其实不重要，我最好奇的就是，`apigateway` 是按照什么规则进行转发的呢？如果说我们新加了一些路由，`apigateway` 怎么感知呢？



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
                 "Disk_size", "Status", "Fs_format",
                 "Disk_type", "Disk_format", "Is_public",
                 "Guest_count", "Storage_type",
                 "Zone", "Device", "Guest",
                 "Guest_id", "Created_at"},
        []string{"Storage", "Tenant"})

    modules.RegisterCompute(&Disks)
}
```

通过这种方式，注册到全局变量中，后续根据请求的路径查询对应的模块。

> 值得一提的是，`apigateway` 中使用的全部是正则路由。

所以，我们每次写新功能的时候都需要额外重启网关服务，个人感觉不是那么的优雅。



那么问题来了，rpc 能否解决这个问题？我写这篇文章的时候并不知道答案。



## rpc 是什么呢？

虽然没用过微服务，但是听的耳朵都要起茧子了。这玩意应该没那么普及吧..

我认为 rpc 框架主要解决的是服务间通信的问题，如果每个服务都使用注册 handler 的方式实现，这个工作量很大并且很难维护。正如 rpc 名字的意思，调用远程接口和调用本地函数一样，rpc 框架封装了这些细节。



另一方面，rpc 通信摒弃了常规的序列化方式，《DDIA》 中有讲到几种压缩方式的对比。



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

可以看到，我们在客户端通过调用函数的方式就实现了对服务端的调用。



但是，重点也不是这些，这些内容大部分是通过 `.proto` 文件生成的，我们只是实现了相应的接口。



## .proto 文件

Google 官网的解释 [[在这里](https://developers.google.com/protocol-buffers/docs/gotutorial)]。

常用的几种数据类型：

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



## 思考

那么我们是否可以通过 rpc 的方式重写那个框架中的内容呢？答案是肯定的，但是貌似仍然解决不了两个服务都重启的问题。



或许可以通过自动的服务发现进行实现，下篇文章的内容有了..
