---
title: etcd实现服务注册与发现
date: 2023-03-17 09:06:20
tags:
---

## 服务发现介绍

服务发现要解决的也是分布式系统中最常见的问题之一，即在同一个分布式集群中的进程或服务，要如何才能找到对方并建立连接。本质上来说，服务发现就是要了解集群中是否有进程在监听udp或tcp端口，并且通过名字就可以查找和连接。

<!--more-->

![](/images/etcd服务发现介绍.png)

服务发现需要实现一个基本功能：

- 服务注册： 同一service的所有节点注册到相同目录下，节点启动后将自己的信息注册到所属服务的目录中
- 健康检查： 服务节点定时进行健康检查。 注册到服务目录中的信息设置一个较短的ttl，运行正常的服务节点每隔一段时间去更新信息的ttl，从而达到健康检查效果
- 服务发现： 通过服务节点能查询到服务提供外部访问的ip和端口号，比如网关代理服务是能够及时的发现服务中新增节点、丢弃不可用的服务节点。

## 服务注册与健康检查

根据etcd的v3 api，当启动一个服务的时候，我们把服务的地址写进etcd，注册服务。 同时绑定租约（lease），并以续租约（keep leases alive）的方式检测服务是否正常运行，从而实现健康检查。


## 服务注册代码

```golang
package etcdv3

import (
	"context"
	"log"
	"time"

	"go.etcd.io/etcd/client/v3"
)

// ServiceRegister 注册租约服务
type ServiceRegister struct {
	cli           *clientv3.Client
	leaseID       clientv3.LeaseID
	keepAliveChan <-chan *clientv3.LeaseKeepAliveResponse
	key           string
	val           string
}

func NewServiceRegister(endpoints []string, serNamePrefix, addr string, lease int64) (*ServiceRegister, error) {
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints, DialTimeout: 5 * time.Second})
	if err != nil {
		return nil, err
	}

	ser := &ServiceRegister{
		cli: cli,
		key: serNamePrefix + "/" + addr,
		val: addr,
	}

	if err = ser.putKeyWithLease(lease); err != nil {
		return nil, err
	}

	return ser, nil
}

// 续租
func (s *ServiceRegister) putKeyWithLease(lease int64) error {
	// 设置租约时间
	resp, err := s.cli.Grant(context.Background(), lease)
	if err != nil {
		return err
	}

	// 注册服务并绑定租约
	_, err = s.cli.Put(context.Background(), s.key, s.val, clientv3.WithLease(resp.ID))
	if err != nil {
		return err
	}

	// 设置续租，定期发送心跳请求
	leaseRespChan, err := s.cli.KeepAlive(context.Background(), resp.ID)
	if err != nil {
		return err
	}

	s.leaseID = resp.ID
	log.Println(s.leaseID)
	s.keepAliveChan = leaseRespChan
	log.Printf("Put key: %s val: %s success !", s.key, s.val)

	return nil
}

// listenLeaseRespChan 监听续租
func (s *ServiceRegister) ListenLeaseRespChan() {
	for leaseKeepResp := range s.keepAliveChan {
		log.Println("续租成功", leaseKeepResp)
	}

	log.Println("关闭续租")
}

// close 注销服务
func (s *ServiceRegister) Close() error {
	if _, err := s.cli.Revoke(context.Background(), s.leaseID); err != nil {
		return err
	}
	log.Println("撤销续租")
	return nil
}
```

## 服务发现代码

```golang
package etcdv3

import (
	"context"
	"log"
	"sync"
	"time"

	"go.etcd.io/etcd/client/v3"
)

// ServiceDiscovery 服务发现
type ServiceDiscovery struct {
	cli        *clientv3.Client
	serverList map[string]string // 服务列表
	lock       sync.Mutex
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
		cli:        cli,
		serverList: make(map[string]string),
	}
}

// WatchService 初始化服务列表与监视
func (s *ServiceDiscovery) WatchService(prefix string) error {
	// 根据前缀获取现有的key
	resp, err := s.cli.Get(context.Background(), prefix, clientv3.WithPrefix())
	if err != nil {
		return err
	}

	for _, ev := range resp.Kvs {
		s.SetServerList(string(ev.Key), string(ev.Value))
	}

	// 监听后续操作
	go s.watcher(prefix)

	return nil
}

// SetServerList 新增服务地址
func (s *ServiceDiscovery) SetServerList(key, val string) {
	s.lock.Lock()
	defer s.lock.Unlock()
	s.serverList[key] = val
	log.Println("put key: ", key, "val: ", val)
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
				s.SetServerList(string(ev.Kv.Key), string(ev.Kv.Value))
			case clientv3.EventTypeDelete:
				// 删除
				s.DelServerList(string(ev.Kv.Key))
			}
		}
	}
}

// DelServerList 删除服务地址
func (s *ServiceDiscovery) DelServerList(key string) {
	s.lock.Lock()
	defer s.lock.Unlock()
	delete(s.serverList, key)
	log.Println("delete key: ", key)
}

// GetServices 获取服务地址
func (s *ServiceDiscovery) GetServices() []string {
	s.lock.Lock()
	defer s.lock.Unlock()
	addrs := make([]string, 0)
	for _, v := range s.serverList {
		addrs = append(addrs, v)
	}
	return addrs
}

func (s *ServiceDiscovery) Close() {
	s.cli.Close()
}
```