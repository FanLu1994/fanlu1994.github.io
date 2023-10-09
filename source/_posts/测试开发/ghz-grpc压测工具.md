---
title: ghz-grpc压测工具
date: 2023-10-18 21:59:42
tags:
  - 工具
  - 网络
categories: 测试开发
---
# 介绍
## 配置
- 支持配置文件、命令行配置
- 可以指定proto源文件或者使用protoc生成的protoset描述文件
- 支持指定协议名
- 支持导入原始路径列表
- 可以指定秘钥信息
- 可以指定tls服务器名称
- 可以跳过tls
- 是否异步请求，不等待上一个请求完成
- 支持设置rps
- 支持指定加载进度
    - 即可以逐步提高rps到预设值
- 可以指定并发工作器数量
- 可以指定并发工作器添加策略
    - 即逐步增加工作线程
- 指定请求总数
- 指定超时时间，默认20，不限制则设为0
- 指定持续时间
- 指定停止策略：关闭、等待请求返回、忽略则忽略正在运行的请求
- 指定请求体，以json格式
- 指定请求体，以文件形式
- 制定请求体，以二进制形式或者二进制文件
- 指定元数据 ，json字符串或者文件
- 流式接口的传输测试
- ...
- 指定连接数，默认一个连接
- 指定初始连接超时时间
- 可以使用的cpu内核数
- 指定客户端负载策略

## 输出
- summary
- csv格式
- html格式
- json格式
- prometheus格式
- influxDB格式

## 实践
### 一个grpc服务demo
#### pb编写
```proto
syntax = "proto3";  
package pb;  
  
option go_package="./;pb";  
  
message HelloRequest{  
  string name = 1;  
  int32 id = 2;  
}  
  
message HelloReply{  
  string name = 1;  
  string message = 2;  
}  
  
  
service HelloService{  
  rpc Hello(HelloRequest) returns (HelloReply);  
}
```
#### 生成go文件和服务端代码
```shell
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative *.proto
```
#### 写一个服务
```go
package main  
  
import (  
    "context"  
    "fmt"    "google.golang.org/grpc"    "grpc_demo/pb"    "log"    "net")  
  
const (  
    port = ":50051"  
)  
  
type myServer struct {  
    pb.UnimplementedHelloServiceServer  
}  
  
func (s *myServer) Hello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {  
    fmt.Println("hello receice message:", req)  
    rep := &pb.HelloReply{  
       Name:    req.Name,  
       Message: "你好呀",  
    }  
    return rep, nil  
}  
    
  
func main() {  
    lis, err := net.Listen("tcp", port)  
    if err != nil {  
       log.Fatalf("failed to listen: %v", err)  
    }  
  
    // 创建一个 gRPC 服务端实例  
    s := grpc.NewServer()  
  
    // 注册  
    pb.RegisterHelloServiceServer(s, &myServer{})  
  
    if err := s.Serve(lis); err != nil {  
       log.Fatalf("failed to serve: %v", err)  
    }  
}
```

### 压测实践
其中参数解释
- proto指定原型
- rps指定目标rps值
- total指定目标请求数
- call指定要压的方法名
- data指定请求体数据 json格式，也可以是数组，那么会轮流请求给定的请求体
- skipTLS insecure说明使用明文压测
- 最后是服务地址

结果可以看到
- 总结：总请求数、总消耗时间、最慢请求、最快请求、平均时间、QPS
- 响应时间直方图
- 百分比延迟分布
- 响应码分布

```shell
ghz --proto=hello.proto --rps=100 --total=100 --call pb.HelloService.Hello --data='{"name": "xiamu",       "id": 1}' --skipTLS --insecure  192.168.31.129:50051

Summary:
  Count:	100
  Total:	1.06 s
  Slowest:	497.68 ms
  Fastest:	4.33 ms
  Average:	189.71 ms
  Requests/sec:	94.43

Response time histogram:
  4.334   [1]  |∎∎
  53.669  [17] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  103.004 [19] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  152.339 [10] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  201.674 [10] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  251.008 [10] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  300.343 [11] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  349.678 [7]  |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  399.013 [5]  |∎∎∎∎∎∎∎∎∎∎∎
  448.348 [5]  |∎∎∎∎∎∎∎∎∎∎∎
  497.683 [5]  |∎∎∎∎∎∎∎∎∎∎∎

Latency distribution:
  10 % in 36.96 ms
  25 % in 67.50 ms
  50 % in 166.24 ms
  75 % in 286.13 ms
  90 % in 397.86 ms
  95 % in 448.03 ms
  99 % in 487.83 ms

Status code distribution:
  [OK]   100 responses


~/code/ghz_test ❯
```

## 原理
本质上还是go协程来实现的。
各个模块区分的很清晰：
- 请求模块
- 数据处理模块
- 日志模块
- 统计模块
- 输出模块
- 连接管理模块

如果用来做自己的压测工具，并且接入prometheus监控应该不需要花太大功夫。
此外如果将其改造为http的压测，应该也非难事。
## 未来
代码里面还包含一个ghz命令，不过还在开发中，可以期待一下。
