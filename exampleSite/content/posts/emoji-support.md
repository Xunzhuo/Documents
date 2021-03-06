---
title: 【gRPC详解】从RPC到gRPC
description: gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持。
toc: true
authors:
  - Example Author
tags:
  - gRPC
  - RPC
categories: RPC框架
series:
date: '2019-03-05'
lastmod: '2019-03-05'
draft: false
---

![image-20210222110839407](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222110839407.png)

## **gRPC简介**

> gRpc官网地址：[https://www.grpc.io](https://www.grpc.io/)

**gRPC is a modern open source high performance RPC framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services.**

gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持。

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特性。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。

gRPC是一款RPC框架，在全篇开始之前，我们一起先了解一下**RPC**是什么？

![image-20210222121911485](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222121911485.png)

## RPC简介

**RPC（Remote Procedure Call）远程过程调用**，是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议，简单的理解是一个节点请求另一个节点提供的服务。

RPC只是一套协议，基于这套协议规范来实现的框架都可以称为 RPC 框架，比较典型的有 Dubbo、Thrift 和 gRPC。



## **RPC的特点**
RPC 是远程过程调用的方式之一，涉及调用方和被调用方两个进程的交互。

因为 RPC 提供类似于本地方法调用的形式，所以对于调用方来说，**调用 RPC 方法和调用本地方法并没有明显区别**。



## **RPC的机制的诞生和基础概念**

1984 年，Birrell 和 Nelson 在 ACM Transactions on Computer Systems 期刊上发表了名为[“Implementing remote procedure calls”](https://dl.acm.org/doi/pdf/10.1145/2080.357392)的论文，该文对 RPC 的机制做了经典的诠释：

RPC 远程过程调用是指计算机 A 上的进程，调用另外一台计算机 B 上的进程的方法。其中A 上面的调用进程被挂起，而 B 上面的被调用进程开始执行对应方法，并将结果返回给 A，计算机 A 接收到返回值后，调用进程继续执行。

发起 RPC 的进程通过参数等方式将信息传送给被调用方，然后被调用方处理结束后，再通过返回值将信息传递给调用方。

这一过程对于开发人员来说是透明的，开发人员一般也无须知道双方底层是如何进行消息通信和信息传递的，这样可以让业务开发人员更专注于业务开发，而非底层细节。

RPC 让程序之间的远程过程调用具有与本地调用类似的形式。比如说某个程序需要读取某个文件的数据，开发人员会在代码中执行 read 系统调用来获取数据。

**当 read 实际是本地调用时**，read 函数由链接器从依赖库中提取出来，接着链接器会将它链接到该程序中。虽然 read 中执行了特殊的系统调用，但它本身依然是通过将参数压入堆栈的常规方式调用的，调用方并不知道 read 函数的具体实现和行为。

**当 read 实际是一个远程过程时**（比如调用远程文件服务器提供的方法），调用方程序中需要引入 read 的接口定义，称为客户端存根（client-stub）。远程过程 read 的客户端存根与本地方法的 read 函数类似，都执行了本地函数调用。

不同的是它底层实现上不是进行操作系统调用读取本地文件来提供数据，而是将参数打包成网络消息，并将此网络消息发送到远程服务器，交由远程服务执行对应的方法，在发送完调用请求后，客户端存根随即阻塞，直到收到服务器发回的响应消息为止。

下图展示了远程方法调用过程中的客户端和服务端各个阶段的操作：
　　　　

![image-20210222121655049](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222121655049.png)

如上图所示：一个完整的RPC架构里面包含了**四个核心的组件**：

分别是**Client**，**Client Stub**，**Server**以及**Server Stub**，这个Stub可以理解为存根。

- 客户端(Client)，服务的调用方。
- 客户端存根(Client Stub)，存放服务端的地址消息，再将客户端的请求参数打包成网络消息，然后通过网络远程发送给服务方。
- 服务端(Server)，真正的服务提供者。
- 服务端存根(Server Stub)，接收客户端发送过来的消息，将消息解包，并调用本地的方法。

当客户端发送请求的网络消息到达服务器时，服务器上的网络服务将其传递给服务器存根（server-stub）。服务器存根与客户端存根一一对应，是远程方法在服务端的体现，用来将网络请求传递来的数据转换为本地过程调用。服务器存根一般处于阻塞状态，等待消息输入。

当服务器存根收到网络消息后，服务器将方法参数从网络消息中提取出来，然后以常规方式调用服务器上对应的实现过程。从实现过程角度看，就好像是由客户端直接调用一样，参数和返回地址都位于调用堆栈中，一切都很正常。

实现过程执行完相应的操作，随后用得到的结果设置到堆栈中的返回值，并根据返回地址执行方法结束操作。以 read 为例，实现过程读取本地文件数据后，将其填充到 read 函数返回值所指向的缓冲区。

read 过程调用完后，实现过程将控制权转移给服务器存根，它将结果（缓冲区的数据）打包为网络消息，最后通过网络响应将结果返回给客户端。网络响应发送结束后，服务器存根会再次进入阻塞状态，等待下一个输入的请求。

客户端接收到网络消息后，客户操作系统会将该消息转发给对应的客户端存根，随后解除对客户进程的阻塞。客户端存根从阻塞状态恢复过来，将接收到的网络消息转换为调用结果，并将结果复制到客户端调用堆栈的返回结果中。当调用者在远程方法调用 read 执行完毕后重新获得控制权时，它唯一知道的是 read 返回值已经包含了所需的数据，但并不知道该 read 操作到底是在本地操作系统读取的文件数据，还是通过远程过程调用远端服务读取文件数据。

**总结一下RPC调用过程**

![img](https://pic1.zhimg.com/80/v2-1ae91aa17808a65ac34b419355cf9b40_720w.jpg)



1. 客户端（client）以本地调用方式（即以接口的方式）调用服务；
2. 客户端存根（client stub）接收到调用后，负责将方法、参数等组装成能够进行网络传输的消息体（将消息体对象序列化为二进制）；
3.  客户端通过sockets将消息发送到服务端；
4.  服务端存根( server stub）收到消息后进行解码（将消息对象反序列化）；
5. 服务端存根( server stub）根据解码结果调用本地的服务；
6. 本地服务执行并将结果返回给服务端存根( server stub）；
7. 服务端存根( server stub）将返回结果打包成消息（将结果消息对象序列化）；
8. 服务端（server）通过sockets将消息发送到客户端；
9. 客户端存根（client stub）接收到结果消息，并进行解码（将结果消息发序列化）；
10. 客户端（client）得到最终结果。

RPC的目标是要把2、3、4、7、8、9这些步骤都封装起来。

注意：无论是何种类型的数据，最终都需要转换成二进制流在网络上进行传输，数据的发送方需要将对象转换为二进制流，而数据的接收方则需要把二进制流再恢复为对象。



## **RPC框架的组成**

一个完整的 RPC 框架包含了服务注册发现、负载、容错、序列化、协议编码和网络传输等组件。

不同的 RPC 框架包含的组件可能会有所不同，但是一定都包含 RPC 协议相关的组件，RPC 协议包括**序列化、协议编解码器和网络传输栈**，如下图所示：

　　　　

![image-20210222121827576](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222121827576.png)

 

RPC 协议一般分为公有协议和私有协议。例如，HTTP、SMPP、WebService 等都是公有协议。如果是某个公司或者组织内部自定义、自己使用的，没有被国际标准化组织接纳和认可的协议，往往划为私有协议，例如 Thrift 协议和蚂蚁金服的 Bolt 协议。

分布式架构所需要的企业内部通信模块，往往采用私有协议来设计和研发。相较公有协议，私有协议虽然有很多弊端，比如在通用性上、公网传输的能力上，但是高度定制化的私有协议可以最大限度地降低成本，提升性能，提高灵活性与效率。定制私有协议，可以有效地利用协议里的各个字段，灵活满足各种通信功能需求。

在协议设计上，你还需要考虑以下三个关键问题：

1. 协议包括的必要字段与主要业务负载字段。协议里设计的每个字段都应该被使用到，避免无效字段。
2.  通信功能特性的支持。比如，CRC 校验、安全校验、数据压缩机制等。
3. 协议的升级机制。毕竟是私有协议，没有长期的验证，字段新增或者修改，是有可能发生的，因此升级机制是必须考虑的。



## **RPC和HTTP区别**
RPC 和 HTTP都是微服务间通信较为常用的方案之一，其实RPC 和 HTTP 并不完全是同一个层次的概念，它们之间还是有所区别的。

RPC 是远程过程调用，其调用协议通常包括序列化协议和传输协议。序列化协议有基于纯文本的 XML 和 JSON、二进制编码的Protobuf和Hessian。传输协议是指其底层网络传输所使用的协议，比如 TCP、HTTP。

可以看出HTTP是RPC的传输协议的一个可选方案，比如说 gRPC 的网络传输协议就是 HTTP。HTTP 既可以和 RPC 一样作为服务间通信的解决方案，也可以作为 RPC 中通信层的传输协议（此时与之对比的是 TCP 协议）。

 

## **常见的 PRC 框架**
目前流行的开源 RPC 框架还是比较多的，有阿里巴巴的 Dubbo、Google 的 gRPC、Facebook 的 Thrift 和 Twitter 的 Finagle 等。

    1. Go RPC：Go 语言原生支持的 RPC 远程调用机制，简单便捷。
    2. gRPC：Google 发布的开源 RPC 框架，是基于 HTTP 2.0 协议的，并支持众多常见的编程语言，它提供了强大的流式调用能力，目前已经成为最主流的 RPC 框架之一。
    3. Thrift：Facebook 的开源 RPC 框架，主要是一个跨语言的服务开发框架，作为老牌开源 RPC 协议，以其高性能和稳定性成为众多开源项目提供数据的方案选项。

## gRPC特点

在gRPC的客户端应用可以想调用本地对象一样直接调用另一台不同的机器上的服务端的应用的对象或者方法，这样在创建分布式应用的时候更容易。下面看看gRPC的特点：

1. 语言无关，支持多种语言；

2. 基于 IDL 文件定义服务，通过 proto3 工具生成指定语言的数据结构、服务端接口以及客户端 Stub；

3. 通信协议基于标准的 HTTP/2 设计，支持双向流、消息头压缩、单 TCP 的多路复用、服务端推送等特性，这些特性使得 gRPC 在移动端设备上更加省电和节省网络流量；

4. 序列化支持 PB（Protocol Buffer）和 JSON，PB 是一种语言无关的高性能序列化框架，基于 HTTP/2 + PB, 保障了 RPC 调用的高性能。


## **gRPC使用说明**

 

gRPC使用和上面RPC使用方法类似，首先定义服务，指定其能够被远程调用的方法，包括参数和返回类型，这里使用protobuf来定义服务。

在服务端实现定义的服务接口，并运行一个gRPC服务器来处理客户端调用。

　　　　![img](https://img2020.cnblogs.com/blog/824470/202011/824470-20201101223910108-256870299.png)

## gRPC安装

使用git下载：

```shell
git clone https://github.com/grpc/grpc-go.git $GOPATH/src/google.golang.org/grpc
git clone https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/textgit clone https://github.com/google/go-genproto.git $GOPATH/src/google.golang.org/genprot
```

 因为gRPC要使用proto及相关依赖，请先安装protobuf

 都安装好之后，可以测试gRPC是否可以正常运行。

```shell
#进入服务端 （先启动）
cd /Users/bitliu/go/src/google.golang.org/grpc/examples/helloworld/greeter_server
#进入客户端端 （服务端启动后在启动）
cd /Users/bitliu/go/src/google.golang.org/grpc/examples/helloworld/greeter_client
```

 运行结果，输出hello world表面可以通信。

#### 客户端：

![image-20210222120500173](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222120500173.png)

#### **服务端：**

![image-20210222120439611](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222120439611.png)

## gRPC四种通信方式

1、 **Simple RPC**   简单rpc
这就是一般的rpc调用，一个请求对象对应一个返回对象

```protobuf
rpc simpleHello(Person) returns (Result) {}
```

2、 **Server-side streaming RPC**   服务端流式RPC
一个请求对象，服务端可以传回多个结果对象

```protobuf
rpc serverStreamHello(Person) returns (stream Result) {}
```

3、 **Client-side streaming RPC**   客户端流式RPC
客户端传入多个请求对象，服务端返回一个响应结果

```protobuf
rpc clientStreamHello(stream Person) returns (Result) {}
```

4、 **Bidirectional streaming RPC**   双向流式RPC
结合客户端流式rpc和服务端流式rpc，可以传入多个对象，返回多个响应对象

```protobuf
rpc biStreamHello(stream Person) returns (stream Result) {}
```

##  **gRPC案例**

下面使用gRPC来实现一个简单的客户端和服务端的通信

本地代码结构：

![image-20210222115959852](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222115959852.png)

　　 1. 先使用protobuf定义服务。

​     创建myProtobuf.proto文件，编辑如下内容。

```protobuf
syntax = "proto3" ;

option go_package = ".;myproto";

//定义服务 
service HelloServer {
  rpc SayHello (HelloReq) returns (HelloRsp){}
  rpc SayName (NameReq) returns (NameRsp){}
}

//客户端发送给服务端
message HelloReq {
  string name = 1 ;
}

//服务端返回给客户端
message HelloRsp {
  string msg = 1 ;
}

//客户端发送给服务端
message NameReq {
  string name = 1 ;
}

//服务端返回给客户端
message NameRsp {
  string msg = 1 ;
}
```

 定义了两个服务SayHello，SayName及对应的四个消息（message）。

 然后在执行命令生成pd.go文件

```
protoc --go_out=plugins=grpc:./ *.proto     #添加grpc插件
```

2. 编写服务端server.go

> 1. 实现protobuf中的service接口
> 2. 监听端口
> 3. 创建grpc服务
> 4. 注册服务

```go
package main


import (
	pd "../myproto" //导入proto
	"context"
	"fmt"
	"google.golang.org/grpc"
	"net"
)

type server struct {}

func (this *server) SayHello(ctx context.Context, in *pd.HelloReq) (out *pd.HelloRsp,err error) {
	return &pd.HelloRsp{Msg:"Hello"}, nil
}

func (this *server) SayName(ctx context.Context, in *pd.NameReq) (out *pd.NameRsp,err error){
	return &pd.NameRsp{Msg:in.Name + "  is my name"}, nil
}

func main()  {
	ln, err := net.Listen("tcp", ":10089")
	if err != nil {
		fmt.Println("network error", err)
	}

	//创建grpc服务
	srv := grpc.NewServer()
	//注册服务
	pd.RegisterHelloServerServer(srv, &server{})
	err = srv.Serve(ln)
	if err != nil {
		fmt.Println("Serve error", err)
	}
}
```

3. 编写客户端client.go

> 1. 连接服务端
> 2. 获得句柄
> 3. 远程调用函数

```go
package main

import (
	pd "../myproto" //导入proto
	"context"
	"fmt"
	"google.golang.org/grpc"
)

func main() {
	//客户端连接服务端
	conn, err := grpc.Dial("127.0.0.1:10089", grpc.WithInsecure())
	if err != nil {
		fmt.Println("network error", err)
	}

	//网络延迟关闭
	defer  conn.Close()
	//获得grpc句柄
	c := pd.NewHelloServerClient(conn)
	//通过句柄进行调用服务端函数SayHello
	re1, err := c.SayHello(context.Background(),&pd.HelloReq{Name:"bitliu"})
	if err != nil {
		fmt.Println("calling SayHello() error", err)
	}
	fmt.Println(re1.Msg)

	//通过句柄进行调用服务端函数SayName
	re2, err := c.SayName(context.Background(),&pd.NameReq{Name:"BitLiu"})
	if err != nil {
		fmt.Println("calling SayName() error", err)
	}
	fmt.Println(re2.Msg)
}
```

运行结果如下

#### 服务端

![image-20210222115803165](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222115803165.png)

#### 客户端

> 从服务端返回了信息

![image-20210222115818238](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210222115818238.png)

## 结语

本文介绍了gRPC的相关内容，有很多地方还需继续深入，以后希望能够更深入的讨论这块知识！

下一篇文章就是讲解：

基于gRPC的Envoy动态下发配置的底层原理，我会从go-control-plane源码上一探究竟~
