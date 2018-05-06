## 题目
基于Ansible的ceph集群部署与性能测试
## 摘要

## 关键字
`ceph`, `ansible`, `分布式`, `云计算`，`块存储`
## 目录
- ansible简单介绍
  - ansible运行原理
- ceph介绍
  - crush map
  - 对象存储引擎bluestore
  - 分布式块存储rbd
  - 性能调优
- host定义
- crush rule定义
- 性能测试
## 正文
1. ansible简单介绍
  1.1 ansible运行原理
  anisble 是通过ssh连接到target主机的，用户把ansible server的ssh 公钥ssh-copy-id到目标主机上，实现免密码登录。

2. ceph简单介绍
  2.1 crush map
  crush 算法依据cluster map 和数据分布规则(placement rule)把数据分布到磁盘中，cluster map是ceph集群层级逻辑描述文件，数据分布规则实现数据映射。
  cluster map描述常见形式：
  root-datacenter-rack-host-osd
  root 是ceph集群入口，比如定位一个osd，这个osd可以这样描述，江阴数据中心(datacenter)的A11机架(rack)的A11-3机器(host)的5号磁盘(osd)。
  这个层级是可修剪的，比较完整的层次类型:

  | 类型ID | 类型名称 | 类型ID | 类型名称   |
  | ------ | -------- | ------ | ---------- |
  | 0      | osd      | 6      | pod        |
  | 1      | host     | 7      | room       |
  | 2      | chassis  | 8      | datacenter |
  | 3      | rack     | 9      | region     |
  | 4      | row      | 10     | root       |
  | 5      | pdu      |        |            |

并非每个cluster map 都需要11个层级，根据实际情况自行修剪层级
cluster map关系使用一个二维映射表建立
<bucket, items>
每个层级是bucket，层级的所有下一级是items，当层级为底层如osd时，就没有下一层了，此时items为NULL，此时这个层级为device即物理存储设备
数据分布策略：
placement rule可以包含如下：
``` plain
take(root)
select(replicas, type)
emit(void)
```
take: 寻找集群入口作为后续输入默认是root
select: type 是容灾域类型，如rack(机架),host(机器)
容灾域类型决定了数据副本的分
比如副本数3，容灾域为rack，那么这三份相同的数据将分布到三台机架上，保证了其中一台机架发生故障，另外两个机架上的依然能提供完整的数据

2.2 对象存储引擎bluestore
介绍bluestore前，先简单谈谈前一代对象存储引擎filestore，在采用filestore存储引擎的集群中，所有读操作都是同步的，除非命中缓存了，而写操作会先在内存中进行缓存，但是由于内存的易失性，掉电之后数据就会消失，所以需要一个中间设备，这这个需要性能比一般的磁盘好很多倍而且掉电后数据不会丢失，如使用NVRAM或者NVMe SSD等高速存储设备，这种设备叫日志设备(journal)，有了journal以后，写操作将数据写到journal中就可以视作写入完成，之后在等到数据写入足够多时，再把数据从journal写到普通磁盘，这个操作叫做 落盘，落盘后journal上的空间就被释放掉了。

filestore最主要的问题：
因为引入了journal的缘故，数据会先写在journal设备上，然后再由journal向本地磁盘写入，这就造成了双写，如果是采用多副本的方案，双写带来的性能问题那就是灾难性的了。
filestore需要本地文件系统间接管理磁盘，所以需要把对象操作先转换为符合POSIX语义的文件操作，这样比较繁琐，相对的执行效率上就大打折扣。

bluestore直接对裸磁盘进行io操作，抛弃了文件系统，缩短了io路径，不在需要journal了，默认采用RocksDB作为元数据 K/V 引擎，直接解决了双写的问题，对纯机械硬盘的顺序写来说，性能有2倍的提升。
bluestore作为灵活的对象存储后端，可以把元数据和用户数据放在不同的存储设备上，元数据有两部分组成：

db 和 WAL

db指的就是racksdb

WAL（write-ahead log）是racksdb预写日志

比较好的一种分布方法是把用户数据放在HDD或普通SSD上，db放在普通SSD,wal 放在NVMe SSD上

PG:
ceph 对集群中的所有资源采取池化管理(pool) , 可以针对一个pool设计一组cursh 规则，比如限制pool使用的存储资源， 备份策略，容灾域等。
为了实现不同pool之间的隔离，ceph不能将上层数据直接映射到磁盘，而是采用了一直中间结构PG
引入PG之后就实现了两级映射。
第一级是将前端数据进行整形之后均匀的映射至PG以实现PG的负载均衡，第二级是PG到OSD的映射




![vgy.me](https://vgy.me/KwVpjX.png)
