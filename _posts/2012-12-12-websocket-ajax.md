---
layout: post
title: 此异步非彼异步
categories: [work]
tags: [ajax,websocket]
---

前两天讨论前段ajax异步的时候，包括html5中websocket，由于一直对前段技术不是很了解，今天查了些资料仔细分析一下，特简单记录。

##AJAX和反向Ajax##
和传统的web交互方式相比，用户体验较好，主要体现在：

- 1、不用整个页面都去刷新，可以进行区域的刷新，能够降低带宽。
- 2、传统的web应用到服务器端请求，整个页面是同步的模式。而ajax对于局部的刷新可以支持同步和异步。
- 3、核心是XMLHTTPRquest对象。send可以是异步的。

但是请求还是**单向**的，只能从客户端向服务端发送请求。反向Ajax (Reverse Ajax) 本质上则是这样的一种概念：能够从服务器端向客户端发送数据。在一个标准的 HTTP Ajax 请求中，数据是发送给服务器端的，反向 Ajax 可以某些特定的方式来模拟发出一个 Ajax 请求，这些方式本文都会论及，这样的话，服务器就可以尽可能快地向客户端发送事件（低延迟通信）。

参考资料：

[《反向ajax》](http://www.ibm.com/developerworks/cn/web/wa-reverseajax1/),介绍的比较清楚。

[《XMLHttpRequest Level 2》](http://www.ruanyifeng.com/blog/2012/09/xmlhttprequest_level_2.html)，html5关于xmlhttp的改进。

#websocket#

WebSocket 技术来自 HTML5，是一种最近才出现的技术，许多浏览器已经支持它（Firefox、Google Chrome、Safari 等等）。WebSocket启用双向的、**全双工的通信信道**，其通过某种被称为 WebSocket 握手的HTTP请求来打开连接，并用到了一些特殊的报头。连接保持在活动状态，您可以用 JavaScript 来写和接收数据，就像是正在用一个原始的 TCP 套接口一样。HTML 5 Web Socket并不是普通HTTP通信的增强版，它代表着一个巨大的进步，特别是针对实时的、事件驱动的Web应用程序 。


前面反向ajax里面对Comet方式有详细的描述，但是看下图的对比还是很吃惊。

![](http://images.51cto.com/files/uploadimg/20100317/0907040.jpg)
  

HTML 5 Web Socket定义在HTML 5规范的通信章节，它代表Web通信的下一个演变：通过一个单一的Socket实现一个全双工，双向通信的信道。HTML 5 Web Socket提供了一个真正的标准，你可以使用它构建可扩展的实时Web应用程序。此外，由于它提供了一个浏览器自带的套接字，消除了Comet解决方案 的许多问题，Web Socket显著降低了系统开销和复杂性。

为了建立一个Web Socket连接，客户端和服务器在初始握手期间要从HTTP协议升级到WebSocket协议，
建立好连接后，WebSocket数据帧就可以在客户端和服务器之间以全双工模式传输，在同一时间任何方向，可以全双工发送文本和二进制帧，最小的帧只有2个字节。在文本帧中，每一帧始于0x00直接，止于0xFF字节，数据使用UTF-8编码。

资料参考：

[html5 websocket](http://www.cnblogs.com/liunx/archive/2011/11/24/2261373.html)

[spdy](http://zh.wikipedia.org/wiki/SPDY) 最近HTTP2.0出草案，基于google spdy。