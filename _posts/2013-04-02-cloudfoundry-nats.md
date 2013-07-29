---
layout: post
title: 以NATS为主线的Cloud Foundry原理
categories: [work]
tags: [cloud]
---

>转一篇关于CloudFoundry的技术文章。

[译注]本文作者张磊，浙江大学VLIS 实验室的硕士研究生，他和他的同学们是国内较早一批针对Cloud Foundry 做系统性研究和改进的学生力量。目前正在Cloud Foundry SE 团队实习，旨在为国内的合作伙伴提供更完善的Cloud Foundry 解决方案。

本文将以Cloud Foundry中的消息组件NATS为主要线索、以在Cloud Foundry中广泛使用的EventMachine为侧重，来串联整个Cloud Foundry主线功能的工作原理，力求能用简单直接的方式描述出更多的系统细节和架构思路。

NATS是Cloud Foundry内部的通信系统，是一款基于EventMachine、使用“发布-订阅”机制的轻量级消息中间件。NATS由负责发出操作指令的客户端和负责处理操作指令的服务器端组成，所有NATS客户端不需要持有其他客户端的信息就可以根据主题匹配机制来完成消息订阅和发布过程。这一特性使得我们可以将单个Cloud Foundry虚拟机使用NATS连接起来从而快速得到Cloud Foundry集群——这正是使用BOSH部署Cloud Foundry的技术基础。

NATS主要依赖EventMachine（EM）进行网络通信。作为Ruby界较为知名的并发和网络编程框架，EM通过实现Reactor设计模式解决了Ruby语言与生俱来的并发能力不足的问题（主要是由Ruby解释器全局锁导致的），也为NATS带来了良好的并发请求处理能力。NATS不对消息做持久化，所以对消息的匹配和订阅过程相对高效。

NATS的通信机制基于EM所提供的TCP连接功能。每次会话起始于NATS客户端发起请求与服务器端建立连接，然后NATS服务器端回复一条INFO信息作为响应，之后NATS便可以开始工作。

 ![](http://cnblog.cloudfoundry.com/wp-content/uploads/2013/03/Picture1.jpg)

图1 一次订阅和发布交互中TCP数据流的顺序图

图1是订阅和发布交互中TCP数据流的顺序图。可以看到这次订阅和发布的交互过程如下。

- 双方的连接成功建立之后（CONNECT操作成功得到响应之后），客户端首先订阅了主题为foo的消息，SID为1。
- 服务器端会记录下这一主题和SID并响应+OK。
- 客户端发布了一条主题为foo的消息，长度为12，然后紧接着发来了消息数据“Hello World!”。
- 服务器端通过主题匹配找到该主题订阅者的SID是1，然后将这个消息的主题foo、SID值1及消息携带的数据“Hello World!”一起返回给客户端。
- 客户端根据SID=1从自己维护的订阅者列表里找到对应的订阅者，将服务器端返回来的数据交给订阅者使用。

NATS服务器端负责进行主题匹配的数据结构被称作Sublist，它是一个由Node和Level两个结构体来描述的树形结构。Sublist使用pwc和fwc来标记消息主题中通配符对应的节点。当NATS收到publish指令之后，它会把诸如“A.B.C”这样的主题拆分成三个Token，然后将每个Token中的内容从Sublist的ROOT层开始按照Level分别进行匹配。如果每个Token都能匹配到节点，那么最后一个Token对应的Node就是这个主题的订阅者了。

 ![](http://cnblog.cloudfoundry.com/wp-content/uploads/2013/03/Picture2.jpg)

图2 Sublist结构

图2形象地描述了整个Sublist结构。

在Cloud Foundry的使用场景中，每个系统组件的启动大多以启动EM和NATS开始，然后在NATS启动过程中设置计时器、注册组件、订阅本组件关心的消息。当然，在Cloud Foundry中并不是所有消息传递都由NATS来做。NATS主要适用以下场景：Publisher并不知道也没有必要关心Subscriber的存在和数量，同样后者对前者的存在也无需关心。而对于有一些需要事先知晓对方信息后再建立通信的场合，Cloud Foundry会采用其他方式来响应请求，比如用户经由Router访问到应用instance及Service Gateway与Cloud Controller之间的关系等。不过，很快就能会看到NATS和EM为Cloud Foundry提供的并不只是消息的传递控制，它们为Cloud Foundry系统带来了基于消息和事件驱动的工作方式以及松耦合和自治式的组件结构。

## 以NATS为主线的Cloud Foundry原理解读 ##

先回顾一下Router的相关知识。Router作为Cloud Foundry的请求访问分配与转发门户，它的工作原理在《Cloud Foundry技术全貌及核心组件分析》文中已有详细描述，本文中仅强调一下NATS在Router中扮演的角色。Router订阅的消息无外乎两种：register和unregister。某个URL对应的组件或者instance正是使用NATS向Router注册自己的IP和port到这个URL上。这些组件需要发布一个以“router.register”为主题的消息，然后这个主题的订阅者——Router会对组件发来的消息做出响应并在回调方法中完成接下来的注册工作。NATS的订阅者无需等待消息也无需关心消息的到达，它们会继续执行订阅语句后面的任务，并放心地把处理消息的过程交给消息订阅的回调方法完成。这种“消息订阅-回调”的工作模式为Cloud Foundry提供了消息驱动的异步执行框架，是整个平台良好运作的重要基础。

## DEA与NATS ##

作为Cloud Foundry中支撑应用运行的重要组件，DEA对消息系统有很大的使用需求。除了在启动时初始化必要变量外，DEA需要订阅一系列主题的消息并在订阅完成后向其他组件广播自己启动的信息。这里DEA的很多订阅是带有reply的，这意味着这些订阅者在的消息处理完毕后还需要使用NATS.publish(reply,response.to_json)来返回处理结果给发布者。这种基于NATS的“请求-响应-回复”机制是DEA的重要的处理框架。下面不妨先通过一个场景来描述一下DEA工作流程。

- 当启动一个App instance时，DEA节点会从指定位置下载一个Droplet的副本来启动。
- 如果将该App扩展到N个instance，那么这个Droplet就会被复制N份交给DEA来启动。
- Cloud Foundry使用NATS来“发现”DEA，DEA根据自己的“能力”来立即或推迟响应请求，instance会被下载到最先响应的DEA上启动。
- 启动后的instance会被分配PID和响应端口，它会将自己的IP+Port信息注册到Router中对应的URL上。
- DEA负责把应用实例的运行状态定时报告给Health Manager。

好了，有了上面的基础，我们来讨论三个问题。

1) 我们知道开发者上传的应用会保存在Cloud Controller的某个目录下，然后DEA才能够从CC处下载到Droplet并在DEA建立本地执行目录。那么DEA是如何从CC处获取到这部分数据的呢？

DEA首先会判断本地是否已有了应用的可执行目录和所需的文件。如果已存在，直接使用就好。

如果不存在，DEA首先需要判断自己与CC之间是否建立了共享文件系统。如果是的话，直接使用文件操作从CC的/var/vcap/shared/下把用户上传的文件复制过来即可。在非BOSH支持情况下，DEA和CC这部分文件共享是需要手动配置的。简言之，就是建立一个NFS server和共享目录，然后将CC和DEA都mount到这个目录。当然，我们可以使用其他支持FUSE的文件系统来实现HP/HA，毕竟这部分用户应用的存储是十分重要的。

如果共享文件系统也没有建立，那我们还有最后一种方式：直接通过HTTP方式下载Droplet。这时DEA会通过EM向目的URL发起HttpRequest，并以流的方式将文件写入到DEA本地的目录中。

然后DEA还要调用bind_local_runtime(instance_dir,runtime_name)方法来绑定runtime才能运行。该方法将VCAP_LOCAL_RUNTIME环境变量被替换成当前DEA runtime的可执行文件路径，如：../devbox/deploy/rubies/ruby-1.9.2-p180/bin/ruby。

这样，应用在DEA里运行起来其实就与本机运行的环境几乎是一致的，而且这里Cloud Foundry没有做其他特殊的处理。对运行环境轻量级的封装不需要用户调用特定的API或导入外部依赖，是Cloud Foundry一个很大的优点。

2) Cloud Foundry如何从众多DEA中选出一个合适的来运行某个应用的Droplet呢？

DEA的process_dea_discover(message, reply)方法中解释了“发现DEA”的过程：在检查过资源限制和运行环境支持后，DEA会调用calculate_help_taint方法来计算出延迟时间，然后DEA会在这个延迟时间过去之后才做出响应，所以最先响应的DEA也就是延时时间较短的DEA。作为度量DEA“好坏”的标准，主要考虑两个因素的和来作为延时值：

- 该DEA上对应Droplet已启动的instance数量；
- 该DEA上的物理资源的使用量。

综合考虑以上两个因素，就可以确定首先做出响应的DEA的确应该被作为“最佳DEA”获得Droplet的运行权。

3) DEA如何让运行起来的instance能被访问到呢

在process_router_start(message)方法中我们可以找到答案：一旦Router启动并发布了router.start消息，订阅这个消息的各个DEA就会向Router注册自己持有的instance，DEA把instance的信息封装成msg_json然后使用NATS.publish(‘router.register’,msg)发布出去。Router中已订阅了register消息，接下来Router就会解析DEA发来的msg_json，从中读取出这个instance的IP、端口和对应的URL，然后将这些信息在Router中保存起来。

## Health Manager与NATS ##

Health Manager（HM）要做的很多工作都要靠其他组件发来的消息来触发，所以可以想象到它必然是一个消息订阅大户。实际上HM订阅的消息包括了DEA发来的心跳消息、DEA关闭instance后的消息、更新DEA instance的消息、查询应用状态的消息、查询应用健康状况的消息等诸多与应用息息相关的信息。另一方面，HM的一个重要职责就是定时分析应用和instance的状态。如果发现instance的状态为down，而对应Droplet的状态却是started，那么会认为此instance需要重启。这时该instance的ID会被记录下来，然后HM将会试图启动对应App的一个新instance。当然，HM是只读的，所以启动instance的实际工作应该由Cloud Controller来完成，而HM只需要组装好一个JSON格式的消息内容，然后使用NATS来发布这个主题为cloudcontrollers.hm.requests的消息就可以了。Cloud Controller在收到这个消息后就会使用发布者传来的消息内容来启动一个新的instance。可以看到在这里CC和HM完全不必相互保存对方的信息就能默契地协作完成一项看似复杂的工作，这是NATS为Cloud Foundry带来的便利之一。

## Service与NATS ##

除了前面提到的应用层面，NATS在service部分同样大有用处——尽管我们前面提到过在Service Gateway–Cloud Controller-Service Node这条线上NATS并不是信息传递的最主要方式。

Service Gateway由两部分组成，Asynchronous Service Gateway是负责Gateway向上与Cloud Controller进行HTTP交互的部分；Provisioner则是负责Gateway向下与Service Node进行交互的过程。在图3中可以清楚地看到，NATS是Gateway通过Provisioner与Service Node进行通信的媒介。

![](http://cnblog.cloudfoundry.com/wp-content/uploads/2013/03/Picture3.jpg)

图3 NATS是Gateway与Service Node的通信媒介

按照前面的规矩，每个Service Node节点在启动后，都要使用NATS向外publish一遍自己的信息，这个操作叫announcement。以MySQL为例，它需要发布的信息包括节点ID、支持的版本和service plan，还有这个Node的capacity等。当然在Service

Gateway中一定订阅了对应主题的消息来负责处理这些内容。正是通过announcement操作，Service Gateway就可以记录下自己需要的Service Node的信息。

当使用vmc create-service来创建服务时，实际上经过了如下几个流程。

- Cloud Controller向Service Gateway请求Provision操作并通过HTTP方式发送请求的内容。
- Gateway响应并把请求内容转交给Provisione，由Provisioner向Service Node发起provision操作。
- Provisioner要先在前面announcement操作得到的Nodes中选择一个最佳Node。这个Node的选择一般会基于该节点上已经创建的服务实例数目等资源因素来决定。
- 然后，Provisioner将前面Cloud Controller传来的请求内容通过NATS#request发给这个最佳Node，Gateway会在这个请求的回调方法中处理回复内容。
- 剩下的provision操作可以放心地交给这个Service Node来完成，然后把操作的结果返回给Gateway。

那么最佳Service Node究竟做了什么工作来完成provision操作呢？

其实，provision的主要工作内容就是负责在Service Node上生成出credential信息来（包括数据库名、用户名和密码等），然后修改该Node上的capacity等属性，最后把这些信息作为回复内容通过NATS#reply返回给发起这次操作的Provisioner。

上述create-service操作创建了数据库，同时也生成了user和password信息，那么bind-service操作实际上就是在之前创建的数据库中为之前生成的用户分配权限，这样持有这些credential信息的App instance就能像普通应用那样访问这个数据库了。

## 总结 ##

举了这么多例子，无非是想说明这样一个事实：在Cloud Foundry中存在着大量异步的、基于消息和响应的工作流需要处理，所以需要一种像NATS这样轻量、兼具效率和功能的消息中间件来完成解耦和提高任务响应能力。有了NATS的帮助，Cloud Foundry的各个组件只需专注于自己的本职工作，然后通过消息和事件来与其他同伴进行交互和协同。NATS为整个平台带来了强大的可扩展性，新的组件仅需要根据消息的主题与NATS建立联系就可以融入到整个Cloud Foundry环境中。这种灵活自治的工作模式为节点数量众多的分布式云平台带来了高效的组件发现能力和水平扩展能力，也是Cloud Foundry设计上的一大亮点。

另外，Cloud Foundry的社区非常活跃，所以这里提到的很多细节在最新的代码里很可能已经发生了改变。希望本文的作用在于能为读者探索Cloud Foundry提供一些有用的线索，相信读者很快会发现更多有趣的东西。

## 扩展 ##
针对上文描述的原理，我们梳理了一个NAT的消息流，可以参考，详细内容参见[excel](http://sdrv.ms/17OIwt4)。

![](https://nuj3vq.bn1.livefilestore.com/y2pFUcfKempSxIEzapoO5IVbf7XLzrslTdx1M5R3H-SlOfrX9iLvQQbUOp3lgReVFrBbPgdnWDcKPUluaxkrkUqwrpVF8W0qF4z_ydZyVDYfE0/nat.jpg?psid=1)


