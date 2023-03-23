---
title: redis实现分布式锁
date: 2023-03-20 16:02:49
tags:
---

# 互斥锁

redis里有一个set nx的命令，我们可以通过这个命令来实现互斥锁功能， 

<!--more-->

在redis官方文档里面推荐的标准实现方式是： 
set resource_name my_random_value nx px 30000 这串命令，其中:

- resource_name: 表示要锁定的资源
- NX 表示如果不存在则设置
- PX 30000 表示过期时间为30000毫秒，也就是30秒
- my_random_value 这个值在所有的客户端必须是唯一的，所有同一key的锁竞争者这个值都不能一样。

值必须是随机数主要是为了更安全的释放锁，释放锁的时候使用脚本告诉redis: 只有key存在并且存储的值和我指定的值一样才能告诉我删除成功，避免错误释放别的竞争者的锁。

由于涉及到两个操作，因此我们需要通过lua脚本保证操作的原子性:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
	return redis.call("del", KEY[1])
else
	return 0
end
```

举个不用lua脚本的例子： 客户端A取的资源锁，但是紧接着被一个操作阻塞了，当客户端A运行完毕其他操作后要释放锁时，原来的锁早已超时并且被redis自动释放，并且在这期间资源锁又被客户端B再次获取到。

因为判断和删除是两个操作，所以有可能A刚判断完锁就过期自动释放了，然后B就获取到了锁，然后A又调用Del，导致把B的锁给释放了。

## tryLock和unlock实现

tryLock其实就是使用set resource_name my_random_value nx px 30000 加锁，这里使用UUID作为随机值，并且在加锁成功时把随机值返回，这个随机值会在unlock时使用；
unlock解锁逻辑就是执行前面说到的lua脚本。

## lock 实现

Lock是阻塞的获取锁，因此在加锁失败的时候，需要重试。当然也可能出现其他异常情况（比如网络问题，请求超时等），这些情况则直接返回error。

步骤如下:
- 尝试加锁，加锁成功直接返回
- 加锁失败则不断循环尝试加锁直到成功或出现异常情况

## 实现看门狗机制

我们前面的例子中提到的互斥锁有一个小问题，就是如果持有锁客户端A被阻塞，那么A的锁可能会超时被自动释放，导致客户端B提前获取到锁。
为了减少这种情况的发生，我们可以在A持有锁期间，不断地延长锁的过期时间，减少客户端B提前获取到锁的情况，这就是看门狗机制。
当然，这没办法完全避免上述情况的发生，因为如果客户端A获取锁之后，刚好与Redis的连接关闭了，这时候也就没办法延长超时时间了。

加锁成功时启动一个线程，不断地延长锁的过期时间；在Unlock时关闭看门狗线程。

看门狗流程如下：

- 加锁成功，启动看门狗
- 看门狗线程不断延长锁的过程时间
- 解锁，关闭看门狗

## 代码示例

```golang

package redis

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8"
	"github.com/rs/xid"
)

var (
	ErrLockFailed = errors.New("get lock failed with max retry times")
	ErrTimeout    = errors.New("get lock failed with timeout")
)

type redisLock struct {
	resource        string
	randomValue     string
	ttl             time.Duration
	tryLockInterval time.Duration
	watchDog        chan struct{}
}

type OptionFunc func(*redisLock)

func WithWatchDog() OptionFunc {
	return func(l *redisLock) {
		l.watchDog = make(chan struct{})
	}
}

func WithTryLockInterval(t time.Duration) OptionFunc {
	return func(l *redisLock) {
		l.tryLockInterval = t
	}
}

func NewLock(key string, duration time.Duration, opts ...OptionFunc) *redisLock {
	l := &redisLock{
		resource: key,
		ttl:      duration,
	}
	for _, opt := range opts {
		opt(l)
	}
	return l

}

func (l *redisLock) TryLock(ctx context.Context) error {
	randomValue := xid.New().String()
	success, err := GetClient().SetNX(ctx, l.resource, randomValue, l.ttl)
	if err != nil {
		return err
	}
	// 加锁失败
	if !success {
		return ErrLockFailed
	}
	// 加锁成功
	l.randomValue = randomValue

	if l.watchDog != nil {
		go l.startWatchDog()
	}
	return nil
}

var unlockScript = redis.NewScript(`
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
`)

func (l *redisLock) Unlock(ctx context.Context) error {
	fmt.Println("Unlock")
	if l.watchDog != nil {
		close(l.watchDog)
	}
	return unlockScript.Run(context.Background(), GetClient().GetGoRedis(), []string{l.resource}, l.randomValue).Err()
}

func (l *redisLock) Lock(ctx context.Context) error {
	// 尝试加锁
	err := l.TryLock(ctx)
	if err == nil {
		return nil
	}
	if !errors.Is(err, ErrLockFailed) {
		return err
	}

	// 加锁失败，不断尝试
	if l.tryLockInterval == 0 {
		return err
	}

	ticker := time.NewTicker(l.tryLockInterval)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			// 超时
			return ErrTimeout
		case <-ticker.C:
			// 重新尝试加锁
			err := l.TryLock(ctx)
			if err == nil {
				return nil
			}
			if !errors.Is(err, ErrLockFailed) {
				return err
			}
		}
	}
}

func (l *redisLock) startWatchDog() {
	ticker := time.NewTicker(l.ttl / 3)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			// 延长锁的过期时间
			ctx, cancel := context.WithTimeout(context.Background(), l.ttl/3*2)
			fmt.Println("expire")
			ok, err := GetClient().Expire(ctx, l.resource, l.ttl)
			cancel()
			// 异常或锁已经不存在则不再续期
			if err != nil || !ok {
				return
			}
		case <-l.watchDog:
			fmt.Println("done")

			// 已经解锁
			return
		}
	}
}
```
