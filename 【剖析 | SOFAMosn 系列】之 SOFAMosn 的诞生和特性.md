原文地址:https://mp.weixin.qq.com/s/j7o6Ex4gZpuOHJNgWmjOtA

本文为《剖析 | SOFAMosn》第一篇。
《剖析 | SOFAMosn》系列由 SOFA 团队和源码爱好者们出品，
项目代号：<SOFA:MosnLab/>，今天开启共建招募，加入方式见底部。


  SOFAMosn 的诞生
云原生时代，Service Mesh 作为一个专用的基础设施层，用于提供安全、快速、可靠、智能的服务间通讯，可为微服务的连接、管理和监控带来巨大的便利，从而加速微服务的落地。



作为国内领先的金融服务提供商，蚂蚁金服对系统架构的性能、稳定性、安全性要求极高，且面对的运维架构复杂。为了达到高可用和快速迭代的目的，蚂蚁金服正全面拥抱微服务，云原生， 故 Service Mesh 成为助力蚂蚁 SOFA5，以及兼容 K8S 的容器平台 Sigma等微服务化关键组件落地的重要推手。



在 Service Mesh 落地的方案挑选中， Istio 作为 Service Mesh 的集大成者，无论在功能实现，稳定性，扩展性，以及社区关注度等方面都是不二选择，其数据平面 Envoy 更是具有优秀的设计，可扩展的 XDS API，以及较高的性能等特点，蚂蚁一开始便将 Istio 作为重点的关注对象。



然而，由于 Envoy 使用 C++ 语言开发，不符合蚂蚁技术栈的发展方向且无法兼容现在的运维体系，以及蚂蚁内部有许多业务定制化的诉求，导致我们无法直接使用 Istio。经过调研发现，作为云计算时代主流语言的 Golang 同样具有较高的转发性能，这促使我们考虑开发 Golang 版本高性能的 sidecar 来替换 Envoy 与 Istio 做集成。



今天，我们就来介绍它：“SOFAMosn” 。

  初识 SOFAMosn


简单来说，SOFAMosn 是一款采用 Golang 开发的 Service Mesh 数据平面代理，由蚂蚁金服系统部网络团队、蚂蚁金服中间件团队、UC 大文娱团队共同开发，功能和定位类似 Envoy，旨在提供分布式，模块化，可观察，智能化的代理能力。它通过模块化，分层解耦的设计，提供了可编程，事件机制，扩展性，高吞吐量的能力。  



当前， SOFAMosn 已支持 Envoy 和 Istio 的 API，实现并验证了 Envoy 的常用功能(全量功能在开发中)，通过 XDS API 与 Pilot 对接，SOFAMosn 可获取控制面推送的配置信息，来完成代理的功能。在实践中，你可以使用 SOFAMosn 替代 Envoy 作为转发平面与 Istio 集成来实现 Service Mesh 组件，也可以单独使用 SOFAMosn 作为业务网关，通过使用 SOFAMosn 你将在如下几个方面获得收益：

SOFAMosn 使用 Golang 作为开发语言，开发效率高，在云原生时代可与 k8s 等技术无缝对接，有利于加速微服务的落地；

SOFAMosn 可代理 Java，C++，Golang，PHP，Python 等异构语言之间组件的互相调用，避免多语言版本组件的重复开发，可提高业务开发效率，目前 SOFAMosn 已经在蚂蚁金服中作为跨语言 RPC 调用的桥梁被使用；

SOFAMosn 可提供灵活的流量调度能力，有助于运维体系的支撑，包括：蓝绿升级、容灾切换等；

SOFAMosn 提供TLS、服务鉴权等能力，可满足服务加密与安全的诉求；



当前 SOFAMosn 已经在 Github 上开源，我们欢迎所有感兴趣的同学参与进来，与我们一起共建一个精品的 Golang Sidecar，项目地址为：https://github.com/alipay/sofa-mosn



为了帮助大家更好的理解 SOFAMosn，本文作为开篇文章，会整体性的介绍 SOFAMosn 的特性以期给大家一个完整的印象，具体的细节这里不做展开，如果您对细节感兴趣，欢迎关注后续文章。



本文介绍的内容将包括 :

SOFAMosn 是如何工作的

SOFAMosn 内部是如何完成代理功能的

SOFAMosn 如何提高Golang的转发性能

SOFAMosn 做了哪些内存优化

SOFAMosn 如何做到系统的高可用

SOFAMosn 如何支持扩展

SOFAMosn 如何做到安全

  SOFAMosn 是如何工作的
SOFAMosn 本质是一个 4-7 层代理，所以它可以以独立进程的形式作为 sidecar 与用户程序部署在相同的物理机或者VM中，当然也可以以独立网关的形式单独运行在一台主机或者虚拟机中。



以下图为例，MOSN （注: SOFAMosn 有时也简称为 MOSN） 与 Service 部署在同一个 Pod 上，MOSN 监听在固定的端口，一个正向的请求链路包括如下步骤：

ServiceA 作为客户端可使用任意语言实现，可使用目前支持的任意的协议类型，比如HTTP1.x，HTTP2.0，SOFARPC 等，将 sub/pub、request 信息等发送给MOSN

MOSN 可代理 ServiceA 的服务发现，路由，负载均衡等能力，通过协议转换，转发 ServiceA 的请求信息到上游的 MOSN

上游 MOSN 将接收到的请求通过协议转换，发送到代理的 ServiceB 上

反向链路类似，通过上述的代理流程，MOSN 代理了 Service A 与 Service B 之间的请求。



这里有一些需要注意的是：

你可以使用 MOSN 只代理 Client 的请求，MOSN 可以直接访问 Server，链路：Client -> MOSN -> Server，反之亦然

MOSN 上下游协议可配置为当前支持的协议中的任意一种

  SOFAMosn 内部是如何完成代理功能的
了解 SOFAMosn 的代理能力，我们需要窥探它的实现框架以及数据在其内部的流转。这里我们先介绍组成 SOFAMosn 的模块，再介绍 SOFAMosn 的分层设计

SOFAMosn 的组成模块


在上图中，蓝色框中的模块为当前已经支持的模块，红色虚线模块为开发中模块，其中：

Starter 用于启动 MOSN，包括从配置文件或者以 XDS 模式启动，其中Config 用于配置文件的解析等，XDS 用于和 Istio 交互，获取 Pilot 推送的配置等

MOSN 解析配置后，会生成 Server以及Listener ，在 Listener 中有监听端口、 ProxyFilter 、Log 等信息；Server 包含 Listener ，为 MOSN 运行时的抽象，Server 运行后，会开启 Listener 监听，接受连接等

MOSN 运行起来后，还会生成 Upstream相关信息，用于维护后端的  Cluster和  Host信息

MOSN 在转发请求时，会在 Upstream 的 Cluster 中通过 Router 以及 LoadBalancer 挑选 Host

Router 为 MOSN 的路由模块，当前支持根据 label 做路由等

LoadBalance 为 MOSN 的负载均衡模块，支持 WRR，Subset LB

Metrics 模块用于对协议层的数据做记录和追踪

Hardware 为 MOSN 后期规划的包括使用加速卡来做 TLS 加速以及 DPDK 来做协议栈加速的一些硬件技术手段

Mixer 用于对请求做服务鉴权等，为开发中模块

FlowControl 用来对后端做流控，为开发中模块

Lab 和 Admin 模块为实验性待开发模块

SOFAMosn 的分层设计
为了转发数据，实现一个4-7层的 proxy，在分层上，SOFAMosn 将整体功能分为 "网络 IO 层"，"二进制协议处理层"，"协议流程处理层"以及"转发路由处理层" 等四层进行设计，每一层实现的功能高度内聚可用于完成独立的功能，且层与层之间可相互配合实现完整的 proxy 转发。

如下图所示：SOFAMosn 对请求做代理的时候，在入口方向，会依次经过网络 IO 层(NET/IO)，二进制协议处理层(Protocol)，协议流程处理层(Streaming)，转发路由处理层(Proxy)；出向与入向过程基本相反





下面我们简单介绍每一层的作用，关于每一层的特性，请参考：

https://github.com/alipay/sofa-mosn/blob/master/docs/design/MOSNLayerFeature.md

NET/IO 层提供了 IO 读写的封装以及可扩展的 IO 事件订阅机制；

Protocol 层提供了根据不同协议对数据进行序列化/反序列化的处理能力；

Streaming 层提供向上的协议一致性，负责 stream 的生命周期，管理 Client / Server 模式的请求流行为，对 Client 端stream 提供池化机制等；

Proxy 层提供路由选择，负载均衡等的能力，做数据流之间的转发；



下面是将此图打开后的示意图

MOSN 在 IO 层读取数据，通过 read filter 将数据发送到 Protocol 层进行 Decode

Decode 出来的数据，根据不同的协议，回调到 stream 层，进行 stream 的创建和封装

stream 创建完毕后，会回调到 Proxy 层做路由和转发，Proxy 层会关联上下游间的转发关系

Proxy 挑选到后端后，会根据后端使用的协议，将数据发送到对应协议的 Protocol 层，对数据重新做 Encode

Encode 后的数据会发经过 write filter 并最终使用 IO 的 write 发送出去







  SOFAMosn 如何提高 Golang 的转发性能
Golang 的转发性能比起 C++ 是稍有逊色的，为了尽可能的提高 MOSN 的转发性能，我们在线程模型上进行优化，当前 MOSN 支持两种线程模型，用户可根据场景选择开启适用的模型。

模型一
如下图所示，模型一使用 Golang 默认的 epoll 机制，对每个连接分配独立的读写协程进行阻塞读写操作， proxy 层做转发时，使用常驻 worker 协程池负责处理 Stream Event







此模型在 IO上使用 Golang 的调度机制，适用于连接数较少的场景，例如：SOFAMosn 作为 sidecar、与 client 同机部署的场景

模型二
如下图所示，模型二基于 NetPoll 重写 epoll 机制，将 IO 和 PROXY 均进行池化，downstream connection将自身的读写事件注册到netpoll的epoll/kqueue wait 协程，epoll/kqueue wait 协程接受可读事件时，触发回调，从协程池中挑选一个执行读操作







使用自定义 Netpoll IO 池化操作带来的好处是：

当可读事件触发时，从协程池中获取一个 goroutine 来执行读处理，而不是新分配一个 goroutine，以此来控制高并发下的协程数量

当收到链接可读事件时，才真正为其分配 read buffer 以及相应的执行协程。这样可以优化大量空闲链接场景导致的额外协程和 read buffer 开销

此模型适用于连接数较多，可读的连接数有限，例如：SOFAMosn 作为 api Gateway 的场景

  SOFAMosn 做了哪些内存优化
Golang 相比于 C++，在内存使用效率上依赖于 GC，为了提高 Golang 的内存使用率，MOSN 做了如下的尝试来减少内存的使用，优化 GC 的效率：

通过自定义的内存复用接口实现了通用的内存复用框架，可实现自定义内存的复用

通过优化 []byte 的获取和回收，进一步优化全局内存的使用；

通过优化 socket 的读写循环以及事件触发机制，减小空闲连接对内存分配的使用，进一步减少内存使用；

使用 writev 替代 write, 减少内存分配和拷贝，减少锁力度；

  SOFAMosn 如何做到系统的高可用
MOSN 在运行时，会开启 crontab 进行监控，在程序挂掉时，会及时拉起；

同时，MOSN 在 进行升级或者 reload 等场景下做连接迁移时， 除了经典的传递 listener fd 加协议层等待方式以外，还支持对存量链接进行协议无关的迁移来实现平滑升级，平滑 reload 等功能；

在对存量连接进行迁移时，mosn 通过 forkexec 生成New mosn，之后依次对存量的请求数据做迁移，对残留响应做迁移来完成；

  SOFAMosn 如何支持扩展
MOSN 当前支持 “协议扩展” 来做到对多协议的支持，支持 “NetworkFilter 扩展” 来实现自定义 proxy 的功能，支持 “StreamFilter 扩展”  来对数据做过滤：

1. 协议扩展
MOSN 通过使用同一的编解码引擎以及编/解码器核心接口，提供协议的 plugin 机制，包括支持

SOFARPC

HTTP1.x, HTTP2.0

Dubbo

等协议，后面还会支持更多的协议

2. NetworkFilter 扩展
MOSN 通过提供 Network Filter 注册机制以及统一的 packet read/write filter 接口，实现了Network Filter 扩展机制，当前支持：

TCP Proxy

Layer-7 Proxy

Fault Injection

3. StreamFilter 扩展
MOSN 通过提供 Stream Filter 注册机制以及统一的 stream send/receive filter 接口，实现了 Stream Filter 扩展机制，包括支持：

支持配置健康检查等

支持故障注入功能

  SOFAMosn 如何做到安全
SOFAMosn 中，通过使用 TLS 加密传输和服务鉴权来保证消息的安全可靠，在未来还会通过使用 keyless 等方案来提高加解密的性能，下面我们介绍SOFAMosn 在 TLS 上的一些实践

1. TLS 选型
在 SOFAMosn 中使用 TLS 有两种选择，1) 使用 Golang 原生的 TLS ， 2) 使用 cgo 调用 Boring SSL

我们通过压测发现，在 ECDHE-ECDSA-AES256-GCM-SHA384 这种常用的加密套件下，Go 自身的 TLS 在性能上优于 Boring SSL，与 Openssl 相差不多

经过调研发现，Go 对 p256，AES-GCM 对称加密，SHA，MD5 等算法上均有汇编优化，因而我们选择使用 Golang 自带的 TLS 来做 SOFAMosn 的 TLS 通信

2. TLS 方案
SOFAMosn 间使用 Golang 原生的 TLS 加密，支持  listener 级别的 TLS 配置，配置包括证书链与证书列表等，用来做监听时使用；支持 cluster 级别的 TLS 配置，cluster 配置对所有的 host 生效，用来向后端发起连接时使用；host 中有一个配置用来标明自己是否支持 TLS

SOFAMosn server 通过使用 Listener 的 Inspector 功能，可同时处理客户端的 TLS 和 非 TLS 请求

SOFAMosn client 在发起连接的时候，根据挑选的 host 是否支持 TLS 来决定是否使用 TLS 发起连接

  欢迎加入 <SOFA:MosnLab/>，参与 SOFAMosn 源码解析
<SOFA:Lab/> 启动一个多月，分别推出 <SOFA:RPCLab/> 和  <SOFA:BoltLab/> 源码分析共建小组，向大家汇报一下进度：

【剖析 | SOFARPC 框架】系列已经完成领取，SOFA 团队正在与爱好者们打磨内容；
【剖析 | SOFABolt】已被认领两篇，持续开放认领中；
