# <center>Etcd学习笔记 </center>

[toc] 

# 一、Etcd的概述

## 1、Etcd的简介

etcd是基于Go语言的一个分布式键值对存储系统，由coreos开发，内部采用raft协议作为一致性算法，用于可靠、快速地保存关键数据，并提供访问。通过分布式锁、leader选举和写屏障(write barriers)，来实现可靠的分布式协作。etcd集群是为高可用、持久化数据存储和检索而准备。

"etcd"这个名字源于两个想法，即 unix “/etc” 文件夹和分布式系统"d"istibuted。 “/etc” 文件夹为单个系统存储配置数据的地方，而 etcd 存储大规模分布式系统的配置信息。因此，"d"istibuted 的 “/etc” ，是为 “etcd”。

etcd以一致和容错的方式存储元数据。分布式系统使用etcd作为一致性键值存储系统，用于配置管理、服务发现和协调分布式工作。使用etcd的通用分布式模式包括领导选举、分布式锁和监控机器活动。

虽然etcd也支持单点部署，但是在生产环境中推荐集群方式部署。由于etcd内部使用投票机制，一般etcd节点数会选择 3、5、7等奇数。etcd会保证所有的节点都会保存数据，并保证数据的一致性和正确性。

## 2、Etcd的架构

通常，一个用户的请求发送过来，会经由HTTP Server转发给Store进行具体的事务处理，如果涉及到节点数据的修改，则交给Raft模块进行状态的变更、日志的记录，然后再同步给别的etcd节点以确认数据提交，最后进行数据的提交，再次同步。

![](C:\Users\lib\Desktop\Study\Etcd-Study-Notes\Architecture.png)

## 3、Etcd的核心组件

### （1）HTTP Server

接受客户端发出的API请求以及其它etcd节点的同步与心跳信息请求。

### （2）Store

用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件理与执行等等，是etcd对用户提供的大多数API功能的具体实现。

### （3）Raft

强一致性算法的具体实现，是etcd的核心算法。

### （4）WAL

WAL，Write Ahead Log，预写式日志，是etcd的数据存储方式，etcd会在内存中储存所有数据状态以及节点的索引，此外，etcd还会通过WAL进行持久化存储。WAL中，所有的数据提交前会事先记录日志。

- Snapshot是为了防止数据过多而进行的状态快照。

- Entry表示存储的具体日志内容。

## 4、Etcd的常用术语

| 术语        | 描述                    | 说明                                    |
|:--------- | --------------------- | ------------------------------------- |
| Raft      | Raft算法，etcd实现一致性的核心   | etcd有etcd-raft模块                      |
| Fellower  | Raft中的从属节点            | 竞争Leader失败的节点                         |
| Leader    | Raft中的领导协调节点，用于处理数据提交 | Leader节点协调整个集群                        |
| Candidate | 候选节点                  | 当Follower接收Leader节点的消息超时会转变为candidate |
| Node      | Raft状态机的实例            | Raft中设计多个节点                           |
| Member    | etcd实例，管理对应的Node节点    | 可处理客户端请求                              |
| Peer      | 同一个集群中的另一个Member      | 其他成员                                  |
| Cluster   | etcd集群                | 拥有多个etcd member                       |
| Lease     | 租期                    | 关键设置的租期，过期删除                          |
| Watch     | 检测机制                  | 监控键值对的变化                              |
| Term      | 任期                    | 某个节点成为Leader，到下一次竞选的时间                |
| WAL       | 预写式日志                 | 用于持久化存储的日志格式                          |
| client    | 客户端                   | 向etcd发起请求的客户端                         |

## 5、Etcd的特点

### （1）简单

安装简单，且为用户提供了HTTP API，使用起来也很简单。

### （2）存储

数据分层存储在文件目录中，类似于我们日常使用的文件系统。

### （3）Watch机制

Watch指定的键、前缀目录的更改，并对更改时间进行通知。

### （4）安全通信

支持SSL证书验证。

### （5）高性能

etcd单实例可以支持 2K/s 读操作，官方也有提供基准测试脚本。

### （6）一致可靠

基于Raft共识算法，实现分布式系统内部数据存储、服务调用的一致性和高可用性。

### （7）Revision机制

每个Key带有一个Revision号，每进行一次事务便加一，因此它是全局唯一的，如初始值为0，进行一次Put操作，Key的 Revision变为1，同样的操作，再进行一次，Revision变为 2；换成Key1进行Put操作，Revision将变为3。这种机制有一个作用，即通过Revision的大小就可知道写操作的顺序，这对于实现公平锁，队列十分有益。

### （8）Lease机制

租约机制（TTL，Time To Live），Etcd可以为存储的Key-Value对设置租约，当租约到期，Key-Value将失效删除；同时也支持续约，通过客户端可以在租约到期之前续约，以避免Key-Value对过期失效；此外，还支持解约，一旦解约，与该租约绑定的Key-Value将失效删除。

## 6、Etcd的应用场景

etcd在稳定性、可靠性和可伸缩性上表现极佳，同时也为云原生应用系统提供了协调机制。etcd经常用于服务注册与发现的场景，此外还有键值对存储、消息发布与订阅、分布式锁等场景。

### （1）键值对的存储

etcd是一个用于键值存储的组件，存储是etcd最基本的功能，其他应用场景都建立在etcd的可靠存储上。比如Kubernetes将一些元数据存储在etcd中，将存储状态数据的复杂工作交给tecd，Kubernetes自身的功能和架构就能更加稳定。

### （2）服务的注册与发现

etcd基于Raft算法，能够有力地保证分布式场景中的一致性。各个服务启动时注册到etcd上，同时为这些服务配置键的TTL 时间。注册到etcd上面的各个服务实例通过心跳的方式定期续租，实现服务实例的状态监控。

服务发现（Service Discovery）要解决的是分布式系统中最常见的问题之一，即在同一个分布式集群中的进程或服务如何才能找到对方并建立连接。

服务发现的实现原理如下:

- 存在一个高可靠、高可用的中心配置节点：基于Ralf算法的etcd天然支持。

- 服务提供方会持续的向配置节点注册服务：用户可以在etcd中注册服务，并且对注册的服务配置租约，定时续约以达到维持服务的目的（一旦停止续约，对应的服务就会失效）。

- 服务的调用方会持续地读取中心配置节点的配置并修改本机配置，然后Reload服务：服务提供方在etcd指定的目录（前缀机制支持）下注册服务，服务调用方在对应的目录下查询服务。通过watch机制，服务调用方还可以监测服务的变化。

### （3）消息的发布与订阅

在分布式系统中，服务之间还可以通过消息通信，即消息的发布与订阅。通过构建etcd消息中间件，服务提供者发布对应主题的消息，消费者则订阅他们关心的主题，一旦对应的主题有消息发布，就会产生订阅事件，消息中间件就会通知该主题所有的订阅者。

- 在分布式系统中，组件间通信常用的方式是消息发布-订阅机制。具体而言，即配置一个配置共享中心，数据提供者在这个配置中心发布消息，而消息使用者则订阅它们关心的主题，一旦有关主题有消息发布，就会实时通知订阅者。通过这种方式可以实现分布式系统配置的集中式管理和实时动态更新。显然，通过Watch机制可以实现。

- 应用在启动时，主动从etcd获取一次配置信息，同时，在etcd节点上注册一个watcher并等待，以后每次配置有更新，etcd都会实时通知订阅者，以此达到获取最新配置信息的目的。

### （4）负载均衡

在分布式系统中，为了保证服务的高可用以及数据的一致性，通常都会把数据和服务部署多份，以此达到对等服务，即使其中的某一个服务失效了，也不影响使用。这样的实现虽然会导致一定程度上数据写入性能的下降，但是却能实现数据访问时的负载均衡。因为每个对等服务节点上都存有完整的数据，所以用户的访问流量就可以分流到不同的机器上。

- etcd本身分布式架构存储的信息访问支持负载均衡。etcd集群化以后，每个etcd的核心节点都可以处理用户的请求。所以，把数据量小但是访问频繁的消息数据直接存储到etcd中也是个不错的选择，如业务系统中常用的二级代码表。二级代码表的工作过程一般是这样，在表中存储代码，在etcd中存储代码所代表的具体含义，业务系统调用查表的过程，就需要查找表中代码的含义。所以如果把二级代码表中的小量数据存储到etcd中，不仅方便修改，也易于大量访问。

- 利用etcd维护一个负载均衡节点表。etcd可以监控一个集群中多个节点的状态，当有一个请求发过来后，可以轮询式地把请求转发给存活着的多个节点。类似KafkaMQ，通过Zookeeper来维护生产者和消费者的负载均衡。同样也可以用etcd来做Zookeeper的工作。

### （5）分布式通知与协调

分布式通知与协调，与消息发布和订阅有些相似。两者都使用了etcd中的Watcher机制，通过注册与异步通知机制，实现分布式环境下不同系统之间的通知与协调，从而对数据变更进行实时处理。实现方式通常为：不同系统都在etcd上对同一个目录进行注册，同时设置Watcher监控该目录的变化（如果对子目录的变化也有需要，可以设置成递归模式），当某个系统更新了etcd的目录，那么设置了Watcher的系统就会收到通知，并作出相应处理。

- 通过etcd进行低耦合的心跳检测。检测系统和被检测系统通过etcd上某个目录关联而非直接关联起来，这样可以大大减少系统的耦合性。

- 通过etcd完成系统调度。某系统有控制台和推送系统两部分组成，控制台的职责是控制推送系统进行相应的推送工作。管理人员在控制台做的一些操作，实际上只需要修改etcd上某些目录节点的状态，而etcd就会自动把这些变化通知给注册了Watcher的推送系统客户端，推送系统再做出相应的推送任务。

- 通过etcd完成工作汇报。大部分类似的任务分发系统，子任务启动后，到etcd来注册一个临时工作目录，并且定时将自己的进度进行汇报（将进度写入到这个临时目录），这样任务管理者就能够实时知道任务进度。

### （6）分布式锁

因为etcd使用Raft算法保持了数据的强一致性，某次操作存储到集群中的值必然是全局一致的，所以很容易实现分布式锁。锁服务有两种使用方式，一是保持独占，二是控制时序。

- 保持独占，即所有试图获取锁的用户最终只有一个可以得到。etcd为此提供了一套实现分布式锁原子操作CAS（CompareAndSwap）的API。通过设置prevExist值，可以保证在多个节点同时创建某个目录时，只有一个成功，而该用户即可认为是获得了锁。

- 控制时序，即所有试图获取锁的用户都会进入等待队列，获得锁的顺序是全局唯一的，同时决定了队列执行顺序。etcd为此也提供了一套API（自动创建有序键），对一个目录建值时指定为POST动作，这样etcd会自动在目录下生成一个当前最大的值为键，存储这个新的值（客户端编号）。同时还可以使用API按顺序列出所有当前目录下的键值。此时这些键的值就是客户端的时序，而这些键中存储的值可以是代表客户端的编号。 

## 7、Etcd的学习文档

etcd官网https://etcd.io/

# 二、Etcd集群安装部署

使用docker-compose来模拟etcd的三个节点，三个节点在同一个桥接网络中。

## 1、编写docker-compose.yaml

### （1）yaml文件

```yaml
version: '2'
services:
  etcd1:
    image: quay.io/coreos/etcd
    container_name: etcd-node1
    command: etcd -name etcd1 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - etcd-net

  etcd2:
    image: quay.io/coreos/etcd
    container_name: etcd-node2
    command: etcd -name etcd2 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - etcd-net

  etcd3:
    image: quay.io/coreos/etcd
    container_name: etcd-node3
    command: etcd -name etcd3 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - etcd-net

networks:
  etcd-net:
```

### （2）集群配置参数

| 参数                                  | 说明                                                                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| name                                | etcd集群中的节点名，这里可以随意，可区分且不重复就行                                                                                                    |
| advertise-client-urls               | 建议客户端使用什么地址访问etcd，该值用于etcd代理或etcd成员与etcd节点通信                                                                                    |
| listen-client-urls                  | 监听的用于和客户端通信的url，可以监听多个，格式：'http://ip:2379'，多个用逗号分隔                                                                              |
| listen-peer-urls                    | 服务端节点之间通讯的监听地址，可监听多个，集群内部将通过这些url进行数据交互(如选举，数据同步等)，格式：'http://ip:2380'                                                          |
| initial-cluster-token  etcd-cluster | 节点的token值，设置该值后集群将生成唯一id，并为每个节点也生成唯一id，当使用相同配置文件再启动一个集群时，只要该token值不一样，etcd集群就不会相互影响                                             |
| initial-advertise-peer-urls         | 建议服务端之间通讯使用的地址列表，节点间将以该值进行通信                                                                                                    |
| initial-cluster                     | 集群中所有的initial-advertise-peer-urls的合集，etcd启动的时候，通过这个配置找到其他ectd节点的地址列表，格式：'节点名字1=http://节点ip1:2380,节点名字2=http://节点ip2:2380,.....' |
| initial-cluster-state               | 初始化的时候，集群的状态 "new" 或者 "existing"两种状态，初始化状态使用new，建立之后改此值为existing，表示加入已经存在的集群                                                    |

## 2、运行docker-compose

### （1）运行docker-compose

运行docker-compose.ymal

```shell
[root@hao etcd]# docker-compose up -d
Creating network "etcd_etcd-net" with the default driver
Pulling etcd1 (quay.io/coreos/etcd:)...
latest: Pulling from coreos/etcd
ff3a5c916c92: Pull complete
96b0e24539ea: Pull complete
d1eca4d01894: Pull complete
ad732d7a61c2: Pull complete
8bc526247b5c: Pull complete
5f56944bb51c: Pull complete
Digest: sha256:5b6691b7225a3f77a5a919a81261bbfb31283804418e187f7116a0a9ef65d21d
Status: Downloaded newer image for quay.io/coreos/etcd:latest
Creating etcd-node2 ... done
Creating etcd-node3 ... done
Creating etcd-node1 ... done
```

### （2）查看运行的容器

```shell
#三个节点都在运行中
[root@hao etcd]# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                                      NAMES
239dc8686a3b   quay.io/coreos/etcd   "etcd -name etcd1 -a…"   27 seconds ago   Up 25 seconds   0.0.0.0:49156->2379/tcp, :::49156->2379/tcp, 0.0.0.0:49154->2380/tcp, :::49154->2380/tcp   etcd-node1
1655025d5a22   quay.io/coreos/etcd   "etcd -name etcd3 -a…"   27 seconds ago   Up 25 seconds   0.0.0.0:49158->2379/tcp, :::49158->2379/tcp, 0.0.0.0:49157->2380/tcp, :::49157->2380/tcp   etcd-node3
473b84539601   quay.io/coreos/etcd   "etcd -name etcd2 -a…"   27 seconds ago   Up 26 seconds   0.0.0.0:49155->2379/tcp, :::49155->2379/tcp, 0.0.0.0:49153->2380/tcp, :::49153->2380/tcp   etcd-node2
```

## 3、 查看etcd集群的节点

### （1）命令行工具查看

查看member list，结果是三个node的数据一样的。

```shell
#在etcd-node1中查看member
[root@hao etcd]# docker exec -it etcd-node1 etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#在etcd-node2中查看member
[root@hao etcd]# docker exec -it etcd-node2 etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#在etcd-node3中查看member
[root@hao etcd]# docker exec -it etcd-node3 etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true
```

### （2）API请求查看

查看members，三个节点显示的信息也是一样的。

```shell
#查看端口
[root@hao etcd]# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                                      NAMES
239dc8686a3b   quay.io/coreos/etcd   "etcd -name etcd1 -a…"   18 minutes ago   Up 18 minutes   0.0.0.0:49156->2379/tcp, :::49156->2379/tcp, 0.0.0.0:49154->2380/tcp, :::49154->2380/tcp   etcd-node1
1655025d5a22   quay.io/coreos/etcd   "etcd -name etcd3 -a…"   18 minutes ago   Up 18 minutes   0.0.0.0:49158->2379/tcp, :::49158->2379/tcp, 0.0.0.0:49157->2380/tcp, :::49157->2380/tcp   etcd-node3
473b84539601   quay.io/coreos/etcd   "etcd -name etcd2 -a…"   18 minutes ago   Up 18 minutes   0.0.0.0:49155->2379/tcp, :::49155->2379/tcp, 0.0.0.0:49153->2380/tcp, :::49153->2380/tcp   etcd-node2

#访问etcd-node1的API获取members信息
[root@hao etcd]# curl http://127.0.0.1:49156/v2/members
{"members":[{"id":"5b926f852fa1811","name":"etcd1","peerURLs":["http://etcd-node1:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9b3cd975d37c44ce","name":"etcd2","peerURLs":["http://etcd-node2:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9e13ad3ed0f8a26b","name":"etcd3","peerURLs":["http://etcd-node3:2380"],"clientURLs":["http://0.0.0.0:2379"]}]}

#访问etcd-node2的API获取members信息
[root@hao etcd]# curl http://127.0.0.1:49155/v2/members
{"members":[{"id":"5b926f852fa1811","name":"etcd1","peerURLs":["http://etcd-node1:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9b3cd975d37c44ce","name":"etcd2","peerURLs":["http://etcd-node2:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9e13ad3ed0f8a26b","name":"etcd3","peerURLs":["http://etcd-node3:2380"],"clientURLs":["http://0.0.0.0:2379"]}]}

#访问etcd-node3的API获取members信息
[root@hao etcd]# curl http://127.0.0.1:49158/v2/members
{"members":[{"id":"5b926f852fa1811","name":"etcd1","peerURLs":["http://etcd-node1:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9b3cd975d37c44ce","name":"etcd2","peerURLs":["http://etcd-node2:2380"],"clientURLs":["http://0.0.0.0:2379"]},{"id":"9e13ad3ed0f8a26b","name":"etcd3","peerURLs":["http://etcd-node3:2380"],"clientURLs":["http://0.0.0.0:2379"]}]}
```

## 4、集群节点信息

| Hostname   | IP(映射到宿主机) | 客户端交互端口    | peer通信端口   |
| ---------- | ---------- | ---------- | ---------- |
| etcd-node1 | 127.0.0.1  | 49156:2379 | 49154:2380 |
| etcd-node2 | 127.0.0.1  | 49155:2379 | 49153:2380 |
| etcd-node3 | 127.0.0.1  | 49158:2379 | 49157:2380 |

## 5、查看集群的健康状态

通过命令行工具查看集群的健康状态为healthy

```shell
#在etcd-node1中查看
[root@hao etcd]# docker exec -it etcd-node1 etcdctl cluster-health
member 5b926f852fa1811 is healthy: got healthy result from http://0.0.0.0:2379
member 9b3cd975d37c44ce is healthy: got healthy result from http://0.0.0.0:2379
member 9e13ad3ed0f8a26b is healthy: got healthy result from http://0.0.0.0:2379
cluster is healthy

#在etcd-node2中查看
[root@hao etcd]# docker exec -it etcd-node2 etcdctl cluster-health
member 5b926f852fa1811 is healthy: got healthy result from http://0.0.0.0:2379
member 9b3cd975d37c44ce is healthy: got healthy result from http://0.0.0.0:2379
member 9e13ad3ed0f8a26b is healthy: got healthy result from http://0.0.0.0:2379
cluster is healthy

#在etcd-node3中查看
[root@hao etcd]# docker exec -it etcd-node3 etcdctl cluster-health
member 5b926f852fa1811 is healthy: got healthy result from http://0.0.0.0:2379
member 9b3cd975d37c44ce is healthy: got healthy result from http://0.0.0.0:2379
member 9e13ad3ed0f8a26b is healthy: got healthy result from http://0.0.0.0:2379
cluster is healthy
```

# 三、Etcdctl的基本使用

## 1、Etcdctl的简介

Etcd二进制发行包中已经包含了etcdctl工具，etcdctl支持的命令大体上分为数据库操作和非数据库操作两类。etcdctl是一个命令行的客户端，它能提供一些简洁的命令，供用户直接跟etcd服务打交道，而无需基于 HTTP API 方式。方便对服务进行测试或者手动修改数据库内容。这些操作跟 HTTP API 基本上是对应的。etcdctl在两个不同的 etcd 版本下的行为方式也完全不同。

```shell
#使用v2版本
export ETCDCTL_API=2  
#使用v3版本
export ETCDCTL_API=3
```

## 2、非数据操作

### （1）查看帮助

```shell
/ # etcdctl -help
NAME:
   etcdctl - A simple command line client for etcd.

WARNING:
   Environment variable ETCDCTL_API is not set; defaults to etcdctl v2.
   Set environment variable ETCDCTL_API=3 to use v3 API or ETCDCTL_API=2 to use v2 API.

USAGE:
   etcdctl [global options] command [command options] [arguments...]

VERSION:
   3.3.8

COMMANDS:
     backup          backup an etcd directory
     cluster-health  check the health of the etcd cluster
     mk              make a new key with a given value
     mkdir           make a new directory
     rm              remove a key or a directory
     rmdir           removes the key if it is an empty directory or a key-value pair
     get             retrieve the value of a key
     ls              retrieve a directory
     set             set the value of a key
     setdir          create a new directory or update an existing directory TTL
     update          update an existing key with a given value
     updatedir       update an existing directory
     watch           watch a key for changes
     exec-watch      watch a key for changes and exec an executable
     member          member add, remove and list subcommands
     user            user add, grant and revoke subcommands
     role            role add, grant and revoke subcommands
     auth            overall auth controls
     help, h         Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug                          output cURL commands which can be used to reproduce the request
   --no-sync                        don't synchronize cluster information before sending request
   --output simple, -o simple       output response in the given format (simple, `extended` or `json`) (default: "simple")
   --discovery-srv value, -D value  domain name to query for SRV records describing cluster endpoints
   --insecure-discovery             accept insecure SRV records describing cluster endpoints
   --peers value, -C value          DEPRECATED - "--endpoints" should be used instead
   --endpoint value                 DEPRECATED - "--endpoints" should be used instead
   --endpoints value                a comma-delimited list of machine addresses in the cluster (default: "http://127.0.0.1:2379,http://127.0.0.1:4001")
   --cert-file value                identify HTTPS client using this SSL certificate file
   --key-file value                 identify HTTPS client using this SSL key file
   --ca-file value                  verify certificates of HTTPS-enabled servers using this CA bundle
   --username value, -u value       provide username[:password] and prompt if password is not supplied.
   --timeout value                  connection timeout per request (default: 2s)
   --total-timeout value            timeout for the command execution (except watch) (default: 5s)
   --help, -h                       show help
   --version, -v                    print the version
```

### （2）查看etcdctl版本

```shell
#当前的etcdctl的版本是3.3.8，API的版本是2
/ # etcdctl -v
etcdctl version: 3.3.8
API version: 2
```

### （3） 查看etcd的版本

```shell
/ # etcd --version
etcd Version: 3.3.8
Git SHA: 33245c6b5
Go Version: go1.9.7
Go OS/Arch: linux/amd64
```

### （4）集群成员操作

* 查看集群成员列表

```shell
/ # etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true
```

### （5）

## 3、数据库操作

作为键值对存储的数据库，etcdctl有相应的对数据库进行操作的命令。数据库操作围绕对键值和目录的 CRUD （即增删改查，符合 REST 风格的一套API操作）完整生命周期的管理。Etcd在键的组织上采用了层次化的空间结构(类似于文件系统中目录的概念)，用户指定的键可以为单独的名字，如：testkey，此时实际上放在根目录/下面，也可以为指定目录结构，如/cluster1/node2/testkey，则将创建相应的目录结构。

### （1）增加键值对

```shell
#etcdctl put 键 值
/ # etcdctl put /key1 'value1'
No help topic for 'put' 
#使用v3的api
/ # export ETCDCTL_API=3
#在/目录下存储
/ # etcdctl put /key1 'value1'
OK
#没有加/，默认在/目录下存储
/ # etcdctl put key2 'value2'
OK
#在多级目录下存储
/ # etcdctl put /key3/key3-3 'value3'
OK
```

### （2）查看键值对

```shell
#查看key1的键值对
/ # etcdctl get /key1
/key1
value1

#查看key2的键值对，键不匹配，无法查看到值
/ # etcdctl get key2
key2
value2
/ # etcdctl get /key2

#查看key3的键值对
/ # etcdctl get /key3
/ # etcdctl get /key3/key3-3
/key3/key3-3
value3
```

# 四、
