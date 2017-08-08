title: motan框架源码解析

date: 2016-05-25

categories: RPC

tags: [motan]

---
>  新浪微博开源了他们内部使用的RPC框架motan，motan相对于阿里巴巴的dubbo框架来说要更轻量级一些，因此选择对motan源码进行学习，但是万变不离其宗，内部机理与dubbo框架一致，motan搞明白了，dubbo也就差不多了，本节主要介绍开源框架motan的基本工程结构和基本模块。

<!--more-->

## motan工程结构

motan整体的工程目录如图所示。主要包含以下几个模块：
![motan整体工程结构图.png](http://upload-images.jianshu.io/upload_images/1938652-0747b6d1fe5b82f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 * motan-core：该工程是motan的核心框架实现，主要包括 Rpc协议实现、网络通信的封装、序列化的实现、集群的管理、配置管理的实现等。
 * motan-registry-zookeeper：zookeeper作为服务注册中心的实现，用于服务的注册、发现等。
 * motan-registry-consul：consul作为服务注册中心的实现，同样用于服务的注册、发现等。
 * motan-transport-netty：该工程实现基于netty的长连接网络通信框架。
 *  motan-springsupport：实现对spring框架的支持。

## motan核心框架 motan-core

因motan-core是整个motan框架的核心，因此在这里简单介绍该工程的各个模块。如图所示，motan的核心实现包括register、transport、serialize、protocol、config等功能模块。

![motan-core模块.png](http://upload-images.jianshu.io/upload_images/1938652-861a9d863b56e45c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

各功能模块功能如下：
 * protocol
最核心层，实现RPC协议框架，包括export暴露服务、refer引用服务等，在该模块中只实现了简单的点对点的RPC调用，对于像服务的发现、集群配置都是在该模块基础之上。
 * registry
用来和注册中心交互，包括服务注册、服务发现、服务订阅、服务变更通知、服务心跳发送等功能；server端会在系统初始化时通过register模块注册服务，client端在系统初始化时会通过register模块订阅到具体提供服务的server列表，当server列表发生变更时也有register模块通知给client。
 * transport
RPC框架实现需要解决的问题之一就是高效的网络通信，该模块实现远程网络通信的接口，默认情况下使用Netty 基于NIO的TCP长连接方式。
 * Serialize
RPC框架实现需要解决的另一个问题是高效的序列化，该模块对现有的序列化框架进行了封装，主要使用的是FastJson和hession序列化框架。
 * Config
提供对协议、注册中心、server、client的配置服务，可直接供外部调用。
 * Cluster
Client端使用的模块，cluster包含若干可以提供RPC服务的Server，实际请求时会根据不同的高可用与负载均衡策略选择一个可用的Server，该模块实现failfast和failover两种集群容错模式，并实现了random、Round-robin、一致性哈希、本地服务优先、低并发优先和权重可配置等六种负载均衡策略。