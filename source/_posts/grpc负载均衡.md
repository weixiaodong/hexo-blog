---
title: grpc负载均衡
date: 2023-03-17 16:17:57
tags:
---

# grpc 负载均衡

## 1. 对每次调用进行负载均衡

grpc中的负载均衡是以每次调用为基础，而不是以每个连接为基础。 换句话说，即使所有的请求都来自一个客户端，我们仍希望它们在所有服务器上实现负载均衡。

<!--more-->

## 2. 负载均衡的方法

- 集中式

![](images/grpc集中式负载均衡.png)

在服务消费者和服务提供者之间有一个独立的负载均衡（LB），通常是专门的硬件设备如 F5，或者基于软件如 LVS，HAproxy等实现。LB上有所有服务的地址映射表，通常由运维配置注册，当服务消费方调用某个目标服务时，它向LB发起请求，由LB以某种策略，比如轮询（Round-Robin）做负载均衡后将请求转发到目标服务。LB一般具备健康检查能力，能自动摘除不健康的服务实例。

该方案主要问题：服务消费方、提供方之间增加了一级，有一定性能开销，请求量大时，效率较低。

>可能有读者会认为集中式负载均衡存在这样的问题，一旦负载均衡服务挂掉，那整个系统将不能使用。
解决方案：可以对负载均衡服务进行DNS负载均衡，通过对一个域名设置多个IP地址，每次DNS解析时轮询返回负载均衡服务地址，从而实现简单的DNS负载均衡。

- 客户端负载

![](images/grpc客户端负载均衡.png)

针对第一个方案的不足，此方法将lb的功能集成到服务消费方进程里，也被称为软负载或者客户端负载方案。 服务提供方启动时，首先将服务地址注册到服务注册表，同时定期报心跳到服务注册表以表明服务的存活状态，相当于健康检查，服务消费方要访问某个服务时，它通过内置的lb组件向服务注册表查询，同时缓存并定期刷新目标服务地址列表，然后以某种负载均衡策略选择一个目标服务地址，最后向目标服务发起请求。 lb和服务发现能力被分散到每一个服务消费者的进程内部，同时服务消费方和服务提供方之间是直接调用，没有额外开销，性能比较好。

该方案主要问题： 要用多种语言、多个版本的客户端编写和维护负载均衡策略，使客户端的代码大大复杂化。

- 独立lb服务

![](images/grpc独立lb服务负载均衡.png)

该方案是针对第二种方案的不足而提出的一种折中方案，原理与第二种方案基本类似。

不同之处是将LB和服务发现功能从进程内移出来，变成主机上的一个独立进程。主机上的一个或者多个服务要访问目标服务时，他们都通过同一主机上的独立LB进程做服务发现和负载均衡。该方案也是一种分布式方案没有单点问题，服务调用方和LB之间是进程内调用性能好，同时该方案还简化了服务调用方，不需要为不同语言开发客户库。


# 实现gRPC客户端负载均衡

gRPC已提供了简单的负载均衡策略（如：Round Robin），我们只需实现它提供的Builder和Resolver接口，就能完成gRPC客户端负载均衡。

```golang
type Builder interface {
	Build(target Target, cc ClientConn, opts BuildOption) (Resolver, error)
	Scheme() string
}
```
Builder接口：创建一个resolver（本文称之服务发现），用于监视名称解析更新。
Build方法：为给定目标创建一个新的resolver，当调用grpc.Dial()时执行。
Scheme方法：返回此resolver支持的方案

```golang
type Resolver interface { 
    ResolveNow(ResolveNowOption) 
    Close() 
}
```
Resolver接口：监视指定目标的更新，包括地址更新和服务配置更新。
ResolveNow方法：被 gRPC 调用，以尝试再次解析目标名称。只用于提示，可忽略该方法。
Close方法：关闭resolver

根据以上两个接口，我们把服务发现的功能写在Build方法中，把获取到的负载均衡服务地址返回到客户端，并监视服务更新情况，以修改客户端连接。

客户端代码参考
```golang
package etcdv3

import (
	"context"
	"log"
	"sync"
	"time"

	"go.etcd.io/etcd/client/v3"
	"google.golang.org/grpc/resolver"
)

// ServiceDiscovery 服务发现
type ServiceDiscovery struct {
	cli        *clientv3.Client
	lock       sync.Mutex
	cc         resolver.ClientConn
	serverList map[string]resolver.Address //服务列表
}

// NewServiceDiscovery 发现服务
func NewServiceDiscovery(endpoint []string) *ServiceDiscovery {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   endpoint,
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatal(err)
	}
	return &ServiceDiscovery{
		cli: cli,
	}
}

//Build 为给定目标创建一个新的`resolver`，当调用`grpc.Dial()`时执行
func (s *ServiceDiscovery) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	s.cc = cc
	s.serverList = make(map[string]resolver.Address)
	prefix := target.Endpoint
	log.Println("Build ", prefix)
	//根据前缀获取现有的key
	resp, err := s.cli.Get(context.Background(), prefix, clientv3.WithPrefix())
	if err != nil {
		return nil, err
	}

	for _, ev := range resp.Kvs {
		s.SetServiceList(string(ev.Key), string(ev.Value))
	}

	s.cc.NewAddress(s.getServices())
	//监视前缀，修改变更的server
	go s.watcher(prefix)
	return s, nil
}

// ResolveNow 监视目标更新
func (s *ServiceDiscovery) ResolveNow(rn resolver.ResolveNowOptions) {
	log.Println("ResolveNow")
}

//Scheme return schema
func (s *ServiceDiscovery) Scheme() string {
	return "etcd"
}

// 监听前缀
func (s *ServiceDiscovery) watcher(prefix string) {
	rch := s.cli.Watch(context.Background(), prefix, clientv3.WithPrefix())
	log.Printf("watching prefix: %s now...", prefix)
	for wresp := range rch {
		for _, ev := range wresp.Events {
			switch ev.Type {
			case clientv3.EventTypePut:
				// 修改或新增
				s.SetServiceList(string(ev.Kv.Key), string(ev.Kv.Value))
			case clientv3.EventTypeDelete:
				// 删除
				s.DelServiceList(string(ev.Kv.Key))
			}
		}
	}
}

// SetServiceList 新增服务地址
func (s *ServiceDiscovery) SetServiceList(key, val string) {
	s.lock.Lock()
	defer s.lock.Unlock()
	s.serverList[key] = resolver.Address{Addr: val}
	s.cc.NewAddress(s.getServices())
	log.Println("put key: ", key, "val: ", val)
}

// DelServiceList 删除服务地址
func (s *ServiceDiscovery) DelServiceList(key string) {
	s.lock.Lock()
	defer s.lock.Unlock()
	delete(s.serverList, key)
	s.cc.NewAddress(s.getServices())
	log.Println("delete key: ", key)
}

//getServices 获取服务地址, 无锁，给Set/Del内部调用
func (s *ServiceDiscovery) getServices() []resolver.Address {
	addrs := make([]resolver.Address, 0, len(s.serverList))
	for _, v := range s.serverList {
		addrs = append(addrs, v)
	}
	return addrs
}

func (s *ServiceDiscovery) Close() {
	s.cli.Close()
}
```
