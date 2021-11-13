---
title: "Grpc"
date: 2021-08-08T16:04:19+08:00
draft: true
---
# 记录一次grpc的使用经历

这是我第一次听说grpc，发现原来是一个微服务之间的通讯协议，顿时来了兴趣。微服务很早之前就想搞了，但是无奈没有契机和方案来实现。token这边也没有使用微服务的项目，索性心一横，这次就用grpc开发公司的项目算了。于是我就这样上了grpc的道路。顺便吐槽一下，千万别用go-zero框架，他的文档超级烂）我们的项目使用的是grpc1.39，这个版本的grpc应该说是比较稳定的，在介绍项目之前先来聊聊啥是grpc吧。

> In gRPC, a client application can directly call a method on a server application on a different machine as if it were a local object, making it easier for you to create distributed applications and services. As in many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, the client has a stub (referred to as just a client in some languages) that provides the same methods as the server.

这是一段来自grpc官网的描述[link]()[Introduction to gRPC | gRPC](https://grpc.io/docs/what-is-grpc/introduction/)，描述了grpc是什么，简单来说就是一种服务间通信的方式，可以跨语言。只要大家都遵守一个proto协议就行。

![](https://grpc.io/img/landing-2.svg)

如图所示，grpc能够跨语言传输。（这个通讯协议真的有意思，以后的blog详解

因为我们的整体项目的go项目，所以使用了go的一体化框架go-zero，这个框架可以通过proto和api生成代码，可以说是go语言界的springboot了。因为整体代码几乎都是生成的，所以可定制化较低。比如持久层使用的是他定义好的orm（话说他的orm也和mybatis像的一匹）。不过这都难不倒我们，最让我奇怪的是他在生成代码之后，api层会和rpc层形成依赖关系，这样的话，我们部署的时候就很神秘了。因为一行代码的修改会使得整个微服务集群全部重启，这违背了初衷。不仅如此，gozero默认使用etcd进行服务发现，这个效率也是比较低的，在大型集群内部也会变得冗余。不过我们后来在issue和pr里面找到了直连的方案。（真的难顶，从issue和pr里面发现文档）在解决完所有的问题之后，我们正准备进行愉快的crud的时候，突然发现grpc协议里面好像不能传输结构体数据~*气*。只能是通过动态类型参数的形式传输，好吧，因为你要多语言兼容，我认了。最终我们的解决方案改成了将所有的数据打散直接进行传播，这样就不用考虑封装的方式。*（就是很不优雅，很烦）*

完成了项目的开发之后，mentor来检查的时候说："你们最好把链路追踪搞上去，要不然链路里面坏掉了都不知道发生了什么。"淦！咋还有链路追踪的活啊。于是我进行了一番疯狂的查（谷）询（歌），终于找到了将elastic.apm打入链路的方式，server和cilent方向都进行了注入。不得不说elastic的一整套封装还是很香的。不过这个时候gozero又出来捣乱了。apm那边是在新建一个server的时候将已经建立好的server进行一下warp，但是go-zero直接就吧server拿去用了，根本没没有暴露出来。于是血小板又去求教于他的menteor，在一番操作之后，终于在这里找到了warp点。至此，apm就算注入到rpc服务里面去了。（等下，api呢？这个时候的血小板直接忘记了api层的操作，结果一看链路傻了，api层没有追上）

```go
feedbackclient.NewFeedback(zrpc.MustNewClient(
c.Feedback,
zrpc.WithDialOption(grpc.WithUnaryInterceptor(apmgrpc.NewUnaryClientInterceptor())))),
```

api层的注入更是曲折，血小板再次求助mentor，mentor说你去模仿一下apm对gin的操作吧，于是血小板就去看了看gin框架是咋注入的，不看不知道，一看吓一跳，原来就是简单的将上下文进行了交换，把apm的上下文warp进了gin的context内部。于是血小板直接写了一个中间件来处理这个问题。(这个中间件没有处理异常，是不安全的中间件，但是管不了难么多了，能run就行)

```go
func (m *ApmMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		reqName := r.URL.Path
		tx, _, req := apmhttp.StartTransactionWithBody(apm.DefaultTracer, reqName, r)
		defer tx.End()
		r = req
		next(w, r)
	}
}
```

**至此，这个框架算是搭完整了。** 吧~~
