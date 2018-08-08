---
tags: [go, 框架]
date: 2018-08-08
---

# 在Go语言中，你已经不需要任何框架

> 经常有做Java的同事问我，说是习惯了 Java的 Spring Cloud生态， 在Go语言下，有没有这种大一统的开发框架呢，答案是肯定的。

得益于Go语言的简单自由， Go语言的发展初期，出现过很多我们熟悉的Web开发框架，你可能听说 mux，gin，beego等，在微服务大行其道之时，诞生了 go-micro，go-kit 等， 当然这些框架都是比较优秀的框架，也在很大程度上帮助企业快速上手Go语言开发。这些框架提供的功能主要有：

1. Restful API 路由
2. 基于HTTP/2 JOSN的RPC通讯
3. 服务的发现和治理

但是，时至今日，在云端k8s，Service Mesh大行其道之时，Go语言的生态也发生了翻天覆地的变化，很多之前优秀的开发框架逐渐被替代，就目前看来，在Go语言开发中，已经形成了以Google为核心的开发体系，这种模式以决定性的优势让众多的早期框架甚至其他Web开发形式变得暗淡失色。

1. GRPC （负责云端业务接口的开发）
2. GrpcGateway (可以在几乎不修改业务代码的情况下，将你的Proto Service直接转换为 HTTP Restful Web API)
3. 各种ES6的前端单页面框架负责UI渲染
4. Istio (负责Service Mesh的微服务相关工作)

这种开发模式，虽然其他语言也可以实现GRPC相关功能， 但是得益于整个生态都是围绕着Go语言而展开的，在go语言中，你可以做到以下几点。

* GRPC的Service直接在进程内转换为 Restful API，而无需外层代理，也不用担心 JSON->PB-> Struct 的性能损耗，你可以像之前使用Srping Cloud 那样使用GRPC生态，如果可能，你的服务甚至不用监听任何GRPC端口，你可以像写GRPC那样去编写一套“工整的，完美的” Restful API。 

代码示例：

```protobuf
service Say {
    rpc Hello (Request) returns (Response) {
        option (google.api.http) = {
        	//将你的GRPC转换为Restful API 的URL匹配
			post: "/v1/greeter/hello/{id}"
			body:"*"
		};
    }
}
```

```go
//sayServer 业务的具体实现类
srv := &sayServer{}
mux := grpcmux.NewServeMux()
//注册到GO的HTTP MUX
pb.RegisterSayServerHandlerClient(context, mux.ServeMux, srv)
http.ListenAndServe(":8080", mux)
```

```bash
curl -X POST -d '{}' http://127.0.0.1:8080/v1/greeter/hello/12
```

值得一提的是，在这种模式下，GRPC支持将错误返回和Response解耦，错误不属于Reponse中的一个字段，错误属于独立的错误对象，你可以在通过proto文件定义自己的业务错误描述：

例如：调用成功返回【200】：

```json
{
    "name":"Lee"
}
```

调用失败则返回【400等】：

```json
{
  "error": "错误描述",
  "code": 2
}
```

Google官方也对常见的错误处理做了定义， 具体可以参考：【GRPC标准错误定义】：https://nanxi.li/#/page/%E5%BC%80%E5%8F%91/grpc/%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86

* Docker Scratch 自从Docker诞生以来， Go语言就以决定性的优势作为微服务开发的主要语言，一个重要的特征就是Scratch， 这种Docker模式下，允许你的程序以极简的形式发布，不用添加任何系统动态库和依赖，你的Docker镜像就是你的程序文件。 这种模式在其他语言中虽然也可以实现，但是大部分情况下，由于历史原因，我们无法保证它能健康地运行在没有任何依赖的linux环境下， 而go语言中，Scratch 几乎成了 标准。

代码地址：https://github.com/ti/noframe/tree/master/grpcmux/_exmaple

* Istio 一种sidecar形式的服务治理方案，于Go语言架构无关，在这里不做赘述，后期会专门介绍上手教程

## 其他参考：

代码地址：https://github.com/ti/noframe/tree/master/grpcmux/_exmaple

GrpcGateway： https://github.com/grpc-ecosystem/grpc-gateway

Istio: https://istio.io/zh/









