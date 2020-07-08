# Zookeeper Go 组件化最小实现

## 背景
Zetta项目需要迁移Hbase业务, 而Hbase依赖zk做一些存储配置。
希望使用Go实现一个提供最小功能的zk，并且用Zetta的Cluster支持，能作为一个组件嵌入Hbase的整体迁移。

etcd 实现过：zetcd, 主要提供的能力是保持使用zookeeper的api，但是使用etcd集群。

## HBase Zk 架构介绍
HBase 使用 Zookeeper 做分布式管理服务，来维护集群中所有服务的状态。Zookeeper 维护了哪些 servers 是健康可用的，并且在 server 故障时做出通知。
通常需要 3-5 台服务器来host。

Region Servers 和 在线 HMaster(active HMaster)和 Zookeeper 保持会话(session)。Zookeeper 通过心跳检测来维护所有临时节点(ephemeral nodes)。

每个 Region Server 都会创建一个 ephemeral 节点。HMaster 会监控这些节点来发现可用的 Region Servers，同样它也会监控这些节点是否出现故障。
HMaster 们会竞争创建 ephemeral 节点，而 Zookeeper 决定谁是第一个作为在线 HMaster，保证线上只有一个 HMaster。在线 HMaster(active HMaster) 会给 Zookeeper 发送心跳，不在线的待机 HMaster (inactive HMaster) 会监听 active HMaster 可能出现的故障并随时准备上位。
如果有一个 Region Server 或者 HMaster 出现故障或各种原因导致发送心跳失败，它们与 Zookeeper 的 session 就会过期，这个 ephemeral 节点就会被删除下线，监听者们就会收到这个消息。Active HMaster 监听的是 region servers 下线的消息，然后会恢复故障的 region server 以及它所负责的 region 数据。而 Inactive HMaster 关心的则是 active HMaster 下线的消息，然后竞争上线变成 active HMaster。

首次读写操作过程：
1. 客户端从 Zookeeper 那里获取是哪一台 Region Server 负责管理 Meta table。
2. 客户端会查询那台管理 Meta table 的 Region Server，进而获知是哪一台 Region Server 负责管理本次数据请求所需要的 rowkey。客户端会缓存这个信息，以及 Meta table 的位置信息本身。
3. 然后客户端回去访问那台 Region Server，获取数据。

对于以后的的读请求，客户端可以从缓存中直接获取 Meta table 的位置信息(在哪一台 Region Server 上)，以及之前访问过的 rowkey 的位置信息(哪一台 Region Server 上)，除非因为 Region 被迁移了导致缓存失效。这时客户端会重复上面的步骤，重新获取相关位置信息并更新缓存。

总结：
zk的能力总结如下：
1. zk 需要与几乎所有服务器进程保持会话（心跳检测），需要维护所有Server的连接状态，~~负责故障Node的重启~~不负责故障恢复，只是为故障恢复提供信息。
2. zk 需要维护一个 Meta table (特殊的 HBase 表)，包含了集群中所有 regions 的位置信息。
3. 需要解析客户端发过来的查询请求，查找到row key的位置信息后返回给客户端（涉及与客户端RPC请求的编解码）。



## 方案
整体来说，实现分为2个部分

1. 熟悉Hbase在运行时和zk需要的交互过程，了解相关接口


2. Back with Zetta Cluster，使用Zetta集群提供的接口，解析zk写入信息


## 具体个人行动建议
1. 熟悉zk API，事务处理原理
2. 熟悉Hbase 和 zk的交互
3. 熟悉Zetta 接口




