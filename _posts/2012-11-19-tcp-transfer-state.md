---
layout: post
title: TCP的连接过程以及状态图
categories: [work]
tags: [tcp]
---

今天在分析一个连接问题的时候，有查了一下状态图，发现脑子真不好使。上次在某行就找了半天，遂放上来以备不时之需：

## TCP状态： ##
> 
> LISTEN：侦听来自远方的TCP端口的连接请求
> 
> SYN-SENT：再发送连接请求后等待匹配的连接请求
> 
> SYN-RECEIVED：再收到和发送一个连接请求后等待对方对连接请求的确认
> 
> ESTABLISHED：代表一个打开的连接
> 
> FIN-WAIT-1：等待远程TCP连接中断请求，或先前的连接中断请求的确认
> 
> FIN-WAIT-2：从远程TCP等待连接中断请求
> 
> CLOSE-WAIT：等待从本地用户发来的连接中断请求
> 
> CLOSING：等待远程TCP对连接中断的确认
> 
> LAST-ACK：等待原来的发向远程TCP的连接中断请求的确认
> 
> TIME-WAIT：等待足够的时间以确保远程TCP接收到连接中断请求的确认
> 
> CLOSED：没有任何连接状态

![](http://mattma2009.qiniudn.com/20140628onedrive/2810527642457071166.png)

![](http://mattma2009.qiniudn.com/20140628onedrive/641199996948314474.jpg)

![](http://mattma2009.qiniudn.com/20140628onedrive/641199996948314489.jpg)

![](http://mattma2009.qiniudn.com/20140628onedrive/641199996948314512.jpg)

![](http://mattma2009.qiniudn.com/20140628onedrive/641199996948314522.jpg)

TCP连接的建立可以简单的称为三次握手，而连接的中止则可以叫做四次握手。

## 1.连接的建立 ##
在建立连接的时候，客户端首先向服务器申请打开某一个端口(用SYN段等于1的TCP报文)，然后服务器端发回一个ACK报文通知客户端请求报文收到，客户端收到确认报文以后再次发出确认报文确认刚才服务器端发出的确认报文（绕口么），至此，连接的建立完成。这就叫做三次握手。如果打算让双方都做好准备的话，一定要发送三次报文，而且只需要三次报文就可以了。

可以想见，如果再加上TCP的超时重传机制，那么TCP就完全可以保证一个数据包被送到目的地。

## 2.结束连接 ##
TCP有一个特别的概念叫做half-close，这个概念是说，TCP的连接是全双工（可以同时发送和接收）连接，我们可以将其看成一对单工链接，每个连接被单独释放，相互独立，为释放连接，任何一方可以发送FIN,表示其已经没有数据可以发送，当FIN被确认，这个方向就停止传数据。因此在关闭连接的时候，必须关闭传和送两个方向上的连接。客户机给服务器一个FIN为1的TCP报文，然后服务器返回给客户端一个确认ACK报文，并且发送一个FIN报文，当客户机回复ACK报文后（四次握手），连接就结束了。

## 3.最大报文长度 ##
在建立连接的时候，通信的双方要互相确认对方的最大报文长度(MSS)，以便通信。一般这个SYN长度是MTU减去固定IP首部和TCP首部长度。对于一个以太网，一般可以达到1460字节。当然如果对于非本地的IP，这个MSS可能就只有536字节，而且，如果中间的传输网络的MSS更佳的小的话，这个值还会变得更小。

## 4.TCP的状态迁移图 ##
书P182页给出了TCP的状态图，这是一个看起来比较复杂的状态迁移图，因为它包含了两个部分---服务器的状态迁移和客户端的状态迁移，如果从某一个角度出发来看这个图，就会清晰许多，这里面的服务器和客户端都不是绝对的，发送数据的就是客户端，接受数据的就是服务器。

**4.1.客户端应用程序的状态迁移图**

客户端的状态可以用如下的流程来表示：

CLOSED->SYN_SENT->ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED

以上流程是在程序正常的情况下应该有的流程，从书中的图中可以看到，在建立连接时，当客户端收到SYN报文的ACK以后，客户端就打开了数据交互地连接。而结束连接则通常是客户端主动结束的，客户端结束应用程序以后，需要经历FIN_WAIT_1，FIN_WAIT_2等状态，这些状态的迁移就是前面提到的结束连接的四次握手。

**4.2.服务器的状态迁移图**

服务器的状态可以用如下的流程来表示：

CLOSED->LISTEN->SYN收到->ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED

在建立连接的时候，服务器端是在第三次握手之后才进入数据交互状态，而关闭连接则是在关闭连接的第二次握手以后（注意不是第四次）。而关闭以后还要等待客户端给出最后的ACK包才能进入初始的状态。

**4.3.其他状态迁移**

书中的图还有一些其他的状态迁移，这些状态迁移针对服务器和客户端两方面的总结如下

LISTEN->SYN_SENT，对于这个解释就很简单了，服务器有时候也要打开连接的嘛。
SYN_SENT->SYN收到，服务器和客户端在SYN_SENT状态下如果收到SYN数据报，则都需要发送SYN的ACK数据报并把自己的状态调整到SYN收到状态，准备进入ESTABLISHED
SYN_SENT->CLOSED，在发送超时的情况下，会返回到CLOSED状态。
SYN_收到->LISTEN，如果受到RST包，会返回到LISTEN状态。
SYN_收到->FIN_WAIT_1，这个迁移是说，可以不用到ESTABLISHED状态，而可以直接跳转到FIN_WAIT_1状态并等待关闭。

**4.4. 2MSL等待状态**

书中给的图里面，有一个TIME_WAIT等待状态，这个状态又叫做2MSL状态，说的是在TIME_WAIT2发送了最后一个ACK数据报以后，要进入TIME_WAIT状态，这个状态是防止最后一次握手的数据报没有传送到对方那里而准备的（注意这不是四次握手，这是第四次握手的保险状态）。这个状态在很大程度上保证了双方都可以正常结束，但是，问题也来了。

由于插口的2MSL状态（插口是IP和端口对的意思，socket），使得应用程序在2MSL时间内是无法再次使用同一个插口的，对于客户程序还好一些，但是对于服务程序，例如httpd，它总是要使用同一个端口来进行服务，而在2MSL时间内，启动httpd就会出现错误（插口被使用）。为了避免这个错误，服务器给出了一个平静时间的概念，这是说在2MSL时间内，虽然可以重新启动服务器，但是这个服务器还是要平静的等待2MSL时间的过去才能进行下一次连接。

**4.5.FIN_WAIT_2状态**

这就是著名的半关闭的状态了，这是在关闭连接时，客户端和服务器两次握手之后的状态。在这个状态下，应用程序还有接受数据的能力，但是已经无法发送数据，但是也有一种可能是，客户端一直处于FIN_WAIT_2状态，而服务器则一直处于WAIT_CLOSE状态，而直到应用层来决定关闭这个状态。

## 5.RST，同时打开和同时关闭 ##
RST是另一种关闭连接的方式，应用程序应该可以判断RST包的真实性，即是否为异常中止。而同时打开和同时关闭则是两种特殊的TCP状态，发生的概率很小。








