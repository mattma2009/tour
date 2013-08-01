---
layout: post
title: tcp_timestamps导致连接失败分析
categories: [work]
tags: [tcp]
---

这两天发现发起方系统很多connect失败的问题，通过抓包工具分析，服务端没有做ACK响应。经过
经过分析试验，最终确认和proc参数tcp_tw_recycle/tcp_timestamps相关；

## 1. 现象 ##

应用实例A(独立部署在一台机器)通过F5访问服务S成功，而应用实例B(独立部署在一台机器)通过F5访问服务S出现connect失败，抓包发现：服务S端已经收到了syn包，但没有回复synack；F5采用了SNAT方式，即两个实例通过F5同一个IP访问服务S。


## 2. 分析 ##

根据现象上述问题明显和tcp timestmap有关；查看linux 2.6.32内核源码，发现tcp_tw_recycle/tcp_timestamps都开启的条件下，60s内同一源ip主机的socket connect请求中的timestamp必须是递增的。

    源码函数：tcp_v4_conn_request(),该函数是tcp层三次握手syn包的处理函数（服务端）；
    源码片段：
       if (tmp_opt.saw_tstamp &&
            tcp_death_row.sysctl_tw_recycle &&
            (dst = inet_csk_route_req(sk, req)) != NULL &&
            (peer = rt_get_peer((struct rtable *)dst)) != NULL &&
            peer->v4daddr == saddr) {
            if (get_seconds() < peer->tcp_ts_stamp + TCP_PAWS_MSL &&
                (s32)(peer->tcp_ts - req->ts_recent) >
                            TCP_PAWS_WINDOW) {
                NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_PAWSPASSIVEREJECTED);
                goto drop_and_release;
            }
        }
        tmp_opt.saw_tstamp：该socket支持tcp_timestamp
        sysctl_tw_recycle：本机系统开启tcp_tw_recycle选项
        TCP_PAWS_MSL：60s，该条件判断表示该源ip的上次tcp通讯发生在60s内
        TCP_PAWS_WINDOW：1，该条件判断表示该源ip的上次tcp通讯的timestamp 大于 本次tcp

   分析：主机应用实例A和应用实例B通过F5（1个ip地址）访问serverN，由于timestamp时间为系统启动到当前的时间，因此，client1和client2的timestamp不相同；根据上述syn包处理源码，在tcp_tw_recycle和tcp_timestamps同时开启的条件下，timestamp大的主机访问serverN成功，而timestmap小的主机访问失败；

    参数：/proc/sys/net/ipv4/tcp_timestamps - 控制timestamp选项开启/关闭
          /proc/sys/net/ipv4/tcp_tw_recycle - 减少timewait socket释放的超时时间

## 3. 解决方法 ##
  echo 0 > /proc/sys/net/ipv4/tcp_tw_recycle;
 
  tcp_tw_recycle默认是关闭的，有不少服务器，为了提高性能，开启了该选项；

  为了解决上述问题，个人建议关闭tcp_tw_recycle选项，而不是timestamp；因为 在tcp timestamp关闭的条件下，开启tcp_tw_recycle是不起作用的；而tcp timestamp可以独立开启并起作用。

    源码函数：  tcp_time_wait()
    源码片段：
        if (tcp_death_row.sysctl_tw_recycle && tp->rx_opt.ts_recent_stamp)
            recycle_ok = icsk->icsk_af_ops->remember_stamp(sk);
        ......
       
        if (timeo < rto)
            timeo = rto;

        if (recycle_ok) {
            tw->tw_timeout = rto;
        } else {
            tw->tw_timeout = TCP_TIMEWAIT_LEN;
            if (state == TCP_TIME_WAIT)
                timeo = TCP_TIMEWAIT_LEN;
        }

        inet_twsk_schedule(tw, &tcp_death_row, timeo,
                   TCP_TIMEWAIT_LEN);


timestamp和tw_recycle同时开启的条件下，timewait状态socket释放的超时时间和rto相关；否则，超时时间为TCP_TIMEWAIT_LEN，即60s；

    内核说明文档 对该参数的介绍如下：
    tcp_tw_recycle - BOOLEAN
    Enable fast recycling TIME-WAIT sockets. Default value is 0.
    It should not be changed without advice/request of technical
    experts.


