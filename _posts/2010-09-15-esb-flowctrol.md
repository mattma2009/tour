---
layout: post
title: 总线之流量控制
categories: [work]
tags: [soa eab]
---

流量控制是总线系统非常核心的模块，它能有效控制在高并发交易场景的流量情况，达到在业务突发状况下的业务系统以及平台的稳定和可靠性。

典型的流量控制算法采用令牌桶的算法控制网络流量，百度百科令牌桶算法如下：
> 令牌桶算法是网络流量整形（Traffic Shaping）和速率限制（Rate Limiting）中最常使用的一种算法。典型情况下，令牌桶算法用来控制发送到网络上的数据的数目，并允许突发数据的发送。
>
令牌桶这种控制机制基于令牌桶中是否存在令牌来指示什么时候可以发送流量。令牌桶中的每一个令牌都代表一个字节。如果令牌桶中存在令牌，则允许发送流量；而如果令牌桶中不存在令牌，则不允许发送流量。因此，如果突发门限被合理地配置并且令牌桶中有足够的令牌，那么流量就可以以峰值速率发送。
>
令牌桶算法的基本过程如下：
>
1、假如用户配置的平均发送速率为r，则每隔1/r秒一个令牌被加入到桶中；
>
2、假设桶最多可以存发b个令牌。如果令牌到达时令牌桶已经满了，那么这个令牌会被丢弃；
>
3、当一个n个字节的数据包到达时，就从令牌桶中删除n个令牌，并且数据包被发送到网络；
如果令牌桶中少于n个令牌，那么不会删除令牌，并且认为这个数据包在流量限制之外；
>
4、算法允许最长b个字节的突发，但从长期运行结果看，数据包的速率被限制成常量r。对于在流量限制外的数据包可以以不同的方式处理：它们可以被丢弃；它们可以排放在队列中以便当令牌桶中累积了足够多的令牌时再传输；它们可以继续发送，但需要做特殊标记，网络过载的时候将这些特殊标记的包丢弃。

总线的流量控制借鉴了令牌桶算法的部分思路，并进行了扩展。实现了主备流控、分时令牌桶、以及本地流控相结合的方式。 

> 首先，总线是面向服务的框架，所以总线的流量控制是基于服务、渠道、服务系统等对象、并可可以基于时间窗口进行流量控制的；对于一些关键服务的并发量需要有一个指标，同样的原理，这一时刻，调用这个服务的请求即申请一个令牌，当并发达到设定的上限，即没有令牌可以申请，请求被拒绝。

> 其次，流量控制是一个关键应用，所有请求都会先到流量控制申请令牌。所以，需要对流量控制进行主备部署，主备流控服务器之间有心跳。当请求达到总线，先到主流控服务器申请令牌，总线运行时和流控服务之间采用长连接实现。当主流控服务器发生故障时，备流控器会自动切换为主，总线运行时会重连到当前的主流控服务器。当主备流控服务器都全部down掉之后，为保证交易的运行，会有一个本地流控服务还接续，但是本地流控是不精确流控。

> 最后，这里面需要考虑故障重连的机制；还有当后端服务调用发生异常，不能正常归还令牌，需要有超时令牌自动回收的机制。

另外，当总线进行总分部署的情况下，需要考虑流控的分级部署。令牌分配的策略需要进行调整，分流控向主流控每次都进行通信，肯定是开销大，相当于集中流控，但是这种方式最精确。分流控每次申请一批的话，如何划分批的策略，需要特定情况下进行分析。还有多级流控，谁是主流控服务需要进行投票的方式进行选举，也是需要斟酌的点。  