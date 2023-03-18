---
title: etcd分布式锁
date: 2023-03-18 08:54:02
tags:
---

当并发的访问共享资源的时候，如果没有加锁的话，无法保证共享资源安全性和正确性。这个时候就需要用到锁

<!--more-->

## 1. 需要具备的特性

1. 需要保证互斥访问（分布式环境需要保证不同节点，不同线程的互斥访问）
2. 需要有超时机制，防止锁意外未释放，其他节点无法获取到锁；也要保证任务能够正常执行完成，不能超时了任务还没结束，导致任务执行一直被释放锁
3. 需要有阻塞和非阻塞两种请求锁的接口

## 2. 本地锁

当业务执行在同一个线程内，也就是我初始化一个本地锁，其他请求也认这把锁，一般是服务部署在单机环境上。 
我们可以看下下面的例子，开1000个goroutine并发的给counter做自增操作，结果会是什么样的呢?

```golang
package main

import (
	"fmt"
	"sync"
)

var cnt int64

func incr() {
	cnt++
}

func main() {
	wg := sync.WaitGroup{}
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			incr()
		}()
	}
	wg.Wait()
	fmt.Println(cnt)
}
```

结果是 count 的数量并不是预想中的 1000，而是下面这样，每次打印出的结果都不一样，但是接近 1000

```bash
[root@localhost 0307]# go run lock.go 
9525
[root@localhost 0307]# go run lock.go 
9693
[root@localhost 0307]# go run lock.go 
9795
[root@localhost 0307]# go run lock.go 
9143
```

出现这个问题的原因就是没有给自增操作加锁

下面我们修改代码如下，在 Incr 中加上 go 的 mutex 互斥锁

```golang
package main

import (
	"fmt"
	"sync"
)

var (
	cnt int64
	mu  sync.Mutex
)

func incr() {
	mu.Lock()
	cnt++
	mu.Unlock()
}

func main() {
	wg := sync.WaitGroup{}
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			incr()
		}()
	}
	wg.Wait()
	fmt.Println(cnt)
}
```

## 3. etcd实现互斥访问

```
package main

import (
	"fmt"
	"sync"

	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/client/v3/concurrency"
)

var (
	cnt int64
	wg  sync.WaitGroup
)

func incr() {
	cnt++
}

func main() {
	endpoints := []string{"http://127.0.0.1:2379"}
	// 初始化etcd客户端
	client, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		fmt.Println(err)
		return
	}
	defer client.Close()

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()

			session, err := concurrency.NewSession(client)
			if err != nil {
				panic(err)
			}
			defer session.Close()
			locker := concurrency.NewLocker(session, "/my-test-lock")
			locker.Lock()
			incr()
			locker.Unlock()

		}()
	}
	wg.Wait()
	fmt.Println(cnt)
}
```

- 实现超时机制

当某个客户端持有锁时，由于某些原因导致锁未释放，就会导致这个客户端一直持有这把锁，其他客户端一直获取不到锁。所以需要分布式锁实现超时机制，当锁未释放时，会因为 etcd 的租约会到期而释放锁。当业务正常处理时，租约到期之前会继续续约，知道业务处理完毕释放锁。

- 实现阻塞和非阻塞接口

上面的例子中已经实现了阻塞接口，即当前有获取到锁的请求，则其他请求阻塞等待锁释放

非阻塞的方式就是尝试获取锁，如果失败立即返回。etcd 中是实现了 tryLock 方法

```golang
// TryLock locks the mutex if not already locked by another session.
// If lock is held by another session, return immediately after attempting necessary cleanup
// The ctx argument is used for the sending/receiving Txn RPC.
func (m *Mutex) TryLock(ctx context.Context) error {
```

## 优缺点

etcd安全性高，性能低，可以做单点服务控制
