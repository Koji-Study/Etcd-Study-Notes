# <center>Etcd学习笔记</center>

​																																																														——by 王浩

[toc] 

# 一、Etcd的概述

## 1、Etcd的简介

Etcd是基于Go语言的一个分布式键值对存储系统，由coreos开发，内部采用raft协议作为一致性算法，用于可靠、快速地保存关键数据，并提供访问。通过分布式锁、leader选举和写屏障(write barriers)，来实现可靠的分布式协作。etcd集群是为高可用、持久化数据存储和检索而准备。

"etcd"这个名字源于两个想法，即 unix “/etc” 文件夹和分布式系统"d"istibuted。 “/etc” 文件夹为单个系统存储配置数据的地方，而 etcd 存储大规模分布式系统的配置信息。因此，"d"istibuted 的 “/etc” ，是为 “etcd”。

etcd以一致和容错的方式存储元数据。分布式系统使用etcd作为一致性键值存储系统，用于配置管理、服务发现和协调分布式工作。使用etcd的通用分布式模式包括领导选举、分布式锁和监控机器活动。

虽然etcd也支持单点部署，但是在生产环境中推荐集群方式部署。由于etcd内部使用投票机制，一般etcd节点数会选择 3、5、7等奇数。etcd会保证所有的节点都会保存数据，并保证数据的一致性和正确性。

## 2、Etcd的架构

通常，一个用户的请求发送过来，会经由HTTP Server转发给Store进行具体的事务处理，如果涉及到节点数据的修改，则交给Raft模块进行状态的变更、日志的记录，然后再同步给别的etcd节点以确认数据提交，最后进行数据的提交，再次同步。

![etcd架构](C:\Users\lib\Desktop\Study\Etcd-Study-Notes\Architecture.png "etcd架构")

## 3、Etcd的核心组件

### （1）HTTP Server

接受客户端发出的API请求以及其它etcd节点的同步与心跳信息请求，grapc方法。

### （2）Store

用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件理与执行等等，是etcd对用户提供的大多数API功能的具体实现，boltdb存储。

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

使用docker-compose来模拟etcd的三个节点，三个节点在同一个桥接网络中。虽然上限没有限制，但是一个etcd cluster最好不要超过7个节点。一方面，节点数量增加的确可以增加容错率，例如7个节点可以容忍3个down，9个则可以容忍4个down。但是另一方面，节点数量增加，同样也会导致quorum增大，例如7个节点的quorum是4，9个节点的quorum则是5；因为etcd是强一致性K/V DB，每一个写请求必须至少quorum个节点都写成功才能返回成功，所以节点数量增加会降低写请求的性能。

## 1、编写docker-compose.yaml

### （1）yaml文件

```yaml
version: '2'
services:
  etcd1:
    image: quay.io/coreos/etcd
    container_name: etcd-node1
    command: etcd -name etcd1 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data1:/etcd-data
    networks:
      - etcd-net

  etcd2:
    image: quay.io/coreos/etcd
    container_name: etcd-node2
    command: etcd -name etcd2 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data2:/etcd-data
    networks:
      - etcd-net

  etcd3:
    image: quay.io/coreos/etcd
    container_name: etcd-node3
    command: etcd -name etcd3 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd-node1:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data3:/etcd-data
    networks:
      - etcd-net

networks:
  etcd-net:
    driver: bridge
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



# 三、Etcdctl的基本使用v3

## 1、Etcdctl的简介

Etcd二进制发行包中已经包含了etcdctl工具，etcdctl支持的命令大体上分为数据库操作和非数据库操作两类。etcdctl是一个命令行的客户端，它能提供一些简洁的命令，供用户直接跟etcd服务打交道，而无需基于 HTTP API 方式。方便对服务进行测试或者手动修改数据库内容。这些操作跟 HTTP API 基本上是对应的。etcdctl在两个不同的 etcd 版本下的行为方式也完全不同。

```shell
#使用v2版本
export ETCDCTL_API=2  
#使用v3版本
export ETCDCTL_API=3
```

## 2、集群工具操作命令

### （1）查看帮助

```shell
/ # etcdctl --help
NAME:
        etcdctl - A simple command line client for etcd3.

USAGE:
        etcdctl

VERSION:
        3.3.8

API VERSION:
        3.3

COMMANDS:
        get                     Gets the key or a range of keys
        put                     Puts the given key into the store
        del                     Removes the specified key or range of keys [key, range_end)
        txn                     Txn processes all the requests in one transaction
        compaction              Compacts the event history in etcd
        alarm disarm            Disarms all alarms
        alarm list              Lists all alarms
        defrag                  Defragments the storage of the etcd members with given endpoints
        endpoint health         Checks the healthiness of endpoints specified in `--endpoints` flag
        endpoint status         Prints out the status of endpoints specified in `--endpoints` flag
        endpoint hashkv         Prints the KV history hash for each endpoint in --endpoints
        move-leader             Transfers leadership to another etcd cluster member.
        watch                   Watches events stream on keys or prefixes
        version                 Prints the version of etcdctl
        lease grant             Creates leases
        lease revoke            Revokes leases
        lease timetolive        Get lease information
        lease list              List all active leases
        lease keep-alive        Keeps leases alive (renew)
        member add              Adds a member into the cluster
        member remove           Removes a member from the cluster
        member update           Updates a member in the cluster
        member list             Lists all members in the cluster
        snapshot save           Stores an etcd node backend snapshot to a given file
        snapshot restore        Restores an etcd member snapshot to an etcd directory
        snapshot status         Gets backend snapshot status of a given file
        make-mirror             Makes a mirror at the destination etcd cluster
        migrate                 Migrates keys in a v2 store to a mvcc store
        lock                    Acquires a named lock
        elect                   Observes and participates in leader election
        auth enable             Enables authentication
        auth disable            Disables authentication
        user add                Adds a new user
        user delete             Deletes a user
        user get                Gets detailed information of a user
        user list               Lists all users
        user passwd             Changes password of user
        user grant-role         Grants a role to a user
        user revoke-role        Revokes a role from a user
        role add                Adds a new role
        role delete             Deletes a role
        role get                Gets detailed information of a role
        role list               Lists all roles
        role grant-permission   Grants a key to a role
        role revoke-permission  Revokes a key from a role
        check perf              Check the performance of the etcd cluster
        help                    Help about any command

OPTIONS:
      --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""                                 identify secure client using this TLS certificate file
      --command-timeout=5s                      timeout for short running command (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connections
  -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
      --hex[=false]                             print byte strings as hex encoded strings
      --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verification
      --insecure-transport[=true]               disable transport security for client connections
      --keepalive-time=2s                       keepalive time for client connections
      --keepalive-timeout=6s                    keepalive timeout for client connections
      --key=""                                  identify secure client using this TLS key file
      --user=""                                 username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)
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

### （4） 查看集群状态

```shell
#endpoint status
/ # etcdctl endpoint status --cluster
http://0.0.0.0:2379, 9e13ad3ed0f8a26b, 3.3.8, 20 kB, false, 3, 25
http://0.0.0.0:2379, 9e13ad3ed0f8a26b, 3.3.8, 20 kB, false, 3, 25
http://0.0.0.0:2379, 9e13ad3ed0f8a26b, 3.3.8, 20 kB, false, 3, 25
```

### （5）查看集群成员列表

```shell
/ # etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#以表格的形式输出
/ # etcdctl member list --write-out=table
+------------------+---------+-------+------------------------+---------------------+
|        ID        | STATUS  | NAME  |       PEER ADDRS       |    CLIENT ADDRS     |
+------------------+---------+-------+------------------------+---------------------+
|  5b926f852fa1811 | started | etcd1 | http://etcd-node1:2380 | http://0.0.0.0:2379 |
| 9b3cd975d37c44ce | started | etcd2 | http://etcd-node2:2380 | http://0.0.0.0:2379 |
| 9e13ad3ed0f8a26b | started | etcd3 | http://etcd-node3:2380 | http://0.0.0.0:2379 |
+------------------+---------+-------+------------------------+---------------------+
```

### （6）更新集群成员信息

当集群的成员的peer-urls发生变化，但是集群未更新的时候，可以使用update命令更新集群节点的peer-urls

```shell
/ # etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#etcdctl member update ID peer-urls
/ # etcdctl member update 9e13ad3ed0f8a26b http://etcd3:2380
Updated member with ID 9e13ad3ed0f8a26b in cluster

/ # etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true
```

### （7）移除集群成员

当cluster中某个节点down了，一般需要把这个节点从cluster中移除，并增加一个新的节点到cluster中去。要先删除老节点，然后再增加新节点。

假设cluster有三个节点，其中一个节点突然发生故障。这时cluster中还有两个节点健康，依然满足quorum，所以cluster还可以正常提供服务。这时如果先增加一个新节点到cluster中，因为那个down的节点还没有从cluster中移除，所以cluster的节点数量就变成了4。如果一切顺利，新节点正常启动，倒也还好。但是如果发生了意外（比如配置错误），导致新增加的节点启动失败，这时cluster的quorum已经从2变成了3，可是cluster中依然只有两个节点能工作，不符合quorum要求，从而导致cluster无法对外提供服务，而且永远无法自动恢复。这个时候，当再想从cluster中删除任何节点，都不会成功了，因为删除节点也需要向cluster中发送一个写请求，这时cluster已经因为quorum不达标而无法提供服务了，所以删除节点的操作不会成功。

```shell
#假设集群的etcd-node1节点出现故障，现在需要将其移除集群
/ # etcdctl member list
5b926f852fa1811: name=etcd1 peerURLs=http://etcd-node1:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#将etcd-node1移除集群
#需要使用ID
/ # etcdctl member remove etcd1
Found a member named etcd1; if this is correct, please use its ID, eg:
        etcdctl member remove 5b926f852fa1811
For more details, read the documentation at https://github.com/coreos/etcd/blob/master/Documentation/runtime-configuration.md#remove-a-member

Couldn't find a member in the cluster with an ID of etcd1.

#使用ID将etcd1移出集群
/ # etcdctl member remove 5b926f852fa1811
Removed member 5b926f852fa1811 from cluster

/ # etcdctl member list
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#节点移除集群后，因为是同一个docker-compose起的，此时容器也停止
[root@wanghao02 alertsnitch]# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS          PORTS                                                                                      NAMES
736b604b1a89   quay.io/coreos/etcd    "etcd -name etcd2 -a…"   3 months ago   Up 15 minutes   0.0.0.0:49162->2379/tcp, :::49162->2379/tcp, 0.0.0.0:49161->2380/tcp, :::49161->2380/tcp   etcd-node2
63820a33ea71   quay.io/coreos/etcd    "etcd -name etcd3 -a…"   3 months ago   Up 15 minutes   0.0.0.0:49160->2379/tcp, :::49160->2379/tcp, 0.0.0.0:49159->2380/tcp, :::49159->2380/tcp   etcd-node3
```

### （8）增加集群成员

增加新的成员时，先在集群中加入新的节点，然后，根据集群显示的设置，给新的节点添加配置并启动节点容器。

* 在集群中添加新的节点到集群

```shell
#查看当前集群的节点信息
/ # etcdctl member list
9b3cd975d37c44ce: name=etcd2 peerURLs=http://etcd-node2:2380 clientURLs=http://0.0.0.0:2379 isLeader=false
9e13ad3ed0f8a26b: name=etcd3 peerURLs=http://etcd-node3:2380 clientURLs=http://0.0.0.0:2379 isLeader=true

#加入的集群的节点状态必须是existing,ETCD_INITIAL_CLUSTER需要包含所有的节点的urls
/ # etcdctl member add etcd4 --peer-urls=http://etcd-node4:2380
Member 8416fda8bc19f4fd added to cluster b410a000b04e40c7

ETCD_NAME="etcd4"
ETCD_INITIAL_CLUSTER="etcd4=http://etcd-node4:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd-node4:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

* 启动新的节点

```shell
#新建docker-compose.yaml文件
[root@wanghao02 data]# mkdir etcd4
[root@wanghao02 data]# cd etcd4
[root@wanghao02 etcd4]# touch docker-compose.yaml

#如果是docker-compose创建容器的话，因为用到了hostname, 所以要加入集群的节点需要和集群现有的其他节点在同一个网络里面
#集群的network是etcd_etcd-net
[root@wanghao02 etcd4]# docker network ls
NETWORK ID     NAME                      DRIVER    SCOPE
0d282e87367e   bridge                    bridge    local
6b0d3376e9c6   etcd_etcd-net             bridge    local
0396c628d491   etcdkeeper_default        bridge    local
14298dbc9108   host                      host      local
303a50f436e7   none                      null      local

#设置新的节点的网络为etcd_etcd-net
[root@wanghao02 etcd4]# vi docker-compose.yaml
version: '2'
services:
  etcd4:
    image: quay.io/coreos/etcd
    container_name: etcd-node4
    command: etcd -name etcd4 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd4=http://etcd-node4:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380" -initial-cluster-state existing
    ports:
      - 2379
      - 2380
    networks:
      - etcd_etcd-net

networks:
 etcd_etcd-net:
  external: true
        
#启动新的节点etcd4
[root@wanghao02 etcd4]# docker-compose up -d
Recreating etcd-node4 ... done

#容器已经成功启动
[root@wanghao02 etcd4]# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS          PORTS                                                                                      NAMES
2a526145cd49   quay.io/coreos/etcd    "etcd -name etcd4 -a…"   2 minutes ago   Up 2 minutes    0.0.0.0:49174->2379/tcp, :::49174->2379/tcp, 0.0.0.0:49173->2380/tcp, :::49173->2380/tcp   etcd-node4
7f7fab46dcd6   evildecay/etcdkeeper   "/bin/sh -c './etcdk…"   3 days ago      Up 3 days       0.0.0.0:9999->8080/tcp, :::9999->8080/tcp                                                  etcdkeeper
736b604b1a89   quay.io/coreos/etcd    "etcd -name etcd2 -a…"   3 months ago    Up 54 minutes   0.0.0.0:49162->2379/tcp, :::49162->2379/tcp, 0.0.0.0:49161->2380/tcp, :::49161->2380/tcp   etcd-node2
63820a33ea71   quay.io/coreos/etcd    "etcd -name etcd3 -a…"   3 months ago    Up 54 minutes   0.0.0.0:49160->2379/tcp, :::49160->2379/tcp, 0.0.0.0:49159->2380/tcp, :::49159->2380/tcp   etcd-node3
```

* 验证新的节点

```shell
#此时节点并没有加入到集群
[root@wanghao02 etcd4]# docker exec -it etcd-node2 /bin/sh
/ # export ETCDCTL_API=3

#此时，节点已经加到集群中了
/ # etcdctl member list
8416fda8bc19f4fd, started, etcd4, http://etcd-node4:2380, http://0.0.0.0:2379
9b3cd975d37c44ce, started, etcd2, http://etcd-node2:2380, http://0.0.0.0:2379
9e13ad3ed0f8a26b, started, etcd3, http://etcd-node3:2380, http://0.0.0.0:2379

#查看集群的状态
/ # etcdctl --write-out="table" --endpoints="http://etcd-node2:2380,http://etcd-node3:2380,http://etcd-node4:2380" endpoint status
+------------------------+------------------+---------+---------+-----------+-----------+------------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+------------------------+------------------+---------+---------+-----------+-----------+------------+
| http://etcd-node2:2380 | 9b3cd975d37c44ce |   3.3.8 |   25 kB |     false |        33 |        109 |
| http://etcd-node3:2380 | 9e13ad3ed0f8a26b |   3.3.8 |   25 kB |      true |        33 |        109 |
| http://etcd-node4:2380 | 8416fda8bc19f4fd |   3.3.8 |   25 kB |     false |        33 |        109 |
+------------------------+------------------+---------+---------+-----------+-----------+------------+

#添加新的数据
/ # etcdctl put /test "test"
OK

/ # etcdctl get /test
/test
test
/ # exit

#到etcd4验证查看数据，数据也同步过来了
[root@wanghao02 etcd4]# docker exec -it etcd-node4 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /test
/test
test
```

* 更新集群的配置文件

此时是在不重启的条件下加入集群，如果不修改配置文件，下次重启docker-compose时就会将新加入的节点排除了，所以，需要在节点的配置上设置

```yaml
 #-initial-cluster设置为三个节点urls
 -initial-cluster "etcd4=http://etcd-node4:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380"
 
 #如果想重启时，继续使用etcd1的话，需要在每个节点设置

 -initial-cluster "etcd4=http://etcd-node4:2380,etcd2=http://etcd-node2:2380,etcd3=http://etcd-node3:2380，http://etcd-node1:2380"
 
 #每个节点需要设置状态为new
 -initial-cluster-state new
```

## 3、数据库数据操作命令

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

#没有加/
/ # etcdctl put key2 'value2'
OK

#在多级目录下存储
/ # etcdctl put /key3/key3-3 'value3'
OK
```

### （2）查看键值对

* 普通查询

```shell
#查看key1的键值对
/ # etcdctl get /key1
/key1
value1

#查看key2的键值对，键不匹配，无法查看到值
/ # etcdctl get /key2
/ # etcdctl get key2
key2
value2

#查看key3的键值对
/ # etcdctl get /key3
/ # etcdctl get /key3/key3-3
/key3/key3-3
value3
```

* 查询十六进制的值

```shell
/ # etcdctl get /key1
/key1
value1

#查询后键值对变为16进制展示
/ # etcdctl get /key1 --hex
\x2f\x6b\x65\x79\x31
\x76\x61\x6c\x75\x65\x31
```

* 根据前缀查询 

```shell
#查询所有键值对（格式错误就会显示所有的键值对）
/ # etcdctl get /  --prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3

/ # etcdctl get /  prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3
foo
bar
```

* 范围查询

范围查询的原则是半开区间，[start-key，ending-key)，即获取了大于等于start-key，且小于ending-key的键值对。

```shell
#范围查找，因为只有/key3/key3-1和/key3/key3-3，所以范围查找的结果是两个
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3

/ # etcdctl get /key3/key3-1 /key3/key3-3
/key3/key3-1
value1
/key3/key3-2
value2
```

* 只查看key

```shell
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3

#查看/开头的key
/ # etcdctl get / --prefix --keys-only
/key3/key3-1

/key3/key3-2

/key3/key3-3

#查看全部的key（格式输错就会显示所有的键值对）
/ # etcdctl get / prefix --keys-only
/key3/key3-1

/key3/key3-2

/key3/key3-3

foo
```

* 只查看value

```shell
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3

#查看/开头的key对应的value
/ # etcdctl get / --prefix --print-value-only
value1
value2
value3
```

* 限制显示条数

```shell
/ # etcdctl get /key3/key3-1 /key3/key3-3
/key3/key3-1
value1
/key3/key3-2
value2

#限制只显示一条，则显示第一条
/ # etcdctl get /key3/key3-1 /key3/key3-3 --limit=1
/key3/key3-1
value1

#限制只显示两条，则显示两条
/ # etcdctl get /key3/key3-1 /key3/key3-3 --limit=2
/key3/key3-1
value1
/key3/key3-2
value2

#限制为0或者限制数超过结果条数的，则显示正常的结果
#限制条数为0时，显示全部结果
/ # etcdctl get /key3/key3-1 /key3/key3-3 --limit=0
/key3/key3-1
value1
/key3/key3-2
value2

#限制条数为3时，显示全部结果
/ # etcdctl get /key3/key3-1 /key3/key3-3 --limit=3
/key3/key3-1
value1
/key3/key3-2
```

### （3）删除键值对

* 删除单个键值对

```shell
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-2
value2
/key3/key3-3
value3

#删除/key3/key3-2的键值对，执行成功返回执行成功的条数
/ # etcdctl del /key3/key3-2
1
#再次删除/key3/key3-2的键值对，因为已经删除过一次了，所以第二次删除失败，返回0
/ # etcdctl del /key3/key3-2
0

#查看删除后的结果
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-3
value3
```

* 删除多个键值对

删除半开区间的键值对，因为只有两个键值对，删除半开区间[/key3/key3-1 /key3/key3-3),s所以执行成功的键值对数为1

```shell
/ # etcdctl get / --prefix
/key3/key3-1
value1
/key3/key3-3
value3

/ # etcdctl del /key3/key3-1 /key3/key3-3
1
```

删除所有以/开头的键值对

```shell
/ # etcdctl get / --prefix
/1
1
/2
2
/3
3
/ # etcdctl del / --prefix
3
#查看删除后的以/开头的键值对,已经删除，无法查看到结果
/ # etcdctl get / --prefix
```

#删除从key2开始包括key2的所有键值对

```shell
/ # etcdctl get / --prefix
/key1
1
/key2
2
/key3
3
/key4
4
/key5-key
5
/key66
66

/ # etcdctl del /key2 --from-key
5
#删除后只剩下key1
/ # etcdctl get / --prefix
/key1
1
```

删除后返回键值对的详细信息

```shell
/ # etcdctl del /key1 --prev-kv
1      #删除键值对的个数
/key1  #被删除的键
1      #被删除的键对应的值
```

## 4、数据库其他操作命令

### （1）数据库数据的备份

etcd数据的备份和恢复可以使用快照的方式，因为etcd集群所有节点的信息是会同步的，所以只需要在集群的其中一个节点上进行备份操作就行。etcd的snapshot的创建就是将默认的存储数据的文件夹下的member/snap/db的文件做一个备份。

使用etcdctl快照还原的集群还原将创建新的etcd数据目录，所有成员都应该使用相同的快照进行恢复。恢复将覆盖某些快照元数据（成员ID和群集ID），该成员失去了以前的身份，此元数据覆盖可防止新成员无意中加入现有群集。因此，为了从快照启动集群，还原必须启动新的逻辑集群。

* 查看当前集群

```shell
[root@wanghao02 ~]# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED             STATUS             PORTS                                                                                      NAMES
1f1068c508a3   quay.io/coreos/etcd    "etcd -name etcd3 -a…"   About an hour ago   Up About an hour   0.0.0.0:49178->2379/tcp, :::49178->2379/tcp, 0.0.0.0:49177->2380/tcp, :::49177->2380/tcp   etcd-node3
5fb5d3286aec   quay.io/coreos/etcd    "etcd -name etcd1 -a…"   About an hour ago   Up About an hour   0.0.0.0:49180->2379/tcp, :::49180->2379/tcp, 0.0.0.0:49179->2380/tcp, :::49179->2380/tcp   etcd-node1
af2e07033f64   quay.io/coreos/etcd    "etcd -name etcd2 -a…"   About an hour ago   Up About an hour   0.0.0.0:49176->2379/tcp, :::49176->2379/tcp, 0.0.0.0:49175->2380/tcp, :::49175->2380/tcp   etcd-node2
7f7fab46dcd6   evildecay/etcdkeeper   "/bin/sh -c './etcdk…"   3 days ago          Up 3 days          0.0.0.0:9999->8080/tcp, :::9999->8080/tcp                                                  etcdkeeper
```

* 随便进入一个节点

```shell
[root@wanghao02 ~]# docker exec -it etcd-node1 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl member list
5b926f852fa1811, started, etcd1, http://etcd-node1:2380, http://0.0.0.0:2379
9b3cd975d37c44ce, started, etcd2, http://etcd-node2:2380, http://0.0.0.0:2379
9e13ad3ed0f8a26b, started, etcd3, http://etcd-node3:2380, http://0.0.0.0:2379
```

* 查看备份前etcd的数据

```shell
/ # etcdctl get /key1
/key1
value1
```

* 查看备份前的目录

```shell
/ # ls
bin         dev         etc         etcd1.etcd  home        lib         media       mnt         proc        root        run         sbin        srv         sys         tmp         usr         var

/ # cd etcd1.etcd/
/etcd1.etcd # ls
member
/etcd1.etcd # cd member
/etcd1.etcd/member # ls
snap  wal
/etcd1.etcd/member # cd snap
/etcd1.etcd/member/snap # ls
db
```

* 用快照的方式进行备份

```shell
/ # etcdctl snapshot save /etcd-snapshot-`date +%Y-%m-%d`.db
Snapshot saved at /etcd-snapshot-2022-11-11.db
```

* 查看备份后的文件目录

```shell
#在指定的目录可以看到生成的指定名字的备份文件
/ # ls
bin                          etcd-snapshot-2022-11-11.db  lib                          proc                         sbin                         tmp
dev                          etcd1.etcd                   media                        root                         srv                          usr
etc
```

* 查看snapshot的状态

```shell
/ # etcdctl snapshot status etcd-snapshot-2022-11-11.db  --write-out=table
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 3c08ee8d |        2 |          7 |      20 kB |
+----------+----------+------------+------------+
```

* 将备份的数据保存到本地

```shell
[root@wanghao02 etcd]# docker cp etcd-node1:/etcd-snapshot-2022-11-11.db /mnt/data/etcd/
[root@wanghao02 ~]# cd /mnt/data/etcd
[root@wanghao02 etcd]# ls
docker-compose.yml  etcd_data1  etcd_data2  etcd_data3  etcd-snapshot-2022-11-11.db
```

* 修改集群的数据，模拟集群数据发生错误。

```shell
[root@wanghao02 etcd]# docker exec -it etcd-node2 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl put /key1 value2
OK
/ # etcdctl get /key1
/key1
value2

#在其他节点查看更新后的数据，已经更新了/key1的值，并且其他节点上没有备份的文件
[root@wanghao02 etcd]# docker exec -it etcd-node3 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /key1
/key1
value2

/ # ls
bin         dev         etc         etcd3.etcd  home        lib         media       mnt         proc        root        run         sbin        srv         sys         tmp         usr         var
```

### （2）数据库数据的恢复

基于3.3版本的etcd，恢复etcd的备份数据时，需要新建一个etcd集群，也可以就在当前的etcd集群上恢复，无论哪种方法，都需要创建新的逻辑集群，然后将备份的数据恢复到每个节点上。

* 拷贝备份的文件到当前文件夹下，方便挂载

```shell
[root@wanghao02 etcdbak]# ls
docker-compose.yml  etcd-snapshot-2022-11-11.db
```

* 新建一个集群

新建的集群yaml，恢复数据时，文件夹不能存在，否则会报错：Error: data-dir "/etcd-data" exists，所以挂载出来的目录为一级目录，恢复的目录为挂载目录下的二级目录，这样就不会产生冲突。

要注意的是新建的集群名，集群token不能和之前的名字相同，快照文件必须要相同的。先用etcdctl恢复备份，然后再启动etcd。

```yaml
version: '2'
services:
  etcd4:
    image: quay.io/coreos/etcd
    container_name: etcd-node4
    command: /bin/sh -c "export ETCDCTL_API=3 && etcdctl snapshot restore etcd-snapshot-2022-11-11.db --name etcd4 --initial-advertise-peer-urls "http://etcd-node4:2380" --initial-cluster-token etcd-cluster-new  --initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" --data-dir="/etcd-data/etcd-data-new"
            && etcd -name etcd4 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data/etcd-data-new" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster-new -initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" -initial-cluster-state new"
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data4:/etcd-data
      - ./etcd-snapshot-2022-11-11.db:/etcd-snapshot-2022-11-11.db
    networks:
      - etcd-net

  etcd5:
    image: quay.io/coreos/etcd
    container_name: etcd-node5
    command: /bin/sh -c "export ETCDCTL_API=3 && etcdctl snapshot restore etcd-snapshot-2022-11-11.db --name etcd5 --initial-advertise-peer-urls "http://etcd-node5:2380" --initial-cluster-token etcd-cluster-new  --initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" --data-dir="/etcd-data/etcd-data-new"
            && etcd -name etcd5 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data/etcd-data-new" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster-new -initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" -initial-cluster-state new"
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data5:/etcd-data
      - ./etcd-snapshot-2022-11-11.db:/etcd-snapshot-2022-11-11.db
    networks:
      - etcd-net

  etcd6:
    image: quay.io/coreos/etcd
    container_name: etcd-node6
    command: /bin/sh -c "export ETCDCTL_API=3 && etcdctl snapshot restore etcd-snapshot-2022-11-11.db --name etcd6 --initial-advertise-peer-urls "http://etcd-node6:2380" --initial-cluster-token etcd-cluster-new  --initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" --data-dir="/etcd-data/etcd-data-new"
            && etcd -name etcd6 -advertise-client-urls http://0.0.0.0:2379 -data-dir="/etcd-data/etcd-data-new" -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster-new -initial-cluster "etcd4=http://etcd-node4:2380,etcd5=http://etcd-node5:2380,etcd6=http://etcd-node6:2380" -initial-cluster-state new"
    ports:
      - 2379
      - 2380
    volumes:
      - ./etcd_data6:/etcd-data
      - ./etcd-snapshot-2022-11-11.db:/etcd-snapshot-2022-11-11.db
    networks:
      - etcd-net

networks:
  etcd-net:
    driver: bridge
```

* 新建集群并查看

```shell
[root@wanghao02 etcdbak]# docker-compose up -d
Creating network "etcdbak_etcd-net" with driver "bridge"
Creating etcd-node4 ... done
Creating etcd-node6 ... done
Creating etcd-node5 ... done

#查看容器已经有两个集群存在
[root@wanghao02 etcdbak]# docker ps
[root@wanghao02 etcdbak]# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                                                                                      NAMES
ba34b9e895b1   quay.io/coreos/etcd    "/bin/sh -c 'export …"   5 minutes ago   Up 5 minutes   0.0.0.0:49340->2379/tcp, :::49340->2379/tcp, 0.0.0.0:49339->2380/tcp, :::49339->2380/tcp   etcd-node6
eb808fc127ea   quay.io/coreos/etcd    "/bin/sh -c 'export …"   5 minutes ago   Up 5 minutes   0.0.0.0:49342->2379/tcp, :::49342->2379/tcp, 0.0.0.0:49341->2380/tcp, :::49341->2380/tcp   etcd-node5
3805155c3cc9   quay.io/coreos/etcd    "bin/sh -c 'export E…"   5 minutes ago   Up 5 minutes   0.0.0.0:49338->2379/tcp, :::49338->2379/tcp, 0.0.0.0:49337->2380/tcp, :::49337->2380/tcp   etcd-node4
eabcd5b53edd   quay.io/coreos/etcd    "etcd -name etcd3 -a…"   3 days ago      Up 3 days      0.0.0.0:49210->2379/tcp, :::49210->2379/tcp, 0.0.0.0:49208->2380/tcp, :::49208->2380/tcp   etcd-node3
c5d9cf1f3b72   quay.io/coreos/etcd    "etcd -name etcd1 -a…"   3 days ago      Up 3 days      0.0.0.0:49206->2379/tcp, :::49206->2379/tcp, 0.0.0.0:49205->2380/tcp, :::49205->2380/tcp   etcd-node1
2b88914fa893   quay.io/coreos/etcd    "etcd -name etcd2 -a…"   3 days ago      Up 3 days      0.0.0.0:49209->2379/tcp, :::49209->2379/tcp, 0.0.0.0:49207->2380/tcp, :::49207->2380/tcp   etcd-node2
7f7fab46dcd6   evildecay/etcdkeeper   "/bin/sh -c './etcdk…"   7 days ago      Up 7 days      0.0.0.0:9999->8080/tcp, :::9999->8080/tcp                                                  etcdkeeper

#查看挂载出来的目录，目录也能挂载出来了
[root@wanghao02 etcdbak]# ls
docker-compose.yml  docker-compose.yml.bak  etcd_data4  etcd_data5  etcd_data6  etcd-snapshot-2022-11-11.db

[root@wanghao02 etcdbak]# tree
.
├── docker-compose.yml
├── docker-compose.yml.bak
├── etcd_data4
│   └── etcd-data-new
│       └── member
│           ├── snap
│           │   ├── 0000000000000001-0000000000000003.snap
│           │   └── db
│           └── wal
│               ├── 0000000000000000-0000000000000000.wal
│               └── 0.tmp
├── etcd_data5
│   └── etcd-data-new
│       └── member
│           ├── snap
│           │   ├── 0000000000000001-0000000000000003.snap
│           │   └── db
│           └── wal
│               ├── 0000000000000000-0000000000000000.wal
│               └── 0.tmp
├── etcd_data6
│   └── etcd-data-new
│       └── member
│           ├── snap
│           │   ├── 0000000000000001-0000000000000003.snap
│           │   └── db
│           └── wal
│               ├── 0000000000000000-0000000000000000.wal
│               └── 0.tmp
└── etcd-snapshot-2022-11-11.db
```

* 进入节点查看

各个节点数据一致，并且/key1是备份前的值

```shell
#进入节点6查看
[root@wanghao02 etcdbak]# docker exec -it etcd-node6 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /key1
/key1
value1
/ # exit

#进入节点5查看
[root@wanghao02 etcdbak]# docker exec -it etcd-node5 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /key1
/key1
value1
/ # exit

#进入节点4查看
[root@wanghao02 etcdbak]# docker exec -it etcd-node4 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /key1
/key1
value1
/ # exit
```



# 四 、Etcd的MVCC

## 1、MVCC的简介

### （1）Etcd v3使用MVCC的背景

etcd v2版本存在丢弃历史版本数据的问题，仅保留最新版本的数据。但是这样做引起了一系列问题，比如watch机制依赖历史版本数据实现相应功能，因此etcd v2又采取了在内存中建立滑动窗口来维护部分历史变更数据的做法，然而在大型的业务场景下还是不足以支撑大量历史变更数据的维护。到了etcd v3版本，该功能得到了更新，etcd v3支持MVCC，可以保存一个键值对的多个历史版本。

### （2）MVCC简介

MVCC（Multi-Version Concurrency Control），即多版本并发控制，它是一种并发控制的方法，可以实现对数据库的并发访问。MVCC模块是etcd的核心模块。MVCC作为底层模块，为上层提供统一的调用方法。

数据库并发场景有三种，分别为读-读、读-写和写-写。第一种读-读没有问题，不需要并发控制；读-写和写-写都存在线程安全问题。读-写可能遇到脏读、幻读、不可重复读的问题；写-写可能会存在更新丢失问题。

并发控制机制用作对并发操作进行正确调度，保证事务的隔离性、数据库的一致性。它的主要技术包括悲观锁和乐观锁等：

* 悲观锁是一种排它锁，事务在操作数据时把这部分数据锁定，直到操作完毕后再解锁，这种方式容易造成系统吞吐量和性能方面的损失；

* 乐观锁在提交操作时检查是否违反数据完整性，大多数基于版本（Version）机制实现，MVCC就是一种乐观锁。

而在MySQL中，快照读实现了MVCC的非阻塞读功能。其为事务分配单向增长的时间戳，每次修改保存一个版本，版本与事务时间戳关联，读操作只读该事务开始前的数据库的快照。

MVCC即在对数据做修改操作时候，并不对原数据做修改，而是在数据基础上追加一个修改后的数据，并通过一个唯一的版本号做区分，版本号一般通过自增的方式，在读取数据时候，读到的实际是当前版本号对应的一份快照数据。

```txt
比如一个键值对数据K->{V.0}，此时value的版本号为0，

操作1首先对数据做修改，读取到的0版本号的数据，对其做修改提交事务后便成为K-> {V.0，V.1}，

操作2之后读到的数据是版本号1的数据，对其做修改后提交事务成功变为K->{V.0, V.1, V.2}。

每次修改只是往后面进行版本号以及数据值追加，通过这种方式使得每个事务操作到的是自己版本号内的数据，实现事务之间的隔离。也可以通过指定版本号访问对应的数据。
```

## 2、Etcd实现MVCC

### （1）版本号

* 版本号的分类和定义

| 版本号类型     | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| Revision       | 作用域为集群，逻辑时间戳，全局单调递增，任何key的增删改都会使其自增 |
| CreateRevision | 作用域为key，等于创建这个key时集群的Revision，直到删除前都保持不变 |
| ModRevision    | 作用域为key，等于修改这个key时集群的Revision，只要这个key更新都会自增 |
| Version        | 作用域为key， 这个key刚创建时Version为1，之后每次更新都会自增，即这个key从创建以来更新的总次数 |

* 添加数据查看版本号

```shell
/ # etcdctl put /name wanghao
OK
/ # etcdctl get /name
/name
wanghao

/ # etcdctl get /name -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":4,"raft_term":2},"kvs":[{"key":"L25hbWU=","create_revision":4,"mod_revision":4,"version":1,"value":"d2FuZ2hhbw=="}],"count":1}

#这里的key和value是以二进制的方式存储的，转换之后可以看到
/ # echo L25hbWU= | base64 -d
/name/ #
/ # echo d2FuZ2hhbw== | base64 -d
wanghao/ #
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 4    |
| create_revision | 4    |
| mod_revision    | 4    |
| version         | 1    |

* 再次给相同的key添加不同的value

```shell
/ # etcdctl put /name wanghao1
OK
/ # etcdctl get /name -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":5,"raft_term":2},"kvs":[{"key":"L25hbWU=","create_revision":4,"mod_revision":5,"version":2,"value":"d2FuZ2hhbzE="}],"count":1}
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 5    |
| create_revision | 4    |
| mod_revision    | 5    |
| version         | 2    |

* 再次给相同的key添加不同的value

```shell
/ # etcdctl put /name wanghao2
OK
/ # etcdctl get /name -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":6,"raft_term":2},"kvs":[{"key":"L25hbWU=","create_revision":4,"mod_revision":6,"version":3,"value":"d2FuZ2hhbzI="}],"count":1}
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 6    |
| create_revision | 4    |
| mod_revision    | 6    |
| version         | 3    |

* 添加不同的key

```shell
/ # etcdctl put /age 18
OK
/ # etcdctl get /age -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":7,"raft_term":2},"kvs":[{"key":"L2FnZQ==","create_revision":7,"mod_revision":7,"version":1,"value":"MTg="}],"count":1}
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 7    |
| create_revision | 7    |
| mod_revision    | 7    |
| version         | 1    |

* 再次添加不同的value

```shell
/ # etcdctl get /age -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":8,"raft_term":2},"kvs":[{"key":"L2FnZQ==","create_revision":7,"mod_revision":8,"version":2,"value":"MTk="}],"count":1}
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 8    |
| create_revision | 7    |
| mod_revision    | 8    |
| version         | 2    |

* 再次添加不同的value

```shell
/ # etcdctl put /age 20
OK
/ # etcdctl get /age -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":9,"raft_term":2},"kvs":[{"key":"L2FnZQ==","create_revision":7,"mod_revision":9,"version":3,"value":"MjA="}],"count":1}
```

版本号信息

| 版本号          | 值   |
| --------------- | ---- |
| revision        | 9    |
| create_revision | 7    |
| mod_revision    | 9    |
| version         | 3    |

* 结论

revision：集群中，4->5->6->7->8->9，只要有key的变动就会增加，全局单调递增。

create_revision：4->4->4->7->7->7创建当前key时的revision值。

mod_revision：4->5->6->7->8->9，修改这个key时的revision的值。

version：1->2->3->1->2->3，根据key的变动去增加，初始值为1。

etcd里面存在一个全局的总版本号revision，充当了逻辑时钟的概念，对每个key做一些修改删除操作都会触发主版本号自增。而每个key所作的就是在创建或者修改时候，记录下该value对应的主版本号。可以通过主版本号来找到任意数据的修改历史。如果双双记录的话，可以通过key查询到它对应的所有版本号，然后可以通过最新版本号找到对应的value。

### （2）MVCC的实现

在内存中维护了一个B-Tree作为key与对应版本号的映射关系，这个结构叫做treeIndex，又使用了BoltDB提供了版本号与对应value的映射关系以及数据的持久化存储。

当查询一个数据时候，首先通过treeIndex定位到最新的版本号revision（如果客户端有指定版本号则查询指定的revision），再通过revision定位到对应的value。

```shell
#查询不同版本的值，--rev表示当前key对应的revision的值的value的值
/ # etcdctl get /age --rev=9
/age
20
/ # etcdctl get /age --rev=8
/age
19
/ # etcdctl get /age --rev=7
/age
18
```



# 五、Etcd的监听Watch

## 1、watch机制简介

Etcd的Watch机制可以实时地订阅到etcd中增量的数据更新，watch支持指定单个key，也可以指定一个key的前缀。Watch观察将要发生或者已经发生的事件，输入和输出都是流;输入流用于创建和取消观察，输出流发送事件。一个观察RPC可以在一次性在多个key范围上观察，并为多个观察流化事件，整个事件历史可以从最后压缩修订版本开始观察。

watch又分为两类，一类是watch某一类的key，这种叫做key watcher，一类是通过--prefix，通过模糊匹配来查询以斜杠开头的所有key的变更，这种叫做range watcher。

## 2、watch的实现

etcd的mvcc模块对外提供了两种访问键值对的实现，一种是键值存储kvstore，另一种是watchableStore。它们都实现了KV接口，KV接口的具体实现则是store结构体。

```go
func testWatch() {
    s := newWatchableStore()

    w := s.NewWatchStream()

    w.Watch(start_key: foo, end_key: nil)

    w.Watch(start_key: bar, end_key: nil)

    for {
        consume := <- w.Chan()
    }
}
```

在上面的实现中，先调用了watchableStore，当要使用Watch功能时，创建了一个watchStream。创建出来的w可以监听的键为hello，之后就可以消费 w.Chan()返回的channel。键为hello的任何变化，都会通过这个channel发送给客户端。watchStream实现了在大量kv的变化中，过滤出当前所监听的key，将 key的变化输出。

### （1）watchableStore存储

每一个watchableStore其实都组合了来自store结构体的字段和方法，除此之外，还有两个watcherGroup类型的字段，watcherGroup管理多个watcher，能够根据key快速找到监听该key的一个或多个watcher。其中unsynced用于存储未同步完成的实例，synced用于存储已经同步完成的实例。

watchableStore收到了所有key的变更后，将这些key交给synced（watchGroup），synced能够快速地从所有key中找到监听的key。将这些key发送给对应的watcher，这些watcher再通过chan()将变更信息发送出去。

### （2）syncWatchers 同步监听

在初始化一个新的watchableStore时，etcd会创建一个用于同步watcherGroup的Goroutine，在syncWatchersLoop循环中会每隔100ms调用一次syncWatchers方法，将所有未通知的事件通知给所有的监听者，这可以说是整个模块的核心。

syncWatchers方法中总共做了三件事情，首先是根据当前的版本从未同步的watcherGroup中选出一些待处理的任务，然后从BoltDB中取当前版本范围内的数据变更并将它们转换成事件，事件和watcherGroup在打包之后会通过send方法发送到每一个watcher对应的Channel中。

### （3）客户端监听事件

客户端监听键值对时，调用的正是Watch方法，Watch在stream中创建一个新的watcher，并返回对应的WatchID。

WatchID是WatchStream中传递的观察者ID。当用户没有提供可用的ID时，如果有传递该值，etcd将自动分配一个ID。如果传递的ID已经存在，则会返回 ErrWatcherDuplicateID错误。

当etcd收到客户端的watch请求，如果请求携带了revision参数，则比较请求的revision和store当前的revision，如果大于当前revision，则放入synced 组中，否则放入unsynced组。

### （4）服务端处理监听

当 etcd 服务启动时，会在服务端运行一个用于处理监听事件的watchServer gRPC服务。

当客户端想要通过Watch结果监听某一个Key或者一个范围的变动，在每一次客户端调用服务端处理监听事件的服务都会创建两个Goroutine，其中一个协程会负责向监听者发送数据变动的事件，另一个协程会负责处理客户端发来的事件。

* ### **recvLoop**

recvLoop协程主要用来负责处理客户端发来的事件。

在用于处理客户端的recvLoop方法中调用了mvcc模块暴露出的 watchStream.Watch方法，该方法会返回一个可以用于取消监听事件的watchID；当gRPC 流已经结束后者出现错误时，当前的循环就会返回，两个Goroutine也都会结束。

* ### sendLoop

如果出现了更新或者删除事件，就会被发送到watchStream持有的Channel中，而sendLoop会通过select来监听多个Channel中的数据并将接收到的数据封装成 pb.WatchResponse结构并通过gRPC流发送给客户端。

对于每一个Watch请求来说，watchServer会根据请求创建两个用于处理当前请求的Goroutine，这两个协程会与更底层的mvcc模块协作提供监听和回调功能。

## 3、watch的使用

### （1）监听单个key

* 创建key1

```shell
[root@wanghao02 ~]# docker exec -it etcd-node3 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl put /key1 value1
OK
/ # etcdctl get /key1
/key1
value1
```

* 监听key

```shell
/ # etcdctl watch /key1

```

* 新建一个session修改key

```shell
[root@wanghao02 ~]# docker exec -it etcd-node3 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl get /key1
/key1
value1

#更新ke，将key1的value值更新为value-new1
/ # etcdctl put /key1 value-new1
OK
/ # etcdctl get /key1
/key1
value-new1

#删除key
/ # etcdctl del /key1
1
/ # etcdctl get /key1
```

* 监听的key的变化

```shell
/ # etcdctl watch /key1

PUT
/key1
value-new1
DELETE
/key1
```

### （2）监听范围内的key

* 创建key1，key2，key3，key4，key5

```shell
/ # etcdctl put /key1 value1
OK
/ # etcdctl put /key2 value2
OK
/ # etcdctl put /key3 value3
OK
/ # etcdctl put /key4 value4
OK
/ # etcdctl put /key5 value5
OK
/ # etcdctl get /key --prefix
/key1
value1
/key2
value2
/key3
value3
/key4
value4
/key5
value5
```

* 新建一个session监听key

```shell
/ # etcdctl watch /key --prefix

```

* 修改key的value

```shell
#修改key1
/ # etcdctl put /key1 value-new1
OK

#删除key2
/ # etcdctl del /key2
1

#修改key3
/ # etcdctl put /key3 value-new3
OK

#连续两次修改key5
/ # etcdctl put /key5 value-new5
OK
/ # etcdctl put /key5 value-new4
OK

#查看最终结果
/ # etcdctl get /key --prefix
/key1
value-new1
/key3
value-new3
/key4
value4
/key5
value-new4
```

* 查看监听的内容

```shell
/ # etcdctl watch /key1
/ # etcdctl watch /key --prefix
#监听的修改key1的内容
PUT
/key1
value-new1

#监听的删除key2的内容
DELETE
/key2

#监听的修改key3的内容
PUT
/key3
value-new3

#监听的连续两次修改key5的内容
PUT
/key5
value-new5
PUT
/key5
value-new4
```



# 六、Etcd的租约Lease

## 1、Lease的简介

ETCD的Lease租约，它类似 TTL（Time To Live），用于etcd客户端与服务端之间进行活性检测。在到达TTL时间之前，etcd服务端不会删除相关租约上绑定的键值对；超过TTL时间，则会删除。因此需要在到达TTL时间之前续租，以实现客户端与服务端之间的保活。

Lease也是etcd v2与v3版本之间的重要变化之一。etcd v2版本并没有Lease概念，TTL直接绑定在key上面。每个 TTL、key创建一个HTTP/1.x 连接，定时发送续期请求给etcd Server。etcd v3则在 v2的基础上进行了重大升级，每个Lease都设置了一个TTL时间，具有相同TTL时间的key绑定到同一个 Lease，实现了Lease的复用，并且基于gRPC协议的通信实现了连接的多路复用。

## 2、Lease的实现

### （1）Lease的创建

当etcd服务端的gRPC Server接收到创建Lease的请求后，Raft模块首先进行日志同步；接着MVCC调用Lease模块的Grant接口，保存对应的日志条目到ItemMap 结构中，接着将租约信息存到boltdb；最后将LeaseID返回给客户端，Lease创建成功。

### （2）Lease与键值绑定

客户端根据返回的LeaseID，在执行写入和更新操作时，可以绑定该LeaseID。如命令行工具etcdctl 指定--lease参数，MVCC会调用Lease模块Lessor接口中的 Attach方法，将key关联到Lease的key内存集合ItemSet 中，以完成键值对与Lease租约的绑定。

## 3、Lease的处理命令

### （1）查看lease相关的命令

```shell
/ # etcdctl lease --help
NAME:
        lease - Lease related commands

USAGE:
        etcdctl lease <subcommand>

API VERSION:
        3.3


COMMANDS:
        grant           Creates leases
        revoke          Revokes leases
        timetolive      Get lease information
        list            List all active leases
        keep-alive      Keeps leases alive (renew)

GLOBAL OPTIONS:
      --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""                                 identify secure client using this TLS certificate file
      --command-timeout=5s                      timeout for short running command (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connections
  -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
      --hex[=false]                             print byte strings as hex encoded strings
      --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verification
      --insecure-transport[=true]               disable transport security for client connections
      --keepalive-time=2s                       keepalive time for client connections
      --keepalive-timeout=6s                    keepalive timeout for client connections
      --key=""                                  identify secure client using this TLS key file
      --user=""                                 username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)
```

命令及说明如下：

| 命令       | 说明            |
| ---------- | --------------- |
| grant      | 创建lease       |
| revoke     | 撤销lease       |
| timetolive | 获取lease的信息 |
| list       | 查看所有的lease |
| keep-alive | 续约lease       |

### （2）创建lease

LeaseGrant方法创建一个租约。当服务器在给定time to live时间内没有接收到keepAlive时租约过期。如果租约过期则所有附加在租约上的key将过期并被删除。每个过期的key在事件历史中生成一个删除事件。

```shell
etcdctl lease grant <ttl>
```

TTL建议以秒为单位的time-to-live，成功后返回lease id。

```shell
#创建一个15s的lease
/ # etcdctl lease grant 15
lease 226b84643e564c5d granted with TTL(15s)

#查看当前的租约
/ # etcdctl lease list
found 1 leases
226b84643e564c5d

#15s之后，查看租约，因为时间已过，所以删除了
/ # etcdctl lease list
found 0 leases
```

### （3）获取lease的信息

LeaseTimeToLive方法获取租约的信息。

```shell
#etcdctl lease timetolive <lease id>

#创建1000s的lease
/ # etcdctl lease grant 1000
lease 226b84643e564c60 granted with TTL(1000s)

/ # etcdctl lease list
found 1 leases
226b84643e564c60

#查看lease的信息，可以看到lease剩下的事件
/ # etcdctl lease timetolive 226b84643e564c60
lease 226b84643e564c60 granted with TTL(1000s), remaining(914s)
```

### （4）续约lease

LeaseKeepAlive方法维持一个租约。LeaseKeepAlive通过从客户端到服务器端的流化的keep alive请求和从服务器端到客户端的流化的keep alive应答来维持租约。续约时不涉及指定的key，只是针对leaseid进行的操作。

可以对一个存在的lease进行续约，续约后，租约的时间重新开始计时。

```shell
#etcdctl lease keep-alive <lease id>
```

* 单次续约

```shell
/ # etcdctl lease timetolive 226b84643e564c60
lease 226b84643e564c60 granted with TTL(1000s), remaining(691s)

#续约lease
/ # etcdctl lease keep-alive 226b84643e564c60
lease 226b84643e564c60 keepalived with TTL(1000)
^C

#查看续约后，lease剩下的时间
/ # etcdctl lease timetolive 226b84643e564c60
lease 226b84643e564c60 granted with TTL(1000s), remaining(998s)
```

* 多次续约

```shell
#创建一个20s的租约
/ # etcdctl lease grant 20
lease 226b84643e564c6d granted with TTL(20s)

#续约不中断时，会一直续约，续约的周期时间小于租期时间（本次的周期为5s，即每5s续约一次）
/ # etcdctl lease  keep-alive 226b84643e564c6d
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
lease 226b84643e564c6d keepalived with TTL(20)
```

* lease过期后续约

```shell
#创建一个20s
/ # etcdctl lease grant 20
lease 226b84643e564c73 granted with TTL(20s)

#查看lease
/ # etcdctl lease list
found 1 leases
226b84643e564c73

#等到过期后查看lease，显示lease已经过期
/ # etcdctl lease timetolive 226b84643e564c73
lease 226b84643e564c73 already expired

#过期后续约，显示想要续约的lease已经过期
/ # etcdctl lease keep-alive 226b84643e564c73
lease 226b84643e564c73 expired or revoked.
```

### （5）撤销lease

LeaseRevoke方法用于撤销指定LeaseID的租约，同时绑定到该Lease上的键值都会被移除。

```shell
#etcdctl revoke <lease id>

/ # etcdctl lease list
found 1 leases
226b84643e564c60

#查看当前的lease，时间还剩下625s
/ # etcdctl lease timetolive 226b84643e564c60
lease 226b84643e564c60 granted with TTL(1000s), remaining(625s)

#撤销lease
/ # etcdctl lease revoke 226b84643e564c60
lease 226b84643e564c60 revoked

#撤销后
/ # etcdctl lease list
found 0 leases
```

## 4、Lease与键值的使用

### （1）创建lease

```shell
/ # etcdctl lease grant 1000
lease 226b84643e564c76 granted with TTL(1000s)

/ # etcdctl lease list
found 1 leases
226b84643e564c76

/ # etcdctl lease timetolive 226b84643e564c76
lease 226b84643e564c63 granted with TTL(1000s), remaining(969s)
```

### （2）创建键值对并绑定lease

```shell
#etcdctl put key value --lease=lease_id

#创建键值，并绑定lease
/ # etcdctl put test lease --lease=226b84643e564c76
OK

#查看键值信息的时候，可以看到lease的信息
/ # etcdctl get test -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":34,"raft_term":2},"kvs":[{"key":"dGVzdA==","create_revision":34,"mod_revision":34,"version":1,"value":"bGVhc2U=","lease":2480221585875029110}],"count":1}
```

### （3）续期lease

续期的对象是lease id，不涉及到key，所以对key没有影响

```shell
/ # etcdctl lease keep-alive 226b84643e564c76
lease 226b84643e564c76 keepalived with TTL(1000)
^C
/ # etcdctl lease timetolive 226b84643e564c76
lease 226b84643e564c76 granted with TTL(1000s), remaining(979s)

/ # etcdctl get test
test
lease
```

### （4）删除键值

删除键值不影响lease

```shell
#删除键值
/ # etcdctl del test
1

/ # etcdctl get test

#lease依然存在，不受key删除的影响
/ # etcdctl lease list
found 1 leases
226b84643e564c76

/ # etcdctl lease timetolive 226b84643e564c76
lease 226b84643e564c76 granted with TTL(1000s), remaining(861s)
```

### （5）撤销lease

```shell
#创建key并绑定到226b84643e564c76上
/ # etcdctl put test lease --lease 226b84643e564c76
OK

#key的revision为36
/ # etcdctl get test -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":36,"raft_term":2},"kvs":[{"key":"dGVzdA==","create_revision":36,"mod_revision":36,"version":1,"value":"bGVhc2U=","lease":2480221585875029110}],"count":1}

/ # etcdctl lease list
found 1 leases
226b84643e564c76

/ # etcdctl lease timetolive 226b84643e564c76
lease 226b84643e564c76 granted with TTL(1000s), remaining(737s)

#撤销lease 226b84643e564c76
/ # etcdctl lease revoke 226b84643e564c76
lease 226b84643e564c76 revoked

#lease已经被撤销
/ # etcdctl lease list
found 0 leases

/ # etcdctl lease timetolive 226b84643e564c76
lease 226b84643e564c76 already expired

#查不到key的信息，因为在撤销lease的时候，可以也被删除了
/ # etcdctl get test


#revision的值为37，相比之前的值加一，因为执行了删除key的操作，所以版本号加一
/ # etcdctl get test -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":37,"raft_term":2}}
```

### （6）lease到期

lease到期后会删除绑定的键值

```shell
#创建一个120s的leasae
/ # etcdctl lease grant 120
lease 226b84643e564c8e granted with TTL(120s)

/ # etcdctl lease list
found 1 leases
226b84643e564c8e

/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(109s)

#设置键值并绑定lease
/ # etcdctl put test lease --lease=226b84643e564c8e
OK

#当前test的revision的值为42
/ # etcdctl get test -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":42,"raft_term":2},"kvs":[{"key":"dGVzdA==","create_revision":42,"mod_revision":42,"version":1,"value":"bGVhc2U=","lease":2480221585875029134}],"count":1}

#等到lease到期
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(65s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(64s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(51s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(39s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(24s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(4s)
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e granted with TTL(120s), remaining(0s)
#lease已经到期
/ # etcdctl lease timetolive 226b84643e564c8e
lease 226b84643e564c8e already expired

#查不到test的值
/ # etcdctl get test

#查看test的revision的值为43，是因为lease到期后会删除绑定的key，所以revision的值会加一
/ # etcdctl get test -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":43,"raft_term":2}}
```

# 七、Etcd的事务Transaction

## 1、事务的背景

当需要批量操作etcd多个key的时候，常常需要让多次操作形成一个原子性的效果，要么同时成功，要么同时失败。etcd提供了事务的机制，可以实现多个key的原子性操作。

## 2、事务的简介

事务可以使得etcd服务端在单个事物中自动处理多个外部请求。对于键值存储库的修改，这意味着该存储库的mod_revision仅对事务增加一次，并且该事务生成的所有事件都将具有相同的mod_revision。需要注意的是，禁止在单个事务中多次修改同一 key。

事务中的每个比较都会检查存储中的单个key，类似于If操作，检查是否存在值，与给定值进行比较或检查键的mod_revision或revision。两种不同的比较可能适用于相同或不同的key。

所有比较都是原子操作。如果所有比较都为真，则表示事务成功，而etcd则应用事务的then/success请求块，否则，则认为失败并应用else/failure请求块。

## 3、事务的处理过程

### （1）Compare

对给定的条件进行比较，根据比较的结果去处理。

* 比较的条件

| 比较条件       | 说明                |
| -------------- | ------------------- |
| CreateRevision | 创建key时的revision |
| LeaseValue     | 租约id              |
| ModRevision    | 修改key时的revision |
| Value          | key的值             |
| Version        | key自身的版本       |

* 比较运算符

| 比较运算符 | 说明   |
| ---------- | ------ |
| <          | 小于   |
| >          | 大于   |
| =          | 等于   |
| !=         | 不等于 |

### （2）Success

当compare的结果为true（即条件成立）时执行的处理。注意：etcd同一个事务不支持对相同的key进行多次的修改。

处理的内容：

| 处理方法 | 说明          |
| -------- | ------------- |
| get      | 获取key       |
| put      | 更新或添加key |
| delete   | 删除key       |

### （3）Failure

当compare的结果为failure（即条件不成立）时执行的处理。注意：etcd同一个事务不支持对相同的key进行多次的修改。

| 处理方法 | 说明          |
| -------- | ------------- |
| get      | 获取key       |
| put      | 更新或添加key |
| delete   | 删除key       |

## 4、事务的使用

进入容器，并查看etcd事务相关的命令。

```shell
[root@wanghao02 ~]# docker exec -it etcd-node3 /bin/sh
/ # export ETCDCTL_API=3
/ # etcdctl txn --help
NAME:
        txn - Txn processes all the requests in one transaction

USAGE:
        etcdctl txn [options]

OPTIONS:
  -i, --interactive[=false]     Input transaction in interactive mode

GLOBAL OPTIONS:
      --cacert=""                               verify certificates of TLS-enabl                                                                                                                                                             ed secure servers using this CA bundle
      --cert=""                                 identify secure client using thi                                                                                                                                                             s TLS certificate file
      --command-timeout=5s                      timeout for short running comman                                                                                                                                                             d (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connecti                                                                                                                                                             ons
  -d, --discovery-srv=""                        domain name to query for SRV rec                                                                                                                                                             ords describing cluster endpoints
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
      --hex[=false]                             print byte strings as hex encode                                                                                                                                                             d strings
      --insecure-discovery[=true]               accept insecure SRV records desc                                                                                                                                                             ribing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verifica                                                                                                                                                             tion
      --insecure-transport[=true]               disable transport security for c                                                                                                                                                             lient connections
      --keepalive-time=2s                       keepalive time for client connec                                                                                                                                                             tions
      --keepalive-timeout=6s                    keepalive timeout for client con                                                                                                                                                             nections
      --key=""                                  identify secure client using thi                                                                                                                                                             s TLS key file
      --user=""                                 username[:password] for authenti                                                                                                                                                             cation (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, j                                                                                                                                                             son, protobuf, simple, table)
```

### （1）比较CreateRevision

* 查看/key1的版本value

```shell
#查看key value值
/ # etcdctl get /key1
/key1
value-new1

#查看key的版本
/ # etcdctl get /key1 -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":25,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":16,"mod_revision":21,"version":2,"value":"dmFsdWUtbmV3MQ=="}],"count":1}
```

* 以交互的方式输入事务，比较create_revision用create方法

```shell
/ # etcdctl txn -i		 #-i或者--interactive表示以交互的方式输入事务 
compares:                #compare中添加比较条件
create("/key1") = "16"   #比较/key1的create_revision是否为16

success requests (get, put, del):     #当比较结果为true时的处理
put /key1 value1					  #将/key1的值设置为value1

failure requests (get, put, del):	  #当比较结果为false时的处理
put /key1 value2					  #将/key1的值设置为value2

SUCCESS				      #事务的执行结果，SUCCESS表示比较的结果为true，因为/key1的create_revision为16，所以比较结果为true

OK						  #处理的执行的结果，OK表示处理语句处理成功
```

* 查看结果

```shell
/ # etcdctl get /key1
/key1
value1
```

### （2）比较ModRevision

* 查看/key1和/key6的版本value

```shell
#查看key value值
/ # etcdctl get /key1
/key1
value1
#/key6未设置值，值为空
/ # etcdctl get /key6
/ #

#查看key的版本
/ # etcdctl get /key1 -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":26,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":16,"mod_revision":26,"version":3,"value":"dmFsdWUx"}],"count":1}

/ # etcdctl get /key6 -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":26,"raft_term":2}}
```

* 执行事务，比较mod_revision用mod方法

```shell
#由上面获取到的/key1的信息可以得知：mod_revision的值为26，不等于27，所以比较结果为false，执行FAILURE的处理（获取/key1的值，/key6设置值）
/ # etcdctl txn -i
compares:
mod("/key1") = "27"

success requests (get, put, del):
get /key1
get /key2
put /key6 value6

failure requests (get, put, del):
get /key1
put /key6 failure

FAILURE

/key1
value1

OK
```

* 查看结果

```shell
/ # etcdctl get /key1
/key1
value1
/ # etcdctl get /key6
/key6
failure
```

### （3）比较Version

* 查看/key1的value和版本信息

```shell
/ # etcdctl put /key1 value1
OK

/ # etcdctl get /key1 -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":44,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":29,"mod_revision":29,"version":1,"value":"dmFsdWUx"}],"count":1}
```

* 执行事务，比较version用version方法

```shell
#判断/key1的version值是不是不是1，如果不为1，则表示/key1被修改过，如果为1，则表示未被修改
/ # etcdctl txn -i
compares:
version("/key1") != "1"

success requests (get, put, del):
put result "the value of key1 has been changed"

failure requests (get, put, del):
put result "nobody changed the value of /key1"

FAILURE

OK

/ # etcdctl get result
result
nobody changed the value of /key1
```

* 查看结果

```shell
/ # etcdctl get /key1
/key1
value1

/ # etcdctl get /key1 -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":44,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":29,"mod_revision":29,"version":1,"value":"dmFsdWUx"}],"count":1}
```

### （4）比较value

* 查看key的值和版本信息

```shell
/ # etcdctl get /key1
/key1
value1

/ # etcdctl get /key1 -w=json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":30,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":29,"mod_revision":29,"version":1,"value":"dmFsdWUx"}],"count":1}
```

* 执行事务，比较value用value方法

```shell
#判断key1的value值是否是value2，如果相同则输出the value of /key1 is value2获取/key1的value值，如果不相同则输出the value of /key1 is not value2并获取/key1的value值。
/ # etcdctl txn -i
compares:
value("/key1") = "value2"

success requests (get, put, del):
put result "the value of /key1 is value2"
get result
get /key1

failure requests (get, put, del):
put result "the value of /key1 is not value2"
get result
get /key1

FAILURE

OK

result
the value of /key1 is not value2

/key1
value1
```

* 查看结果

```shell
/ # etcdctl get /key1
/key1
value1

/ # etcdctl get result
result
the value of /key1 is not value2
```

### （5）比较lease id（暂时无法使用）





# 八、Etcd的Compact

## 1、Compact简介

Compact是ETCD提供的清理命令，由于ETCD会记录历史，所以如果不限制的话，历史会越来越多，占用磁盘空间，ETCD提供了etcd compact {revision}命令，可以把{revision}之前（包含）的历史全部清除。虽然compact翻译过来是压缩的、紧密的，但实际上，ETCD的compact执行了就真的没了，所以更贴近清理的概念。

## 2、Compact的使用

* 创建key

```shell
#将/key1连选设置4个版本的值
/ # etcdctl put /key1 a
OK
/ # etcdctl put /key1 b
OK
/ # etcdctl put /key1 c
OK
/ # etcdctl put /key1 d
OK
```

* 查看各个版本的值

```shell
#查看最新版本对应的revision
/ # etcdctl get /key1 -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":50,"raft_term":2},"kvs":[{"key":"L2tleTE=","create_revision":47,"mod_revision":50,"version":4,"value":"ZA=="}],"count":1}

#查看各个版本对应的值
/ # etcdctl get /key1 --rev=50
/key1
d
/ # etcdctl get /key1 --rev=49
/key1
c
/ # etcdctl get /key1 --rev=48
/key1
b
/ # etcdctl get /key1 --rev=47
/key1
a
```

* 压缩版本

```shell
#将版本号为48之前的key进行压缩
/ # etcdctl compaction 48
compacted revision 48

#48版本之前的数据被删除
/ # etcdctl get /key1 --rev=47
Error: etcdserver: mvcc: required revision has been compacted

/ # etcdctl get /key1 --rev=48
/key1
b
/ # etcdctl get /key1 --rev=49
/key1
c
/ # etcdctl get /key1 --rev=50
/key1
d
```

* 再设置/key2的几个版本

```shell
/ # etcdctl put /key2 1
OK
/ # etcdctl put /key2 2
OK
/ # etcdctl put /key2 3
OK
/ # etcdctl put /key2 4
OK
/ # etcdctl get /key2 -w json
{"header":{"cluster_id":12975046451272761543,"member_id":11390638367855649387,"revision":55,"raft_term":2},"kvs":[{"key":"L2tleTI=","create_revision":52,"mod_revision":55,"version":4,"value":"NA=="}],"count":1}
/ # etcdctl get /key2 --rev=55
/key2
4
/ # etcdctl get /key2 --rev=54
/key2
3
/ # etcdctl get /key2 --rev=53
/key2
2
/ # etcdctl get /key2 --rev=53
/key2
2
/ # etcdctl get /key2 --rev=52
/key2
1
```

* compact进行压缩

```shell
/ # etcdctl compaction --physical=true 53
compacted revision 53


#进行压缩后，53号之前的所有的key都被删除了
/ # etcdctl get /key1 --rev=48
Error: etcdserver: mvcc: required revision has been compacted
/ # etcdctl get /key1 --rev=49
Error: etcdserver: mvcc: required revision has been compacted
/ # etcdctl get /key1 --rev=50
Error: etcdserver: mvcc: required revision has been compacted
/ # etcdctl get /key2 --rev=51
Error: etcdserver: mvcc: required revision has been compacted
/ # etcdctl get /key2 --rev=52
Error: etcdserver: mvcc: required revision has been compacted
/ # etcdctl get /key2 --rev=53
/key2
2
/ # etcdctl get /key2 --rev=54
/key2
3
/ # etcdctl get /key2 --rev=55
/key2
4
```



# 九、Etcd的metrics

## 1、Metrics

Etcd的指标用来实时监控和调试。etcd不会保留metrics，当成员发生重启，metrics将被重置。所以etcd可结合prometheus，对etcd进行监控。

## 2、Etcd namespace metrics

etcd前缀下的指标用于监控和警报。它们是稳定的高级指标。

### （1）Server

这些指标描述了etcd服务器的状态。为了检测故障或故障排除问题，应密切监控每个生产etcd集群的服务器指标。所有这些指标都以 etcd_server_作为前缀。

指标名称及说明如下：

| 名称                      | 描述                             | 类型    |
| ------------------------- | -------------------------------- | ------- |
| has_leader                | 领导者是否存在。1是存在，0不是。 | Gauge   |
| leader_changes_seen_total | 看到的领导者变更次数。           | Counter |
| proposal_committed_total  | 提交的共识提案总数。             | Gauge   |
| proposal_applied_total    | 已应用的共识提案总数。           | Gauge   |
| proposal_pending          | 当前待处理提案的数量。           | Gauge   |
| proposal_failed_total     | 看到的失败提案总数。             | Counter |

* has_leader

指示成员是否有领导者。如果成员没有领导者，则完全不可用。如果集群中的所有成员都没有任何领导者，则整个集群完全不可用。

* leader_changes_seen_total

计算成员自开始以来看到的领导者更改次数。领导层的快速变化会显着影响etcd的性能。它还表明领导者不稳定，可能是由于网络连接问题或etcd集群负载过大。

* proposals_committed_total

记录提交的共识提案总数。如果集群运行状况良好，该指标应该会随着时间的推移而增加。etcd集群的几个健康成员可能同时拥有不同的总提交提案。这种差异可能是由于在启动后从对等体恢复、落后于领导者，或者是领导者因此拥有最多的提交。监控集群中所有成员的这个指标很重要；单个成员与其领导者之间持续较大的滞后表明该成员行动缓慢或不健康。

* proposals_applied_total

记录应用的共识提案总数。etcd服务器异步应用每个提交的提案。proposals_committed_total和proposals_applied_total之间的差异通常应该很小（即使在高负载下也应该在几千以内）。如果它们之间的差异继续上升，则表明etcd服务器过载。在应用昂贵的查询（例如大范围查询或大型txn操作）时可能会发生这种情况。

* proposals_pending

表示有多少提案排队等待提交。未决提案上升表明客户端负载很高或成员无法提交提案。

* proposals_failed_total

通常与两个问题有关：与领导者选举相关的临时故障或由于集群中的仲裁丢失而导致的更长停机时间。

### （2）Disk

这些指标描述磁盘操作的状态。所有这些指标都以etcd_disk_为前缀。

| 名称                            | 描述                       | 类型      |
| ------------------------------- | -------------------------- | --------- |
| wal_fsync_duration_seconds      | wal调用的fsync的延迟分布   | Histogram |
| backend_commit_duration_seconds | 后端调用的提交的延迟分布。 | Histogram |

高磁盘操作延迟 (wal_fsync_duration_seconds或backend_commit_duration_seconds) 通常表示磁盘问题。它可能会导致高请求延迟或使集群不稳定。

### （3）Network

这些指标描述了网络的状态。所有这些指标都以 etcd_network_为前缀。

| 名称                             | 描述                                   | 类型          |
| -------------------------------- | -------------------------------------- | ------------- |
| peer_sent_bytes_total            | 发送到具有 ID 的对等方的总字节数To。   | Counter(To)   |
| peer_received_bytes_total        | 从具有 ID 的对等方接收的总字节数From。 | Counter(From) |
| peer_sent_failures_total         | 来自具有 ID 的对等方的发送失败总数To。 | Counter(To)   |
| peer_received_failures_total     | 从具有 ID 的对等方接收失败的总数From。 | Counter(From) |
| peer_round_trip_time_seconds     | 同行之间的往返时间直方图。             | Histogram(To) |
| client_grpc_sent_bytes_total     | 发送到 grpc 客户端的总字节数。         | Counter       |
| client_grpc_received_bytes_total | 接收到 grpc 客户端的总字节数。         | Counter       |

* peer_sent_bytes_total

计算发送到特定对等方的总字节数。通常领导成员发送的数据比其他成员多，因为它负责传输复制的数据。

* peer_received_bytes_total

计算从特定对等方接收的总字节数。通常追随者成员只从领导者那里接收数据。

## 3、etcd_debugging namespace metrics

etcd_debugging前缀下的指标用于调试。它们非常依赖于实现且易变。在新的 etcd 版本中，它们可能会在没有任何警告的情况下被更改或删除。当某些指标变得更加稳定时，它们可能会移至etcd前缀。

### （1）snapshot

| 名称                                 | 描述                     | 类型      |
| ------------------------------------ | ------------------------ | --------- |
| snapshot_save_total_duration_seconds | 快照调用的保存总延迟分布 | Histogram |

异常高的快照持续时间 ( snapshot_save_total_duration_seconds) 表示磁盘问题，并可能导致集群不稳定。

## 4、Prometheus supplied metrics

Prometheus客户端在go和process命名空间下提供了许多指标。

| 名称             | 描述                         | 类型    |
| ---------------- | ---------------------------- | ------- |
| process_open_fds | 打开的文件描述符数。         | Counter |
| process_max_fds  | 打开的文件描述符的最大数量。 | Counter |

大量的文件描述符 ( process_open_fds) 使用（即接近进程的文件描述符限制process_max_fds）表示潜在的文件描述符耗尽问题。如果文件描述符耗尽，etcd可能会因为无法创建新的WAL文件而发生恐慌。
