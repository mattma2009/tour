---
layout: post
title: 公有云的选型和性能测试
categories: [work]
tags: [cloud,iaas]
---

由于业务的需要，近期需要展开对阿里云和华为云的测试，主要集中在云主机这块。会陆续把测试的情况记录下来。目前我们关注的指标主要包括：

- 与PAAS的集成能力和兼容性；
- 提供的基础服务；
- 云主机的性能；
- JAVA基准测试；

## 云主机的性能测试 ##

这两天也找了一些关于云主机性能测试的方法，发现基本也是差不多的。蒋清野有一篇[《VM性能的快速测试方法》](http://www.qyjohn.net/?p=2715)，记得两年前参见云计算大会的时候，听过这哥们一个[topic](http://www.qyjohn.net/?p=1948)，关于几个公有云测试的过程和结果。还有一个[《公有云性能测试》](http://blog.csdn.net/shaunfang/article/details/11194289)的情况；总结一下用的方法基本差不多。

工具大概如下，常使用UnixBench来评估虚拟机CPU性能，mbw来评估内存性能，iozone来评估文件IO性能，iperf来评估网络性能，pgbench来评估数据库性能。当然，IO方面也可采用Orion，测定磁盘的IOPS、吞吐量、延迟等。当然在安装UnixBench也可能会遇到一些问题，详细可以参见如下[《UnixBench安装》](http://www.laozuo.org/506.html)。

由于华为云在分配的内网主机是不能访问直接访问互联网的，所以可以在互联网入口机器上做个squid代理。这样就可以愉快的使用了。




 



