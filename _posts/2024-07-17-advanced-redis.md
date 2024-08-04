---
layout: post
title: 一些redis进阶知识
date: 2024-07-17
categories: blog
tags: [Tech]
description: 
---

最近突然接了一个性能优化的需求。

![截屏2024-07-21 22.05.46.png](..%2F..%2F..%2F..%2F..%2Fvar%2Ffolders%2Fpn%2Fpt8lztr55bb81tycwd9964bh0000gn%2FT%2FTemporaryItems%2FNSIRD_screencaptureui_EmNIif%2F%E6%88%AA%E5%B1%8F2024-07-21%2022.05.46.png)
大概是一个这样架构的微服务，对外提供根据主键查询数据的接口，对于此接口，性能要求较高，需支撑IAM系统中获取鉴权相关资源的操作。

还是先描述一下机器规格与采用的集群模式：
1，负载均衡器采用NAT模式，转发TCP数据包，在Server端与Client端皆观测为与LVS建立的TCP连接。负载均衡算法此处不表。
2，服务器规格为8U16G
3，Redis为采用Cluster模式的3 shard集群，每个shard一主一被
4，数据库为MongoDB，为一主二被的Replica Set。
我们有一个自建的缓存中间件，是通过本地缓存+Redis构成的一个双层缓存，此双层缓存提供了一个异步刷新的批量获取的方法：get_all_with_loading_function，