# TiDB 总体架构

## [How do we build TiDB](https://pingcap.com/blog-cn/how-do-we-build-tidb/)

<img src="pic\architecture.png" alt="avatar" style="zoom:80%;" />

## [三篇文章了解 TiDB 技术内幕 - 说存储](https://pingcap.com/blog-cn/tidb-internal-1/)

<img src="pic\tikv.png" alt="avatar" style="zoom:60%;" />

- TiKV 是一个全局有序的键值存储模型：
  - Key 和 Value 通过原始的字节数组来保存
  - 其中的键值对按照键的二进制顺序有序
- TiKV 的数据保存在 RocksDB 中（实现单机快速地存储到磁盘），通过 Raft 来实现容错
- Region：
  - 水平扩展的实现方式：
    - 对 Key 进行一致性 Hash
    - 分 Range ——> TiKV 的方式
  - TiKV 将整个键值空间划分成多段，每一段包含一系列连续的 Key - Value，这样的段称为 Region
    - 每个 Region 中保存的数据不超过 96MB，超过了则执行分裂
    - 每个 Region 中保存的数据不小于 20MB，小于则执行合并（B+Tree 结点有类似操作）
    - 每个 Region 可以用区间 $[StartKey,\  EndKey)$ 来表示
  - 以 Region 为单位，将数据分散在集群中的所有节点上，并尽量保证每个节点上的 Region 数量差不多
    - 增加新节点后，通过 PD 自动将其他节点上的 Region 调度过来，实现水平扩展和负载均衡
  - 以 Region 为单位实现 Raft 的复制和成员管理
    - 单个 Region 被复制多份分散到不同节点上，这些 Replica 构成了一个 Raft Group
    - 副本中其中一份会作为 Group 的 Leader，其他副本作为 Follower，读写都是通过 Leader 来发起的（Spanner 模型）
- MVCC：
  - 通过在 key 后面加上版本号来实现 
    - 同一个 key，版本号大的在前，版本号小的在后
  - 隔离级别：快照隔离（默认），读提交
- 事务模型：Percolator 模型
  - 2PL
  - 中央时间戳分配（Timestamp Allocator），每秒约 4 百万个时间戳，理论写入吞吐 2 百万每秒
  - 最初默认使用乐观锁，3.0.8 后默认使用悲观锁（更好的提供 MySQL 事务兼容性）

## [三篇文章了解 TiDB 技术内幕 - 说计算](https://pingcap.com/blog-cn/tidb-internal-2/)

- SQL 层架构：

  <img src="pic\TiDB-sql.png" alt="avatar" style="zoom:80%;" />

- 从关系模型到键值模型：

  - 转换的思路：看看关系模型中到底存了些什么，然后再考虑怎么转换
  - 对于一个表来说，需要存储的数据包括三部分：
    - 元信息
    - 数据：行存和列存
    - 索引：主键索引，次级索引
  - 关系操作：
    - 增删改查：操作数据的同时还要考虑更新索引
    - 点查，范围查：
      - 速度要快，开销要小
      - 精确定位（ID），快速执行
    - 索引利用

- 每个表都会分配一个 TableID，每个索引都会分配一个 IndexID，每一行也都有一个 RowID（有整形主键的话，会使用整形主键）

  <img src="pic\relation-to-kv.png" alt="avatar" style="zoom:80%;" />

  - ID 都是 int64 类型

  - TableID 在整个集群内唯一，IndexID 和 RowID 在表内唯一

    ```markdown
    # 行数据
    Key: tablePrefix{tableID}_recordPrefixSep{rowID}
    Value: [col1, col2, col3, col4]
    
    # 唯一索引数据
    Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
    Value: rowID
    
    # 非唯一索引数据
    Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue_rowID
    Value: null
    
    # 其中，xxPrefix 是字符串常量，用于区分命名空间
    var(
    	tablePrefix     = []byte{'t'}
    	recordPrefixSep = []byte("_r")
    	indexPrefixSep  = []byte("_i")
    )
    ```

  - 编码方案在 `codec` 包中

  - 通过编码前和编码后的比较关系不变，将一个表的所有行数据按照 RowID 的顺序映射到 TiKV 的键空间中，同理所以数据按照索引的列值顺序映射到键空间内

- TiDB 的整体架构：SQL on KV

  - TiKV 层作为 KV 引擎存储数据
  - TiDB 层都是无状态节点，本身不存数据，用于处理用户请求，执行 SQL 运算逻辑

- 分布式 SQL 运算：

  - 将计算节点尽量靠近存储节点，避免大量 RPC 调用
  - 下推到算子到存储节点计算，只返回有效的数据，避免无意义的网络传输

## [三篇文章了解 TiDB 技术内幕 - 谈调度](https://pingcap.com/blog-cn/tidb-internal-3/)

- 调度的基本操作：
  - 增加一个副本
  - 删除一个副本
  - Leader 角色在 Raft Group 中的切换
- TiKV 集群向 PD 汇报两类信息：
  - 每个 TiKV 节点定期汇报节点的整体信息
  - 每个 Raft Group 的 Leader 定期汇报 Region 的状态
  - 两类信息交互都是在心跳检测中完成的
- 调度要完成的工作：
  - 保证一个 Region 的副本数量正确
  - 一个 Raft Group 中的多个副本的位置调度
    - 通常为了容错，要保证多个副本不在同一节点上
    - 实际可能需要满足一些特殊需求：
      - 多节点部署在同一台物理机上
      - 多节点分布在多机架上，希望单个机架掉电时，也能保证系统可用性
      - 多节点分布在多个 IDC 中，希望单个机房掉电时，也能保证系统可用
    - 通过给节点配置 lables 来实现
  - 副本、Leader、热点在节点之间分布均匀
  - 保证各节点存储空间占用大致相等
  - 控制调度速度
  - 支持手动下线节点
- 调度的实现：
  - 通过心跳包收集信息，生成调度操作序列
  - 将操作通过心跳包回复给 Region Leader，并在后面的心跳包中监测执行结果
  - 发给 Region Leader 的操作只是建议，不一定能得到执行，具体执行由 Leader 根据自身状态来决定

## 其他

- https://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247484187&idx=1&sn=90a7ce3e6db7946ef0b7609a64e3b423&chksm=eb162471dc61ad679fc359100e2f3a15d64dd458446241bff2169403642e60a95731c6716841&scene=4