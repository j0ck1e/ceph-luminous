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
  1.1 什么是anisble
  ansible 是个开源的自动化平台，可以用来配置管理，应用部署，任务自动化等，也可以按照编排好的任务，在不同服务器上执行各自的任务，ansible允许你用一种简单的方法定义这些不同服务器。
  1.2 为什么使用ansible
  当选择自动化配置软件时面临一个问题，包括为什么需要自动化部署，为什么选择ansible，ansible有什么特点，下面我来简单解答这些问题
  ### 没有agent
  对比自动化部署的其他工具如chef,puppet和saltstack，ansible不需要在远程机器上安装客户端进程，而是通过ssh去管理机器，这是一个很nice的特点，意味这你不用事先在被控端安装软件了。
  ### 并行执行

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

bluestore直接对裸磁盘进行io操作，抛弃了文件系统，缩短了io路径，不在需要journal了，默认采用RocksDB作为元数据 K/V 引擎，直接解决了双写的问题，对纯机械硬盘的顺序写来说，性能有近2倍的提升。
bluestore作为灵活的对象存储后端，可以把元数据和用户数据放在不同的存储设备上，元数据有两部分组成：

db 和 WAL

db指的就是racksdb

WAL（write-ahead log）是racksdb预写日志

比较好的一种分布方法是把用户数据放在HDD或普通SSD上，db放在普通SSD,wal 放在NVMe SSD上

PG:
ceph 对集群中的所有资源采取池化管理(pool) , 可以针对一个pool设计一组cursh 规则，比如限制pool使用的存储资源， 备份策略，容灾域等。
为了实现不同pool之间的隔离，ceph不能将上层数据直接映射到磁盘，而是采用了一直中间结构PG
引入PG之后就实现了两级映射。
第一级是将前端数据进行整形之后均匀的映射至PG以实现PG的负载均衡，第二级是PG到OSD的映射,负责将数据相对均匀的写到osd中

## 优化
``` yaml
  - { name: fs.file-max, value: 26234859 }
  - { name: kernel.pid_max, value: 4194303 }
  - { name: vm.swappiness, value: 0 }
  - { name: vm.zone_reclaim_mode, value: 0 }
  - { name: vm.min_free_kbytes, value: "{{ vm_min_free_kbytes }}" }
  - { name: fs.aio-max-nr, value: 1048576}
```
针对内核参数的优化
### fs.file-max
表示系统最大的打开文件的数量，主要针对系统的限制，建议优化值为26234859
### kernel.pid_max
由于ceph-osd 进程会消耗大量的pid，默认值是32768，优化值为4194303
### vm.swappiness
当系统物理内存不足时，系统会把一部分磁盘空间当内存使用，这一部分空间叫交换分区(swap)，而我们实际看到的是，系统还有比较充足物理内存时，swap就已经开始使用了，这是因为vm.swappiness的默认值是60，也就是说当物理内存使用了60%或以上时就开始使用交换分区，这个值越大，内核换页就会越频繁，这会产生大量的磁盘io，因此在物理内存充足的情况下，要保证系统性能，把vm.swappiness的值设为0，禁用swap
### vm.zone_reclaim_mode
内存回收区域，指定内存区域的内存不足时，是从自己区域回收内存的还是其他区域回收内存，区域的概念来自NUMA，NUMA负责把内存和CPU划分成多区域，每个区域叫NODE,NODE之间互联，一个NODE内的CPU和内存访问比跨NODE访问快，NUMA解决了多核cpu获取内存时北桥响应瓶颈问题，而该参数设置为0，禁用区域回收
### min_free_kbytes
系统最小空闲内存，为了防止程序占用过多系统内存导致系统hung住或者是程序出现OOM(out of memory)的情况，设置一下系统保底内存，这里就设置成4G，当空闲内存不足4G时，系统会强制从cache中回收内存
### fs.aio-max-nr
这个参数用来限制活动的异步io请求队列长度，当aio-nr的值达到aio-max-nr值时，会触发EAGAIN（非阻塞IO），默认值是65536，优化后1048576




![vgy.me](https://vgy.me/KwVpjX.png)


