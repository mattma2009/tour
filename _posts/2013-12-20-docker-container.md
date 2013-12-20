---
layout: post
title:  Docker Getting Start
categories: [work]
tags: [paas]
---
转载自：如下地址，原文链接如下[docker介绍](http://tiewei.github.io/cloud/Docker-Getting-Start/)，一篇不错介绍docker的文章；


> Docker is an open-source engine that automates the deployment of any application as a lightweight, portable, self-sufficient container that will run virtually anywhere.

Docker 是 PaaS 提供商 dotCloud 开源的一个基于 LXC 的高级容器引擎， 源代码托管在 Github 上, 基于go语言并遵从Apache2.0协议开源。 Docker近期非常火热，无论是从 github 上的代码活跃度，还是Redhat在RHEL6.5中集成对Docker的支持, 就连 Google 家的 Compute Engine 也支持 docker 在其之上运行, 最近百度也用 Docker 作为其PaaS的基础(不知道规模多大)。

一款开源软件能否在商业上成功，很大程度上依赖三件事 - 成功的 user case, 活跃的社区和一个好故事。 dotCloud 自家的 PaaS 产品建立在docker之上，长期维护 且有大量的用户，社区也十分活跃，接下来我们看看docker的故事。

环境管理复杂 - 从各种OS到各种中间件到各种app, 一款产品能够成功作为开发者需要关心的东西太多，且难于管理，这个问题几乎在所有现代IT相关行业都需要面对

云计算时代的到来 - AWS的成功, 引导开发者将应用转移到 cloud 上, 解决了硬件管理的问题，然而中间件相关的问题依然存在 (所以openstack HEAT和 AWS cloudformation 都着力解决这个问题)。开发者思路变化提供了可能性。

虚拟化手段的变化 - cloud 时代采用标配硬件来降低成本，采用虚拟化手段来满足用户按需使用的需求以及保证可用性和隔离性。然而无论是KVM还是Xen在 docker 看来, 都在浪费资源，因为用户需要的是高效运行环境而非OS, GuestOS既浪费资源又难于管理, 更加轻量级的LXC更加灵活和快速

LXC的移动性 - LXC在 linux 2.6 的 kernel 里就已经存在了，但是其设计之初并非为云计算考虑的，缺少标准化的描述手段和容器的可迁移性，决定其构建出的环境难于 迁移和标准化管理(相对于KVM之类image和snapshot的概念)。docker 就在这个问题上做出实质性的革新。