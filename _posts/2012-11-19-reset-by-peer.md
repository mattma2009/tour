---
layout: post
title: reset by peer
categories: [work]
tags: [tcp]
---

reset by peer这个错误信息一般而言对应的是复位信息，就是收到了RST。查了一下，这个信号一般只有两种情况会收到，一是向不存在的端口发起了一个连接请求；再有就是连接异常终止，这里跟SO_LINGER有关.
SO_LINGER选项用来改变此缺省设置。使用如下结构：

`struct linger {

     int l_onoff; /* 0 = off, nozero = on */
     int l_linger; /* linger time */
};`

有下列三种情况：

l_onoff为0，则该选项关闭，l_linger的值被忽略，等于缺省情况，close立即返回；

l_onoff为非0，l_linger为0，则套接口关闭时TCP夭折连接，TCP将丢弃保留在套接口发送缓冲区中的任何数据并发送一个RST给对方，而不是通常的四分组终止序列，这避免了TIME_WAIT状态；

l_onoff 为非0，l_linger为非0，当套接口关闭时内核将拖延一段时间（由l_linger决定）。如果套接口缓冲区中仍残留数据，进程将处于睡眠状态，直 到（a）所有数据发送完且被对方确认，之后进行正常的终止序列（描述字访问计数为0）或（b）延迟时间到。此种情况下，应用程序检查close的返回值是非常重要的，如果在数据发送完并被确认前时间到，close将返回EWOULDBLOCK错误且套接口发送缓冲区中的任何数据都丢失。close的成功返回仅告诉我们发送的数据（和FIN）已由对方TCP确认，它并不能告诉我们对方应用进程是否已读了数据。如果套接口设为非阻塞的，它将不等待close完成。

当客户端关闭，并且设置了solinger为0，这是服务端执行socket.read产生reset。之前都是先关了server端socket。而这种场景下要求客户端决定是否close这个连接，而服务端监听，然后关闭。

没有POSP直接拿POS连还是头一回见。





