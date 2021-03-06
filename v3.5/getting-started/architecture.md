---
title: Rainbond架构
summary: 介绍Rainbond架构及组成部分
toc: false
---

本文我们讲述Rainbond用怎样的架构践行“以应用为中心，软件定义一切”的设计理念。

## 无服务器PaaS

PaaS在云计算典型层级中介于应用和基础设施之间，提供运算平台和解决方案堆栈，像我们经常提到的Google App Engine、Cloud Foundry等平台均属于PaaS。这些早期的PaaS在一定程度上为开发者带来了效率和成本上的提升，但也存在着支持语言有限、支持场景有限、学习成本太高的问题。

不过随着近年来容器和软件定义系列技术的成熟，以上阻碍PaaS发展的问题有机会得到解决，PaaS可以变得更灵活，形成支持各种语言和架构场景且简单易用的平台——无服务器PaaS（Serverless PaaS），可以从应用角度支持各类复杂技术架构和业务交付流程，让用户专注业务开发和管理，而不需要关注底层服务器相关的复杂技术。

## 抽象架构
<img src="https://static.goodrain.com/images/acp/docs/getting-started/PastedGraphic-1.png" width="100%"/>

基础设施是基础的计算资源，PaaS可以对接管理普通硬件服务器，IaaS和虚拟化，实现各类计算资源的统一管理。

无服务器PaaS ，将各类基础设施通过软件定义（SDN/SDS/LB/docker）的方式对接管理，包装成应用，并提供DevOps、自动运维、微服务架构、应用市场等功能，简化用户使用，提高应用交付效率。

云原生SaaS，各类应用通过PaaS平台构建部署，自动变成云原生SaaS服务，用户不需要懂技术，即点即用。包括各类技术服务：开发平台、数据库、技术工具和云框架（大数据/区块链/人工智能等），各类业务应用：企业应用、行业应用、办公应用和互联网应用，以上各类应用通过微服务架构实现业务整合和数据互通。

## 组件架构

Rainbond完整架构由三个部分组成，云帮应用控制台，云帮资源控制台，云帮数据中心。其中云帮资源控制台作为企业版的特性提供，下文我将分别介绍三部分的架构方式。

<img src="https://static.goodrain.com/images/acp/docs/getting-started/rainbond_architecture.png" width="100%"/>

### [云帮数据中心](https://github.com/goodrain/rainbond)
云帮数据中心完成对计算资源的抽象，提供应用运行的抽象环境，提供应用构建，应用全生命周期的管理，对外提供统一的API服务和消息服务。因此，云帮数据中心是云帮的核心部分，其由一系列的分布式的，高可用的组件构成，我将其命名为插件化服务架构。以下我将对各个组件设计和工作模式进行解读：

* API服务   
数据中心API是一个无状态服务，基于`DDD`设计规划应用及其相关模型操作API，并动态服务发现其他组件提高的API并进行反向代理。其基于简单的RBAC权限管理规范提供统一的对外API的授权管理，因此，数据中心可控得接受多个客户端（云帮应用控制台，云帮资源控制台）的同时访问。另外当前服务也提供了Websocket协议的消息服务，此类服务直接面浏览器端。

* MQ消息服务   
Rainbond数据中心对消息系统的要求是简单得，其可以提供全局一致，高可用的消息队列服务即可。因此，我们基于etcd(v3)包装了rbd-mq服务，提供gRPC接口处理各类异步处理的消息。其本身可以被认为是无状态的，数据无缓存存储于etcd中。MQ服务串接起API与几个异步任务处理模块。

* 应用构建与存储模块   
应用构建器是一个接受并处理应用构建任务的框架，插件式得扩展各种构建方式。目前我们支持[基于源码构建]()，基于Docker镜像或者Dockerfile构建，基于云市应用构建等方式。其中源码构建本身也是插件式得支持各类语言。应用构建器从消息系统接受构建任务，分析构建方式并调用相关插件进行构建。最终输出云帮本地应用。本地应用介质存储分为镜像存储和源码环境包存储两个方面，应用构建器是无状态的，可以部署多点，提高可用性得同时也可以提高应用构建的效率。

* 应用管理器   
应用管理器是一个接受并处理应用生命管理任务的服务，整合应用，应用扩展，存储等资源，智能生成和操作Kubernetes资源，构建应用的运行态。应用管理器提供应用的启停，各类型升级，回滚等操作，并维护应用的运行状态。目前应用管理器是一个部分有状态的服务，我们的目标是将其完全无状态化。

* 负载均衡管理器   
负载均衡管理器是一个框架，完成应用外部访问的路由信息的自动发现，将其动态配置于不同的负载均衡插件。云帮负载均衡管理器类似于Kubernetes的Ingress概念，我们更早出现，应用可以定义自己的负载均衡选项，例如4层协议的服务和7层协议的服务可以使用不同的负载均衡入口。目前我们支持`OpenResty`,`商业宙斯负载均衡`等插件，后续计划增加`envoy`等支持。

* 应用运行时   
应用运行时包含应用网络适配器，微服务运行环境支持，应用业务级性能分析等功能。我们以标准的CNI协议接入应用互联网络，因此目前社区已有的容器网络解决方案Calico,Flannel云帮无缝支持，云帮提供了基于Midonet的多租户应用网络方案。应用运行时同时提供了租户内应用，应用间网络路由等自定义资源的自动发现机制。应用扩展的功能输入输出通过应用运行时与用户互动。应用运行时插件化得支持各类应用协议的性能分析。

* 分布式消息与日志服务   
云帮对每一个异步操作进行分布式跟踪，通过本服务进行日志聚集。形成一个异步操作的最终状态决定机制，同时实时通过Websocket给予操作者反馈。同时本服务作为应用日志的收集处理中心，记录应用标准输出的所有日志并实时传输控制台日志终端。本服务是有状态服务，日志处理有分区机制，其分区完全动态完成，因此其可多点部署。

* 节点管理器   
云帮节点管理器完成对集群，节点的抽象。云帮将节点分为管理，存储，计算，负载均衡等多种属性。对节点进行自动化安装，配置和监控。云帮节点管理器组成云帮集群，内部实现分布式任务机制。可在集群任意节点执行定义的任何任务。我们将Kubernetes的节点抽象作为云帮计算节点抽象的一部分，存储模块节点抽象作为云帮存储节点的一部分，以此类推。

* 性能分析与资源监控服务   
本模块统一收集，存储，处理应用业务性能分析数据，节点监控数据，容器监控数据，应用资源申请数据等，提供查询API。可对接Grafana或自定义的Dashboard进行数据展示。对于监控数据提供报警服务，并逐步抽象出应用的运行状况，实现应用的按需实时伸缩，故障报警等。


### [云帮应用控制台](https://github.com/goodrain/rainbond-ui)

云帮应用控制台是一个Python实现的Web控制台，其抽象站在数据中心更高的维度。一个控制台可以管理多个数据中心，可以在多个数据中心创建管理应用。并能够支持部署跨数据中心的应用。一个数据中心解决了应用与基础设施的解耦，应用与应用的互联。控制台将解决基础设置与基础设置的互联互通的逻辑抽象，以及应用与应用跨数据中心的互联抽象。

### 云帮资源控制台

云帮资源控制台是图形化管理和监控云帮数据中心物理资源的控制台，同时也提供了租户级资源监控与管理功能，做到对资源的高效管理。目前云帮资源控制台未开源，提供企业版使用。



