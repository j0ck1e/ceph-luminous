## 题目
基于Ansible的ceph集群部署与性能测试
## 摘要
  ceph诞生于2006年是sega 的博士论文作品，第一个稳定版本在2012年发布，6年的发展已经成为最热门的存储后端，在云计算的大环境下，分布式存储已经成为云计算基础建设中一个不可或缺的组成部分，ceph拥有很多引人注目的特性。

  1. ceph是一种软件定义存储，可以运行在所有主流Linux发行版上，如centos和ubuntu，unix上也可以运行，如freebsd，甚至是arm架构中。

  2. ceph是分布式的，所以可以有上百个节点，PB级的数据存储量，在大量磁盘的前提下，iops性能总体呈线性增长，

  2. ceph是一个统一存储系统，支持传统的块存储（block storage），文件存储协议(file system)，还有正在飞速发展的对象存储(object storage)

  理论上只要有存储需求，就可以部署一套ceph

  ansible是自动化配置管理工具，由python语言开发，因为简单实用等特点得到广泛关注，对运维开发人员来说，可以减少重复工作

Ceph was born in 2006. It is a doctoral dissertation of sega. The first stable version was released in 2012. Six years of development has become the most popular storage back end. Under the cloud computing environment, distributed storage has become cloud computing. As an integral part of infrastructure, ceph has a lot of eye-catching features.

  1. ceph is a software-defined storage that can run on all major Linux distributions, such as centos and ubuntu, and can also be run on unix, such as freebsd, or even the arm architecture.

  2. ceph is distributed, so it can have hundreds of nodes, PB-level data storage, on the premise of a large number of disks, the overall performance of iops linear growth.

  2. ceph is a unified storage system that supports traditional block storage, file system, and rapidly expanding object storage

  Theoretically, as long as there is a storage requirement, you can deploy a set of ceph

  Ansible is an automated configuration management tool developed by the python language. Because it is simple and practical, it has attracted wide attention and can reduce repetitive work for maintenance and development personnel.

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
- 性能测试
## 正文
1. ansible简单介绍
  1.1 什么是anisble
  ansible 是个开源的自动化平台，可以用来配置管理，应用部署，任务自动化等，也可以按照编排好的任务，在不同服务器上执行各自的任务，ansible允许你用一种简单的方法定义这些不同服务器。
  1.2 为什么使用ansible
  当选择自动化配置软件时面临一个问题，包括为什么需要自动化部署，为什么选择ansible，ansible有什么特点，下面我来简单解答这些问题
  ### 没有agent
  对比自动化部署的其他工具如chef,puppet和saltstack，ansible不需要在远程机器上安装客户端进程，而是通过ssh去管理机器，这是一个很nice的feature，意味这你不用事先在被控端安装软件了。
  ### 并行执行
  ansible执行一个操作，比如给100台机器重启某个服务，如果一个一个的执行，那执行效率就比较低了，但ansible也可以并行执行，这点对集群主机数量比较多时还是很有用的，能极大的缩短执行时间
  ### 自动报告
  建议在写tasks的给每个task写一段描述，这样在执行时ansible就会把每个task的执行信息很整洁的打印出来
  ### 使用简单
  ansible使用`YAML`作为配置语法，YAML的可读性很好是一个非常优雅的语言，而且很容易上手

  ### 为什么需要自动化配置？
  自动化配置可以把大量重复的部署工作减轻，不在需要运维人员去手动一台一台的配置服务器，能避免因为手动部署不慎造成的错误；自动部署是一套模板，能保证配置的统一

2. ceph简单介绍
  ceph是开源的分布式存储系统；其具备极好的可靠性，数据统一性，而且经过近几年ceph社区的蓬勃发展，ceph开辟了一个数据存储的途径。ceph支持横向扩展，不会出现单点故障。

  ceph 实现了三种存储形式
  - 对象存储(object storage) 主要操作对象是对象，k/v存储，http协议，使用调用api的方式操作对象，缺点是获取对象比较麻烦，需要先查阅api手册，不兼容现有模式，应用需要修改
  - 块存储(block storage) 主要操作对象是磁盘文件，磁盘文件作为磁盘直接挂载给虚拟机用，虚拟机里看到的就像插了一块硬盘，块存储的优点是可以直接以硬盘的形式挂载给虚拟机，缺点是数据共享很不方便。
  - 文件系统( file system) 支持POSIX接口，主要操作对象是文件和文件夹，类似smb和NFS的操作，可以挂载目录到本地，优点是共享，缺点是性能和文件锁

本编论文重点讲述块存储，毕设中也只实现和测试块存储

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

bluestore直接对裸磁盘进行io操作，抛弃了文件系统，缩短了io路径，不在需要journal了，默认采用RocksDB作为元数据 K/V 引擎，直接解决了双写的问题，对纯机械硬盘的顺序写来说，性能有近2倍的提升。BlueFS是一种简化的日志性文件系统，实现RocksDB定义的所有接口，用于持久化rocksdb运行过程中产生的.sst和.log文件，BlueFs在设计上支持将.sst和.log分开存储，如将.log文件放在高速的NVMe ssd上，来进行写加速

db 和 WAL

db用于存储bluestore内部元数据(.sst)

WAL（write-ahead log）存储racksdb预写日志.log文件(.log)

比较好的一种分布方法是把用户数据放在HDD或普通SSD上，db放在普通SSD,wal 放在NVMe SSD上

PG:
ceph 对集群中的所有资源采取池化管理(pool) , 可以针对一个pool设计一组cursh 规则，比如限制pool使用的存储资源， 备份策略，容灾域等。
为了实现不同pool之间的隔离，ceph不能将上层数据直接映射到磁盘，而是采用了一直中间结构PG
引入PG之后就实现了两级映射。
第一级是将前端数据进行整形之后均匀的映射至PG以实现PG的负载均衡，第二级是PG到OSD的映射,负责将数据相对均匀的写到osd中。

### RDB
在ceph设计之初被定义为一个分布式文件系统，但随着openstack 等云计算技术的兴起，ceph社区调整重心，往分布式块存储发展--RDB(RADOS Block Device)，RDB是ceph三大存储服务组件之一，也是最稳定使用最广泛的块存储方案了，rbd由对象数据和元数据两部分组成，元数据存储在上文的db中，元数据保存了自身容量，快照,锁等基本信息。

### Cache Tiering
分层缓存为ceph 客户端提供了更好的性能，由两个池组成，一个高速且昂贵存储设备池（如ssd）作为缓存层，一个普通且便宜的存储设备（机械硬盘）作为存储层，ceph对象处理决定对象放置，分层代理完成冷数据何时向存储层迁移，存储层和缓存层对客户端来说是完全是透明的
![cache tiering分层模型](http://docs.ceph.com/docs/master/_images/ditaa-2982c5ed3031cac4f9e40545139e51fdb0b33897.png)
缓存代理自动处理缓存层和存储层的数据迁移，但也可以手动配置迁移模式，只要迁移模式有两种
writeback mode: 当配置成writeback 模式时，ceph 客户端把数据写入缓存层就可以返回一个ACK
read-proxy mode:

## 优化
ceph一直存在不能完全发挥存储设备性能的问题，但可以通过更改一些系统参数来让ceph解除一些枷锁
``` yaml
  - { name: fs.file-max, value: 26234859 }
  - { name: kernel.pid_max, value: 4194303 }
  - { name: vm.swappiness, value: 0 }
  - { name: vm.zone_reclaim_mode, value: 0 }
  - { name: vm.min_free_kbytes, value: "{{ vm_min_free_kbytes }}" }
  - { name: fs.aio-max-nr, value: 1048576}
```
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
### 磁盘调度
Linux默认的磁盘调度算法是针对机械硬盘寻址时间的，而固态硬盘就不存在这个问题，因为固态硬盘的没有磁头，使用电门寻址的时间几乎可以忽略，使用系统默认的调度算法`deadline`反而会浪费cpu资源，所以使用`NOOP`代替默认算法
### 系统预读(read_ahead_kb)
系统会把一定的磁盘空间内容放入到内存中，因为内存的读写速度比磁盘快了很多很多倍，系统在读那部分内容的时候会直接到内存中读取，这样缩短了IO等待时间，系统默认的预读值为128KB，这不能满足分布式存储的需要，优化后的值为8192KB
### Jumbo frame（巨型帧）
先讲讲MTU(最大传输单元)，MTU定义了一个网卡接口每一次发送数据包的最大比特，默认MTU值一般为1500，超过1500Byte的数据包将被分片，ceph集群数据是靠网络传输的，一般ceph集群都要求使用万兆以太网，MTU1500在万兆以太网的ceph集群中的发包数量是很恐怖的，大量的数据包会消耗掉交换设备的资源，拖慢整个集群，所以需要减少数据分片，在ceph节点和交换设备上把mtu改成9000能有效减少数据包分片

## 部署
下图为拓扑图
![vgy.me](https://vgy.me/heE3pM.png)
ansible执行命令
`ansible-playbook deploy.yml -i hosts`
hosts 文件定义了主机组和主机变量
``` ini
[ceph-mon]
172.16.8.241  hostname=s241.jy.dev.tonyc.cn
[ceph-osd]
172.16.8.241  hostname=s241.jy.dev.tonyc.cn
172.16.8.242  hostname=s242.jy.dev.tonyc.cn
```
定义了两个主机组，ceph-mon和ceph-osd
主机变量定了了主机名，因为系统安装时没有没有自定义主机名，主机名会在初始化主机名和配置分发是使用。
这里的deploy.yml 在ansible中被称为playbook，以下为playbook中简要片段
``` yaml
- hosts: all
  gather_facts: False
  tasks:
    - name: get python2
      stat:
        path: /usr/bin/python
      register: python2_exist
    - name: install python2
      raw: apt-get -y install python-simplejson
      ignore_errors: yes
      when: python2_exist.failed == true
- hosts: all
  roles:
    - ceph-common
- hosts: ceph-mon
  roles:
    - ceph-mon
    - ceph-osd
- hosts: ceph-osd
  roles:
    - ceph-osd
```
简单来说上面就是指定以下什么主机组执行什么内容，比如hosts: all 表示所有主机，hosts: ceph-mon 表示ceph-mon组的主机，tasks 表示具体执行的内容，name对执行任务写一段简单的描述信息，之后是使用的模块名，所有的模块在ansible官方可以查到使用方法，模块名里面是模块的参数，roles表示执行对象充当什么角色，一个role由许许多多的tasks组成
执行结果：
![vgy.me](https://vgy.me/6AQc8c.png)
ansible执行修改过得会用changed标记，不符合判断条件的标记为skipping，已经符合执行过结果的标记为ok
ansible是基于状态的，比方说，用一个copy模块去copy一个文件到目标服务器上，ansible会先检测文件是否和目标文件相同，使用对比哈希值的方法，如果哈希值相同，说明文件是完全相同的，这时ansible会把执行结果直接标记成ok，而不会执行copy动作，或者说安装一个软件，ansible会先检测软件是否已经安装，如果已经安装过了，那就不需要执行安装动作了，直接标记成ok状态，不过也有特殊情况，比如使用shell,command,raw模块，这些模块都是直接执行命令的，并不会检测什么状态，所以我根据ansible最佳实践，尽量不使用这类模块，就算要使用这些模块的时候先自己写一个task去检测状态，把检测结果注册一个变量，然后执行模块时先去判断一下是否符合预设条件，符合条件才会执行，这样能解决对已存在内容的重复操作的问题，事例代码如下
``` yaml
- name: check bootstarp keys is exists
  stat:
    path: /var/lib/ceph/bootstrap-osd/{{ ceph.cluster_name }}.keyring
  register: key_exists
- name: generation keys
  shell: ceph-create-keys --cluster {{ ceph.cluster_name }} -i {{ monitor_name }} -t 30
  when: key_exists.stat.exists == false
```
先用stat模块获取文件状态，并把状态注册成变量key_exists，之后的shell模块下面有个when，表示当key_exists这个变量的状态是不存在时，才会去执行shell模块里面具体的命令

## 性能测试
### 硬件配置

|      | 172.16.8.241                 | 172.16.8.242                 |
| ---- | ---------------------------- | ---------------------------- |
| cpu  | E5-2650 v3 @ 2.30GHz 38 core | E5-2640 v3 @ 2.60GHz 30 core |
| 内存 | 128G                         | 32G                          |
| 硬盘 | Samsung SSD 850 Pro 1TB * 4  | Samsung SSD 850 Pro  1TB * 4 |
| 网络 | 10000Mb/s                    | 10000Mb/s                    |

故障域：host
副本数：2
先用4个SSD osd组成一个资源池，从资源池中创建rbd卷，使用fio测试rbd卷的性能，每次测好之后记录，然后往池中添加两个osd，能观察到osd个数和iops，延迟的关系
fio是一款开源的IO压力测试工具，主要可以用来测试磁盘设备的iops，吞吐量和延迟，支持多种引擎，fio的作者Jens Axboe是linux内核IO的主要开发者和维护者
测试命令：
`fio -filename=/dev/rbd0 -bs=64k -ioengine=libaio -iodepth=32 -numjobs=4 -direct=1 -thread -rw=randwrite -size=200G -runtime=50 -group_reporting -name=test`
其中的参数解释
filename: 对象名，也就是块设备映射后的块设备文件名
bs：每次读取或写入的字节数
ioengine： io引擎，为了测试多线程的读写性能，使用libaio异步引擎
iodepth: io队列深度, 
numjobs：多进程
direct：使用non-buffered io (使用O_DIRECT)，直写，不经过缓冲
thread: 使用多线程代替多进程
rw: 读或者写，这里使用随机写
size: 测试的文件大小
runtime: 运行时间
group_reporting: 当使用`numjobs`参数时，使用分组报告而不是按任务报告
测试数据如下

| osd  | iops  | lat(ms) |
| ---- | ----- | ------- |
| 4    | 12278 | 10.422  |
| 6    | 15682 | 8.16    |
| 8    | 18652 | 6.857   |

由上表数据不难看出iops会随着osd个数的增加而增加，延迟随osd个数增加而减小，换句话说就是集群越大，性能越好，在一定区间内差不多每增加两个osd，iops就会增加3000，延迟降低2ms。
对一块samsung 850 pro 1TB进行裸盘测试，同样的测试参数能达到6000左右的iops，如果是8个的话就是48000iops，但由于ceph测试副本数是2，所以无损耗iops值是24000，但实际只达到18000iops，性能损失近1/4，



## 参考文献
1. [anisble 官方文档](https://docs.ansible.com)
2. [ceph 官方文档](http://docs.ceph.com/docs/master/)
2. [Ceph Performance Tuning Checklist](http://accelazh.github.io/ceph/Ceph-Performance-Tuning-Checklist)
2. 谢果型，任焕文，等.ceph设计原理与实现[M].北京：机械工业出版社.2017年.8月：1-30
## 致谢
时光飞逝，转眼间四年的校园时光即将结束，回首过去四年的学习生活，我要诚挚的感谢所有关心和帮助过我的老师、同学和朋友们。
首先要衷心感谢我的指导老师朱宝华老师，我的论文是在朱老师的悉心指导下完成的，在论文选题及整个毕业设计的完成过程中朱老师都给予了我很多帮助，提供给我很多有价值的宝贵文献资料。朱老师在学术研究方面的深刻造诣、对学术研究的严谨态度以及在指导我们学习研究过程中的认真负责都让我印象深刻，同时更佩服不已。每次遇到自己解决不了的问题时，朱老师都会不厌其烦的指导我，在这里，我想向朱老师表示衷心的感谢！
## 注释

## 附件
https://github.com/j0ck1e/ceph-luminous

![vgy.me](https://vgy.me/KwVpjX.png)


