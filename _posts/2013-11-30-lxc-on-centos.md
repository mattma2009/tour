---
layout: post
title: 折腾lxc-centos模板
categories: [work]
tags: [lxc，centos]
---

最近在折腾linuxContainer，ubuntu的版本比较容易弄，已经集成到版本中；而centos的比较难折腾，centos官方是用[libvirt](http://wiki.centos.org/HowTos/LXC-on-CentOS6)管理的，跟sourceforge的[lxc](http://sourceforge.net/projects/lxc/)完全是两套东西。折腾了两天，在github的lxc中找到了一个[centos模板的脚本](https://github.com/lxc/lxc)，然后各种改，终于搞定。

