---
title: grpcurl
date: 2023-08-27 13:41:39
tags: 工具 网络
---
## cURL
curl是一个常见的命令行工具，用来请求web服务器接口，c就是client的意思。
curl支持的协议很多
- http
- https
- FTP
- SCP
- SMTP
- Websockect

其使用方法可以参考这里： https://www.ruanyifeng.com/blog/2019/09/curl-reference.html

但是grpc是不支持的.

## grpcurl
[grpcurl](https://github.com/fullstorydev/grpcurl)就是用来支持grpc的一个工具。
支持功能包括：
- 调用RPC接口，支持携带参数
- 可以根据proto列出来所有的服务
- 可以通过反射生成服务的描述

## 试用
### 环境搭建
- 安装go
- 安装grpcurl
- proto生成相关（非必须）
  这里就不赘述了。

### 启动grpc服务
首先本地起一个grpc服务，这里可以简单使用官方提供的模板。
```shell
git clone -b v1.57.0 --depth 1 https://github.com/grpc/grpc-go
cd grpc-go/examples/helloworld
```
修改服务，因为grpcurl需要服务器支持反射
```go
// 修改 grpc-go/examples/helloworld/greeter_server/main.go

s := grpc.NewServer()  
  
reflection.Register(s) // 添加这一行
```
然后启动服务
```shell
go run main.go

```

### 使用grpcurl

#### 列举服务
因为本地不是tls服务，需要加-plaintext
```shell
(base) PS C:\Users\fanlu> grpcurl -plaintext localhost:50051 list
grpc.reflection.v1.ServerReflection
grpc.reflection.v1alpha.ServerReflection
helloworld.Greeter

// 指定service
(base) PS C:\Users\fanlu> grpcurl -plaintext localhost:50051 list helloworld.Greeter
helloworld.Greeter.SayHello
```
#### 描述协议
```shell
(base) PS C:\Users\fanlu> grpcurl -plaintext localhost:50051 describe helloworld.Greeter.SayHello
helloworld.Greeter.SayHello is a method:
rpc SayHello ( .helloworld.HelloRequest ) returns ( .helloworld.HelloReply );
```

#### 调用协议
```shell
(base) PS C:\Users\fanlu> grpcurl -plaintext localhost:50051 helloworld.Greeter.SayHello
{
  "message": "Hello "
}
```
## 注意：要区分大小写！

更复杂的用法还没用过。
