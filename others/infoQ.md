# 1号店系统
* 一致性
* Hedwig：核心中间件。RPC框架、异步消息平台、服务治理平台
* 技术人员优势：系统化思维

# ruby在京东云擎中的应用
* 京东云擎：ruby，event-machine，paas
* reactor设计模式：I/O，timer，signal，无需多线程


# JFS：京东云存储的存储后端，京东文件系统
* 为什么自主研发？
* 分期开展，缩短开发周期
* 电商的很多业务是database驱动的
* 强可靠、强一致、高可用
* 简单的才是可靠的
* Go写系统框架，C写单机引擎
* 分布式存储最难的问题是复制
* zookeeper管理集群
* JFS复制协议：Paxos协议变种，1primary+2follower
* 单机存储引擎：无随机写、无内存索引、编码在key里。优势：便于recovery；缺点：不便与垃圾收集
* 可靠性、一致性
