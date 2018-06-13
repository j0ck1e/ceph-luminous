# 基于Ansible的ceph集群部署与性能测试

摘要： ceph诞生于2006年，是sega的博士论文作品，第一个稳定版本在2012年发布，6年的发展已经成为最热门的存储后端，在云计算的大环境下，分布式存储已经成为云计算基础建设中一个不可或缺的组成部分，ceph拥有很多引人注目的特性。
本文首先介绍ceph的三种存储形式：对象存储，块存储和文件系统，还有ceph的灵魂crush 算法中的crush map 和数据分布策略的介绍，然后ceph的新引擎bluestore，简单讲讲前一代引擎filestore的弊端和相比filestore的一些改进。ceph的性能一直是ceph新手所头疼的问题，论文里会整合网上的经验，对linux内核参数，磁盘和网络做一些简单的优化。
Ansible是自动化配置管理工具，由python语言开发，因为简单实用等特点得到广泛关注，对运维开发人员来说，可以减少重复工作。

关键字：ceph, ansible, 分布式系统, 云计算，块存储

ABSTRACT: Ceph was born in 2006 and is a sega doctoral dissertation. The first stable version was released in 2012. Six years of development has become the hottest storage backend. In the cloud computing environment, distributed storage has become a cloud. In an integral part of computing infrastructure, ceph has a lot of eye-catching features.
This article first introduces the three storage modes of ceph: object storage, block storage, and file system, as well as the introduction of crush's soul crush algorithm's crush map and data distribution strategy. Then ceph's new engine, bluestore, simply talks about the previous generation of engines. The drawbacks of filestore and some improvements over filestore. The performance of ceph has always been a headache for ceo novices. The paper will integrate online experience and make some simple optimizations for Linux kernel parameters, disks, and networking.

Keywords: ceph,ansible,distributed system,cloud computing,block storage

## 第1章 绪论
### 1.1 背景
  云计算发展的前提下，如何存储用户数据和保障数据可靠性？
ceph是开源的分布式存储系统；其具备极好的可靠性，数据统一性，而且经过近几年ceph社区的蓬勃发展，ceph开辟了一个数据存储的途径。ceph支持横向扩展，不会出现单点故障。
  ceph对数据存放有两种选择方案，一个是多副本，一个是纠删码，这两种方案都能实现数据的冗余，极大的保障了数据的物理安全性。
  数据对企业何其重要，在大数据云计算时代，一个可靠的数据存储方案是每个企业所必须的。

### 1.2 选题目的
  在云计算环境中，以OpenStack为例，openstack的nova，glance，cinder服务需要一个稳定可靠的后端用以存储数据，我们可以以file的形式存储在本地磁盘上，但是这样的话数据迁移，共享数据和数据物理安全性是个问题，如果发生意外断电或磁盘坏道，将会对数据造成不可预知的风险，所以我们有了共享存储用来解决这个问题，典型如NFS网络文件系统，使用NFS这种方案虽然解决了数据共享问题，但是依然不能成为一个企业存储方案，因为NFS存储数据不可靠，性能不够好，而且NFS是文件系统的一种实现方式，存在文件锁的问题。
  综上所述，如何实现一个稳定可靠，适应云计算大数据的统一存储方案？ceph是一个分布式存储开源项目，使用ceph可以很好的解决数据存储的问题，ceph手工部署是一项非常复杂的工作，如果配置不当，可能造成一些性能问题，所有可以通过ansible这样的工具实现自动化部署，减少手动部署的失误。

## 第二章 技术实现
### 2.1 实现目的
  ceph的块存储服务是目前最好的块存储方案，块存储可以作为openstack的存储后端，当部署云计算环境前，需要先把存储环境部署好，然后部署云计算环境时直接对接存储集群。
  为了部署块存储环境，需要先对系统进行优化，因为部署过程中会启动ceph服务，这些服务会在启动的过程中读取这些优化好的参数。
  如何让块存储环境性能达到线上使用的需求，需要对块存储环境进行性能测试，主要测出iops和延迟，应对线上环境的需求，带宽需要达到300Mb/s以上,业务上的延迟25ms以内。

### 2.2 ansible
  ansible 是个开源的自动化平台，可以用来配置管理，应用部署，任务自动化等，也可以按照编排好的任务，在不同服务器上执行各自的任务。
  自动化配置可以把大量重复的部署工作减轻，不在需要运维人员去手动一台一台的配置服务器，能避免因为手动部署不慎造成的错误；自动部署是一套模板，能保证配置的统一。
  对比自动化部署的其他工具如chef,puppet和saltstack，ansible不需要在远程机器上安装客户端进程，而是通过ssh去管理机器，这是一个很nice的feature，意味这你不用事先在被控端安装软件了。
  ansible执行一个操作，比如给100台机器重启某个服务，如果一个一个的执行，那执行效率就比较低了，但ansible也可以并行执行，这点对集群主机数量比较多时还是很有用的，能极大的缩短执行时间。
  在写tasks时可以给每个task写一段描述，这样在执行时ansible就会把每个task的执行信息很整洁的打印出来。
  Ansible使用`YAML`作为配置语法，YAML的可读性很好是一个非常优雅的语言，而且很容易上手。
  Ansible官方有上百个模块可以直接使用，这些模块基本满足使用需求，而且也可以自己开发模块使用。
  Ansible 使用时首先需要把public SSH key传到被控主机authorized_keys文件里，实现免密码登录。

### 2.3 openstack cinder 服务
  cinder服务是openstack的卷服务，卷服务为云计算实例提供块设备，比如创建实例需要的磁盘，或者是实例提升存储需求，添加一块新磁盘，cinder 服务主要由cinder-api 和 cinder-volume 组成，cinder-api 负责客户端对cinder api的调用，而cinder-volume 实现对接ceph后端，客户端通过api请求，cinder-api收到请求之后会先去调用cinder-volume服务，cinder-volume实现卷设备的创建，cinder-api把请求结果返回给客户端。
  cinder 服务有多种存储后端，file是cinder默认的存储方式，直接把块设备以文件的形式存储在本地磁盘，其他还有rbd，http等形式，如果需要对接ceph，把rbd配置成默认后端。

### 2.4 ceph
#### 2.4.1  ceph国内发展现状
  近年来，大型企业以及开源社区不断的推动中国开源技术的发展，今天的中国已然成为OpenStack & Ceph等开源技术大放光彩的乐土。
  Ceph作为全球最火热的开源分布式存储项目，同样在中国的发展也是非常火热，不断开始在不同领域不同行业及客户系统相融合。典型应用在国内一线互联网公司以及运营商、政府、金融、广电、能源、游戏、直播等行业。 
  当前中国Ceph形势对比前几年已经发生了决定性的变化，随着国内越来越多的各行业用户的使用，足以见证它的稳定性可靠性。Ceph中国用户生态已然形成，可以看到国内如：中国移动、腾讯、阿里、网易、乐视、携程、今日头条、中国电信、中兴、恒丰银行、平安科技、YY、B站、360等。正是由于众多用户的使用验证了它的稳定性和可靠性的同时也促进了Ceph的进步。
  Ceph中国社区成立于2014年7月，包括线上的QQ群/微信群、翻译小组、订阅号和线下沙龙。Ceph中国线下沙龙已经成功举办多次，邀请国内一线工程师专家讲最实际最落地的采坑经验，首次提出Ceph中国行布道之旅，全面深入的在中国多座城市进行Ceph & OpenStack布道。目前Ceph中国已经成为国内最专业的Ceph技术交流社区平台。

#### 2.4.2 ceph简单介绍
  ceph是开源的分布式存储系统；其具备极好的可靠性，数据统一性，而且经过近几年ceph社区的蓬勃发展，ceph开辟了一个数据存储的途径。ceph支持横向扩展，不会出现单点故障。
  ceph对比其他存储方案GlusterFS，只能说各有优势，ceph的优势在于灵活拓展，可以结合到非Linux环境中，ceph目前是公认最好的块存储方案，而我们主要就用到块存储。

#### 2.3.3 ceph三种存储形式
对象存储(object storage)：
  主要操作对象是对象，k/v存储，http协议，使用调用api的方式操作对象，缺点是获取对象比较麻烦，需要先查阅api手册，不兼容现有模式，应用形式需要修改。
块存储(block storage)：
  主要操作对象是磁盘文件，磁盘文件作为磁盘直接挂载给虚拟机用，虚拟机里看到的就像插了一块硬盘，块存储的优点是可以直接以硬盘的形式挂载给虚拟机，缺点是数据共享很不方便。
文件系统(file system)：
  支持POSIX接口，主要操作对象是文件和文件夹，类似smb和NFS的操作，可以挂载目录到本地，优点是共享，缺点是性能和文件锁。

#### 2.3.4 crush map
  crush 算法依据crush map 和数据分布规则(placement rule)把数据均衡分布到各个磁盘中，crush map是ceph集群层级逻辑描述文件，数据分布规则实现数据映射。

| 类型ID | 类型名称 | 类型ID | 类型名称   |
| ------ | -------- | ------ | ---------- |
| 0      | osd      | 7      | room       |
| 1      | host     | 8      | datacenter |
| 2      | Chassis  | 9      | Region     |
| 3      | Rack     | 10     | Root       |
| 4      | Row      |        |            |
| 5      | Pdu      |        |            |
| 6      | Pod      |        |            |

cluster map描述常见形式：
`root-datacenter-rack-host-osd`
  root 是ceph集群入口，比如定位一个osd，这个osd可以这样描述，1号数据中心(datacenter)的A11机架(rack)的A11-3机器(host)的5号磁盘(osd)。
cluster map关系使用一个二维映射表建立
`<bucket, items>`
  每个层级是bucket，层级的所有下一级是items，当层级为底层如osd时，就没有下一层了，此时items为NULL，此时这个层级为device即物理存储设备。

Placement rule是数据分布策略主要用来设置容灾域和副本数
placement rule包含如下参数：
``` plain
take(root)
select(replicas, type)
emit(void)
```
take: 寻找集群入口作为后续输入默认是root。
select: type 是容灾域类型，如rack(机架),host(机器)。
容灾域类型决定了数据副本的分布。
比如副本数3，容灾域为rack，那么这三份相同的数据将分布到三台机架上，保证了其中一台机架发生故障，另外两个机架上的依然能提供完整的数据。

#### 2.3.5 对象存储引擎bluestore
  介绍bluestone前，先简单谈谈前一代对象存储引擎filestore，在采用filestore存储引擎的集群中，所有读操作都是同步的，除非命中缓存了，而写操作会先在内存中进行缓存，但是由于内存的易失性，掉电之后数据就会消失，所以需要一个中间设备，这这个需要性能比一般的磁盘好很多倍而且掉电后数据不会丢失，如使用NVRAM或者NVMe SSD等高速存储设备，这种设备叫日志设备(journal)，有了journal以后，写操作将数据写到journal中就可以视作写入完成，之后在等到数据写入足够多时，再把数据从journal写到普通磁盘，这个操作叫做落盘，落盘后journal上的空间就被释放掉了。

![filestore-vs-bluestore](https://i0.wp.com/ceph.com/wp-content/uploads/2017/08/filestore-vs-bluestore-2.png?w=813&ssl=1)

filestore最主要的问题：
  因为引入了journal的缘故，数据会先写在journal设备上，然后再由journal向本地磁盘写入，这就造成了双写，如果是采用多副本的方案，双写带来的性能问题那就是灾难性的了。
filestore需要本地文件系统间接管理磁盘，所以需要把对象操作先转换为符合POSIX语义的文件操作，这样比较繁琐，相对的执行效率上就大打折扣。

bluestore的优势：
  bluestore直接对裸磁盘进行io操作，抛弃了文件系统，缩短了io路径，不在需要journal了，默认采用RocksDB作为元数据 K/V 引擎，直接解决了双写的问题，对纯机械硬盘的顺序写来说，性能有近2倍的提升。BlueFS是一种简化的日志性文件系统，实现RocksDB定义的所有接口，用于持久化rocksdb运行过程中产生的.sst和.log文件，BlueFs在设计上支持将.sst和.log分开存储，如将.log文件放在高速的NVMe ssd上，来进行写加速。

db 和 WAL
  db用于存储bluestore内部元数据(.sst)
WAL（write-ahead log）存储racksdb预写日志.log文件(.log)
比较好的一种分布方法是把用户数据放在HDD或普通SSD上，db放在普通SSD,wal放在NVMe SSD上

## 第3章 方案设计
### 3.1 需求分析
  为了满足云计算环境中的云实例对存储的要求，需要创建一个实例的时候不影响其他已存在实例业务的正常运行，为了保障数据的可靠性，使用基于多副本的方案，虽然纠删码方案对存储空间使用率上比多副本方案好，但是多副本以其性能和数据存储安全方面的优势成为首选方案。为了提供足够的性能，使用全闪存做ceph环境，环境中使用8块三星850 pro 1TB 作为osd。
  云计算实例必须要有制作快照、回滚快照的功能，这个功能由ceph实现，快照功能是rbd 命令创建，是ceph自带的功能。

### 3.2 功能分析
  为了能实时的观测到每个osd的数据吞吐量，使用ceph 自带的 web界面bashboard，
实现这个需求需要先安装ceph-mgr，这个服务是从ceph-mon中分离出来，给ceph提供类似插件的功能，ceph-mgr可以开启多个插件，而bashboard这个插件刚好能满足这个需求。

  部署ceph环境，由于有多台节点，需要保证cephx key在多个节点中保持同步，这个功能由ansible的fetch模块实现，这个模块把cephx key先获取到ansible 主机上，然后从ansible主机上向目标节点同步。

## 第4章 部署
### 4.1 部署流程
  在部署ceph集群之前需要先对内核参数，磁盘和网络进行优化，在实例部署的过程中，ansible会启动ceph多个进程，这些进程会在启动时读取相应的系统参数。
  完成优化后，需要安装ceph-mon软件包，ceph-mon主要对ceph集群数据进行控制，同时在安装中需要生成cephx key，有了cephx key之后就可以用ceph命令查看修改集群信息，cephx key一旦生成，需要在整个集群中保持完全一致，完全一致才能对ceph 集群操作。
  接着安装ceph-mgr，ceph-mgr是为了简化ceph-mon的功能，让ceph-mon的功能更专注于ceph集群数据，官方把ceph-mon功能做了分离，ceph-mgr 提供了一些插件，开启bashboard插件，可以在7000端口开启一个官方的web界面，这个界面显示了一些ceph环境的基础信息，如osd个数，集群健康状态，有一个页面可以显示ceph每个osd的吞吐量，这个功能很重要，因为有个这个界面可以很方面的显示集群的状态。
  Ceph-osd是ceph 的osd服务，使用ceph-osd可以向集群添加osd等操作，使用ceph-volume命令向集群添加osd，这个操作需要先把磁盘分区表类型改成gpt，然后创建一个分区，最后使用ceph-volume create 创建一个osd。
  接着创建资源池，ceph资源池可以设置副本数和pg数，有了这些可以报数据均衡的映射到磁盘，然后在资源池里创建rbd块，由于内核版本问题，rbd块默认的feature不能正常挂载，所以需要用rbd feature disable把这些feature关闭掉。关闭好后即可挂载,挂载成功后可以在系统里看到一个/dev/rbd0,这是对象就是对这个块设备测试的。

### 4.2 优化
#### 4.2.1 优化目的
  在云计算环境中的实例需要大量的磁盘容量和磁盘性能，磁盘容量问题很好解决，只需要往集群里面添加磁盘就行了，但是集群整体的性能就需要不断的优化了，考虑到后期的实例数量比较多，后期磁盘带宽大概需要达到300Mb/s，延迟需要达到业务上无感知在25ms左右。
  Ceph可能受到系统参数的限制而影响到性能，可以通过更改一些系统参数来让ceph性能提升。

#### 4.2.2 优化参数概述
内核优化参数：
内核参数修改写入/etc/sysctl.conf文件，执行sysctl -p生效
fs.file-max：
  表示系统最大的打开文件的数量，主要针对系统的限制，建议优化值为26234859
kernel.pid_max：
  由于ceph-osd 进程会消耗大量的pid，默认值是32768，优化值为4194303
vm.swappiness：
  当系统物理内存不足时，系统会把一部分磁盘空间当内存使用，这一部分空间叫交换分区(swap)，而我们实际看到的是，系统还有比较充足物理内存时，swap就已经开始使用了，这是因为vm.swappiness的默认值是60，也就是说当物理内存使用了60%或以上时就开始使用交换分区，这个值越大，内核换页就会越频繁，这会产生大量的磁盘io，因此在物理内存充足的情况下，要保证系统性能，把vm.swappiness的值设为0，禁用swap。
vm.zone_reclaim_mode：
  内存回收区域，指定内存区域的内存不足时，是从自己区域回收内存的还是其他区域回收内存，区域的概念来自NUMA，NUMA负责把内存和CPU划分成多区域，每个区域叫NODE,NODE之间互联，一个NODE内的CPU和内存访问比跨NODE访问快，NUMA解决了多核cpu获取内存时北桥响应瓶颈问题，而该参数设置为0，禁用区域回收。
min_free_kbytes：
  系统最小空闲内存，为了防止程序占用过多系统内存导致系统宕机或者是系统OOM(out of memory)的情况，设置一下系统保底内存，这里就设置成4G，当空闲内存不足4G时，系统会强制从cache中回收内存。
fs.aio-max-nr：
  这个参数用来限制活动的异步io请求队列长度，当aio-nr的值达到aio-max-nr值时，会触发EAGAIN（非阻塞IO），默认值是65536，优化后1048576。

磁盘调度：
  Linux默认的磁盘调度算法是针对机械硬盘寻址时间的，而固态硬盘就不存在这个问题，因为固态硬盘的没有磁头，使用电门寻址的时间几乎可以忽略，使用系统默认的调度算法`deadline`反而会浪费cpu资源，所以使用`NOOP`代替默认算法。

系统预读(read_ahead_kb)
  系统会把一定的磁盘空间内容放入到内存中，因为内存的读写速度比磁盘快了很多很多倍，系统在读那部分内容的时候会直接到内存中读取，这样缩短了IO等待时间，系统默认的预读值为128KB，这不能满足分布式存储的需要，优化后的值为8192KB

网络优化
Jumbo frame（巨型帧）
  先讲讲MTU(最大传输单元)，MTU定义了一个网卡接口每一次发送数据包的最大比特，默认MTU值一般为1500，超过1500Byte的数据包将被分片，ceph集群数据是靠网络传输的，一般ceph集群都要求使用万兆以太网，MTU1500在万兆以太网的ceph集群中发包数量是很恐怖的，大量的数据包会消耗掉交换设备的资源，拖慢整个集群，所以需要减少数据分片，在ceph节点和交换设备上把mtu改成9000有效减少数据包分片。

#### 4.2.3 ansible优化实现
这些优化参数对内核限制有效，使用ansible模块sysctl，对优化参数进行loop遍历生效
``` yaml
os_tuning_params:
  - { name: fs.file-max, value: 26234859 }
  - { name: kernel.pid_max, value: 4194303 }
  - { name: vm.swappiness, value: 0 }
  - { name: vm.zone_reclaim_mode, value: 0 }
  - { name: vm.min_free_kbytes, value: "{{ vm_min_free_kbytes }}" }
  - { name: fs.aio-max-nr, value: 1048576}

- name: tuning kern params
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_set: yes
    reload: yes
  loop: "{{ os_tuning_params }}"

```

下面代码片段片段是对磁盘的优化
```yaml
- name: change disk scheduler persistent
  blockinfile:
    path: /lib/udev/rules.d/60-ssd-scheduler.rules
    create: yes
    block: |
      SUBSYSTEM=="block", ATTR{device/model}=="Samsung SSD 850", ACTION=="add|change", KERNEL=="sd[a-d]", ATTR{queue/scheduler}="noop",ATTR{queue/read_ahead_kb}="8192"
```
  这段代码主要实现了磁盘调度算法的指定和磁盘系统预读的大小,网上的方法都是直接修改/proc目录里的内容的，重启设置的所有内容就丢失了，所以为了重启也能生效，使用blockinfile模块对udev进行修改

  网卡mtu的修改，这个需要在服务器和交换机上统一修改，交换机在服务器所连接的端口模式下配置mtu，服务器在网卡配置文件里改一下就行了。

### 4.3 部署ceph-mon
首先安装ceph-mon软件包
ceph-mon程序相当于ceph集群的主控程序
``` yaml
- name: install packages
  package:
    name: "{{ item_package }}"
  loop:
- ceph-mon
  loop_control:
    loop_var: item_package
```
  使用package模块安装ceph-mon软件包，使用package模块而不是apt模块是因为package模块整合了apt模块和yum模块等包管理工具的基本功能，使用这个模块安装软件不需要考虑目标主机使用的是什么包管理工具

### 4.4 cephx key同步问题的解决
同步cephx key代码片段
``` yaml
- name: copy keys to the ansible server
  fetch:
    src: "{{ item_bootstrap }}"
    dest: "{{ ceph.fetch_directory }}/{{ ceph.fsid }}/{{ item_bootstrap }}"
    flat: yes
  loop:
    - /var/lib/ceph/bootstrap-osd/{{ ceph.cluster_name }}.keyring
    - /var/lib/ceph/bootstrap-rgw/{{ ceph.cluster_name }}.keyring
    - /var/lib/ceph/bootstrap-mds/{{ ceph.cluster_name }}.keyring
    - /var/lib/ceph/bootstrap-rbd/{{ ceph.cluster_name }}.keyring
  loop_control:
    loop_var: item_bootstrap
```
  为了使所有节点上的cephx key完全一致，这里使用fetch模块把生成的key先获取到ansible 服务器上，源地址使用loop遍历目标主机要获取文件的路径，目的地址写ansible 服务器上想要存放key的路径。
  之后同步的时候，只需要使用copy模块把这些key直接copy到目标节点的相应目录的就行了，这样就解决了cephx key同步的问题

### 4.5 部署ceph-mgr
  ceph-mgr 的开发目的主要是为了把ceph-mon的功能迁移到ceph-mgr中，使ceph-mon的功能更专注于集群的数据层面的控制，部署ceph-mgr是为了开启ceph-mgr中的一个模块：dashboard
Dashboard是ceph的一个内置web界面，可以很方面的展示一些osd的性能数据，比如每个osd的读写带宽。
``` yaml
- name: install ceph mgr
  package:
    name: ceph-mgr
    state: present

- name: enable_mgr_modules
  shell: ceph mgr module enable {{ item }}
  loop: "{{ ceph.enable_mgr_modules }}"
  when: item not in ((enabled_ceph_mgr_modules.stdout | from_json).enabled_modules)

```
部署ceph-mgr比较简单，先安装ceph-mgr软件包，然后enable dashboard模块就行了

### 4.6 部署ceph-osd
  这个操作是为了向集群添加osd，以shell方式实现，比方说现在有一块磁盘/dev/sdb，我们需要把这块磁盘添加到集群里面去，先安装软件包:
``` bash
apt install –y ceph-osd parted
```
parted 软件主要是用来给磁盘设置分区表类型和完成分区的。
``` bash
parted –s /dev/sdb mklabel gpt mkpart 1 ext2 0% 100%
```
parted 先把磁盘分区表类型改成gpt，然后把分了一个区，这个分区使用ext2文件系统并使用磁盘所有空间。
经过上面的步骤一块磁盘就准备好作为一个osd了，接着运行ceph-volume
``` bash
ceph-volume lvm create --data /dev/sdb1
```
![vgy.me](https://vgy.me/YBs1aX.png)

这条命令运行之后会有以下主要结果：
  首先sdb1这个分区会作为lvm的逻辑卷，然后从这个逻辑卷创建一个osd，把这个osd添加进crush map的相应host下面并激活，这样一个osd就添加完成了，使用相应的步骤添加其他硬盘，如图5-1中，已经添加了7个osd了，在osd tree可以看到每个osd所在的host

### 4.7 创建资源池及后续
  ceph资源池可以对不同需求的资源池指定不同的策略，比如副本数，pg数等
由于测试节点只有两台，所以副本数为2，首先在ceph配置文件里添加一行：
osd pool default = 2
这样的话创建的池的副本数就默认是两个副本了。
  下面这个创建pool的命令，同时指定pool的pg数
ceph osd pool create test_pool 512 512
资源池的名字是test_pool
pg 和gpg为512

  执行好之后资源池就创建成功了,接着在资源池中创建块设备并挂载：
rbd create test_block0 –-size 10240 –-pool test_pool
rbd map test_pool/test_block0

![vgy.me](https://vgy.me/PIq5sZ.png)

从图中可以看出这个块的大小和特性，由于节点系统内核版本为4.4，使用不了这么多特性，使用rbd feature disable test_pool关闭这些特性。
关闭好特性之后，rbd就能成功挂载，挂载成功后可以用parted –l 看到个/dev/rbd0的块设备，性能测试就是对这个块设备进行的。

### 4.8 ansible执行操作
  上面的部署是细分后的大致步骤，最后在总的讲讲ansible是怎么执行的
执行部署命令：
`ansible-playbook deploy.yml -i hosts`
hosts 文件定义了主机组和主机变量：
``` ini
[ceph-mon]
172.16.8.241  hostname=s241.jy.dev.tonyc.cn
[ceph-osd]
172.16.8.241  hostname=s241.jy.dev.tonyc.cn
172.16.8.242  hostname=s242.jy.dev.tonyc.cn
```
定义了两个主机组，ceph-mon和ceph-osd
主机变量定了了主机名，因为系统安装时没有没有自定义主机名，主机名会在初始化主机名和配置分发时使用。
这里的deploy.yml 在ansible中被称为playbook，以下为playbook中简要片段：
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
  简单来说上面就是指定什么主机组执行什么内容，比如hosts: all 表示所有主机，hosts: ceph-mon 表示ceph-mon组的主机，tasks 表示具体执行的内容，name对执行任务写一段简单的描述信息，之后是使用的模块名，所有的模块在ansible官方可以查到使用方法，模块名里面是模块的参数，roles表示执行对象充当什么角色，一个role由许许多多的tasks组成。

![vgy.me](https://vgy.me/Sfk5vU.png)

ansible执行修改过得会用changed标记，不符合判断条件的标记为skipping，已经符合执行过结果的标记为ok。
  ansible是基于状态的，比方说，用一个copy模块去copy一个文件到目标服务器上，ansible会先检测文件是否和目标文件相同，使用对比哈希值的方法，如果哈希值相同，说明文件是完全相同的，这时ansible会把执行结果直接标记成ok，而不会执行copy动作，或者说安装一个软件，ansible会先检测软件是否已经安装，如果已经安装过了，那就不需要执行安装动作了，直接标记成ok状态，不过也有特殊情况，比如使用shell,command,raw模块，这些模块都是直接执行命令的，并不会检测什么状态，所以我根据ansible最佳实践，尽量不使用这类模块，就算要使用这些模块的时候先自己写一个task去检测状态，把检测结果注册一个变量，然后执行模块时先去判断一下是否符合预设条件，符合条件才会执行，这样能解决对已存在内容的重复操作的问题，事例如下：

``` yaml
- name: check bootstarp keys is exists
  stat:
    path: /var/lib/ceph/bootstrap-osd/{{ ceph.cluster_name }}.keyring
  register: key_exists
- name: generation keys
  shell: ceph-create-keys --cluster {{ ceph.cluster_name }} -i {{ monitor_name }} -t 30
  when: key_exists.stat.exists == false
```
  先用stat模块获取文件状态，并把状态注册成变量key_exists，之后的shell模块下面有个when，表示当key_exists这个变量的状态是不存在时，才会去执行shell模块里面具体的命令。

## 第5章 性能测试
  为了检验ceph是否达到理想性能，需要对ceph进行性能测试，本篇论文主要实现块存储，对挂载的/dev/rbd0块设备进行测试。

### 5.1 硬件配置
|      | 172.16.8.241                 | 172.16.8.242                 |
| ---- | ---------------------------- | ---------------------------- |
| cpu  | E5-2650 v3 @ 2.30GHz 38 core | E5-2640 v3 @ 2.60GHz 30 core |
| 内存 | 128G                         | 32G                          |
| 硬盘 | Samsung SSD 850 Pro 1TB * 4  | Samsung SSD 850 Pro  1TB * 4 |
| 网络 | 10000Mb/s                    | 10000Mb/s                    |

测试环境
故障域：host
副本数：2
  先用4个SSD osd组成一个资源池，从资源池中创建rbd卷，使用fio测试rbd卷的性能，每次测好之后记录，然后往池中添加两个osd，能观察到osd个数和iops，延迟的关系。

### 5.2 测试工具
  fio是一款开源的IO压力测试工具，主要可以用来测试磁盘设备的iops，吞吐量和延迟，支持多种引擎，fio的作者Jens Axboe是linux内核IO的主要开发者和维护者。

### 5.3 测试命令
`fio -filename=/dev/rbd0 -bs=64k -ioengine=libaio -iodepth=32 -numjobs=4 -direct=1 -thread -rw=randwrite -size=200G -runtime=50 -group_reporting -name=test`
其中的参数解释：
filename: 对象名，也就是块设备映射后的块设备文件名
bs：每次读取或写入的字节数
ioengine： io引擎，为了测试多线程的读写性能，使用libaio异步引擎
iodepth: io队列深度
numjobs：多进程
direct：使用non-buffered io (使用O_DIRECT)，直写，不经过缓冲
thread: 使用多线程代替多进程
rw: 读或者写，这里使用随机写
size: 测试的文件大小
runtime: 运行时间
group_reporting: 当使用`numjobs`参数时，使用分组报告而不是按任务报告

### 5.4 测试数据
| osd  | iops  | lat(ms) |
| ---- | ----- | ------- |
| 4    | 12278 | 10.422  |
| 6    | 15682 | 8.16    |
| 8    | 18652 | 6.857   |

### 5.5 测试总结
  由上表数据不难看出iops会随着osd个数的增加而增加，延迟随osd个数增加而减小，换句话说就是集群越大，性能越好，在一定区间内差不多每增加两个osd，iops就会增加3000，延迟降低2ms。
  对一块samsung 850 pro 1TB进行裸盘测试，同样的测试参数能达到6000左右的iops，如果是8个的话就是48000iops，但由于ceph测试副本数是2，所以无损耗iops值是24000，但实际只达到18000iops，性能损失1/4左右。
  这个性能损失还是比较大的，但对于一个追求数据安全的存储集群来说还是可以接受的，
  以上数据来自2017年8月我对ceph纯ssd集群的一个简单测试。

## 第6章 总结与展望
  以上是我对ceph和ansible的一些简单研究，ansible以其使用简单，没有agent的特点，受到越来越多运维和开发的青睐，ansible还有上百个模块，每个模块可以实现不一样的功能，当然也可以自己写模块，不过ansible有一个挺大缺点，就是执行速度上不如同类的puppet和saltstack，同一个tasks里的执行先后顺序，是按照文件中定义的顺序依次执行的。
  虽然现在ceph还有些性能问题，这也可能是我优化不够到位造成的，但是ceph以其卓越的可靠性，成为OpenStack官方推荐的后端存储，其中rbd功不可没，OpenStack实例的磁盘对接ceph，使得OpenStack实例迁移非常方便，迁移一台实例只需要花上几秒钟，而且迁移速度还不会受到磁盘大小的影响；crush map是ceph的灵魂，crush map定义了基于主机的故障域，基于机柜，机架或者数据中心等等，还有两种存储方案，多副本和纠删码，多副本就是阿里云所说的那个99.999999%可靠性的那种存储方式，如果是多副本是raid 1的话，纠删码就是raid 5 或10了，纠删码的特点是空间利用率高，但性能不如多副本方案；bluestore是ceph从jewel版本引入的新型对象存储引擎，中间经过了一个大版本的迭代，在L版终于稳定下来，并取代的了filestore成为默认的后端存储引擎，bluestore解决了filestore的两个痛点，依赖文件系统和写放大；cache tiering特性对ceph三种存储组件中的对象存储有非常大的性能提升，还有就是ceph优化，ceph优化分linux系统优化和ceph配置优化，这是一个非常困难的工作，需要对linux内核参数，linux文件系统，分布式存储系统，计算机网络，linux系统调优和各钟性能分析和检测工具有充分的了解才能在遇到问题的时候有点思路。
  展望方面主要是ceph全新的存储方式bluestore,可以直接对裸盘进行读写，避免了filestore写journal时的写放大问题，bluestore完全不需要journal,可以把更多的ssd投入到cache tier中,目前稳定版是Luminous，bluestore作为默认存储引擎，还有一些高性能底层新技术，如SPDK、RDMA，3d point等等，这些将对ceph的性能有不可估量的提升。