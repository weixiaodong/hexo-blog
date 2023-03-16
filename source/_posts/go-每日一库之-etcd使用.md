---
title: go-每日一库之-etcd使用
date: 2023-03-16 10:39:18
tags:
---

# 一 概述

## 1.1 etcd简介

etcd是coreOS团队于2013年6月发起的开源项目，它的目标是构建一个高可用的分布式键值（key-value）数据库。 etcd内部采用raft协议作为一致性算法，etcd基于go语言实现。

<!--more-->

## 1.2 发展历史

![](/images/etcd发展历史.png)


## 1.3 etcd的特点

- 简单： 安装配置简单，而且提供了http api进行交互，使用也很简单
- 安全： 支持ssl证书验证
- 快速： 根据官方提供的benchmark数据，单实例支持每秒2k+读操作
- 可靠： 采用raft算法，实现分布式系统数据的可用性和一致性

## 1.4 概念术语

- raft: etcd所采用的的保证分布式系统强一致性的算法
- node： 一个raft状态机实例
- member： 一个etcd实例。 它管理着一个node，并且可以为客户端请求提供服务。
- cluster: 由多个member构成可以协同工作的etcd集群
- peer： 对同一个etcd集群中另外一个member的称呼
- client: 向etcd集群发送http请求的客户端
- wal： 预写式日志，etcd用于持久化存储的日志格式。
- snapshot： etcd防止wal文件过多而设置的快照，存储etcd数据状态。
- proxy： etcd的一种模式，为etcd集群提供反向代理服务。
- leader: raft算法中通过竞选而产生的处理所有数据提交的节点
- follower: 竞选失败的节点作为raft中的从属节点，为算法提供强一致性保证。
- candidate: 当follower超过一定时间接收不到leader的心跳时转变为candidate开始竞选。
- term： 某个节点成为leader到下一次竞选时间，称为一个term。
- index： 数据项编号。 raft中通过term和index来定位数据

## 1.5 数据读写顺序

为了保证数据的强一致性，etcd集群中所有的数据流向都是一个方向，从leader（主机点）流向follower，也就是所有follower的数据必须与leader保持一致，如果不一致会被覆盖。

用户对于etcd集群所有节点进行读写

- 读取： 由于集群所有节点数据是强一致性的，读取可以从集群中随便哪个节点进行读取数据
- 写入： etcd集群有leader，如果写入往leader写入，可以直接写入，然后leader节点会把写入分发给所有follower，如果往follower写入，然后leader节点会把写入分发给所有follower


## 1.6 leader选举

假设三个节点的集群，三个节点上均运行timer（每个timer持续时间是随机的），raft算法使用随机timer来初始化leader选举流程，第一个节点率先完成了timer，随后它就会向其他两个节点发送成为leader的请求，其他节点接收到请求后会以投票回应然后第一个节点被选举为leader。

称为leader后，该节点会以固定时间间隔向其他节点发送通知，确保自己仍是leader，有些情况下当follower收不到leader的通知后，比如leader节点宕机或者失去连接，其他节点会重复之前选举过程选举出新的leader。

## 1.7 判断数据是否写入

etcd认为写入请求被leader节点处理并分发给了多数节点后，就是一个成功的写入。 

# 二 etcd架构及解析

## 2.1 架构图

![](/images/etcd架构图.png)


## 2.2 架构解析

从etcd的架构图中我们可以看到，etcd主要分为四个部分

- http server: 用于处理用户发送的API请求以及其他etcd节点的同步与心跳信息请求。
- store: 用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是etcd对用户提供的大多数API功能的具体实现
- raft: raft强一致性算法的具体实现，是etcd的核心。
- wal: write ahead log(预写式日志)，是etcd的数据存储方式。 除了在内存中存有所有数据的状态以及节点的索引以外，etcd就通过wal进行持久化存储。 wal中，所有的数据提交前都会事先记录日志。
	+ snapshot 是为了防止数据过多而进行的状态快照
	+ entry 表示存储的具体日志内容

通常，一个用户的请求发送过来，会经由http server转发给store进行具体的事务处理，如果涉及到节点的修改，则交给raft模块进行状态的变更、日志的记录，然后再同步给别的etcd节点以确认数据提交，最后进行数据的提交、再次同步。

# 三 应用场景

## 3.1 服务注册与发现

etcd可以用于服务的注册与发现

- 前后端业务注册发现

![](/images/etcd服务注册与发现.png)

中间件以及后端服务在etcd中注册，前端和中间件可以很轻松的从etcd中发现相关服务器，然后服务器之间根据调用关系相关绑定调用

- 多组后端服务器注册发现

![](/images/多组后端服务器注册与发现.png)

后端多个无状态相同副本的app可以同时注册到etcd中，前端可以通过haproxy从etcd中获取到后端的ip和端口组，然后进行请求转发，可以用来故障转移、屏蔽后端端口以及后端多组app实例

## 3.2 消息发布与订阅

![](/images/etcd消息发布与订阅.png)

etcd可以充当消息中间件，生产者可以往etcd中注册topic并发送消息，消费者从etcd中订阅topic，来获取生产者发送至etcd中的消息。

## 3.3 负载均衡

![](/images/etcd负载均衡.png)

后端多组相同的服务提供者可以经自己服务注册到etcd中，etcd并且会与注册的服务进行监控检查，服务请求这首先从etcd中获取到可用的服务提供者真正的ip:port，然后对此多组服务发送请求，etcd在其中充当了负载均衡的功能

## 3.4 分部署通知与协调

![](/images/etcd分部署通知与协调.png)

- 当etcd watch服务发现丢失，会通知服务检查
- 控制器向etcd发送启动服务，etcd通知服务进行相应操作
- 当服务完成work会讲状态更新至etcd，etcd对应会通知用户

## 3.5 分布式锁

![](/images/etcd分布式锁.png)

当有多个竞争者node节点，etcd作为总控，在分布式集群中与一个节点成功分配lock

## 3.6 分布式队列

![](/images/etcd分布式队列.png)

有多个node，etcd根据每个node来创建对应node的队列，根据不同的队列可以在etcd中找到对应的competitor


## 3.7 集群与监控与Leader选举

![](/images/etcd集群监控与leader选举.png)

etcd可以根据raft算法在多个node节点来选举出leader


# 四 安装部署

## 4.1 单机部署

```bash
hostnamectl set-hostname etcd-1
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -ivh epel-release-latest-7.noarch.rpm
# yum 仓库中的etcd版本为3.3.11，如果需要最新版本的etcd可以进行二进制安装
yum -y install etcd
systemctl enable etcd
```

可以查看yum安装的etcd的有效配置文件，根据自己的需求来修改数据存储目录，已经监听端口url/etcd的名称等

- etcd 默认将数据存放到当前路径的 default.etcd/ 目录下
- 在 http://localhost:2380 和集群中其他节点通信
- 在 http://localhost:2379 提供 HTTP API 服务，供客户端交互, 该节点的名称默认为 default
	+ heartbeat 为 100ms，后面会说明这个配置的作用
- election 为 1000ms，后面会说明这个配置的作用
- snapshot count 为 10000，后面会说明这个配置的作用
- 集群和每个节点都会生成一个 uuid
- 启动的时候，会运行 raft，选举出 leader

```bash
[root@VM_0_8_centos tmp]# grep -Ev "^#|^$" /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
ETCD_NAME="default"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
[root@VM_0_8_centos tmp]# systemctl status etcd
```
