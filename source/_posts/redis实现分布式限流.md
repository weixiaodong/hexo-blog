---
title: redis实现分布式限流
date: 2023-03-19 23:43:06
tags:
---

限流是项目中经常需要用到的一种工具，一般用于限制用户的请求频率，也可以避免流量过大导致的系统崩溃，或者稳定消息处理速率。 并且有时候我们还需要使用到分布式限流，常见的实现方式是使用redis作为中心存储。

<!--more-->

# 固定窗口（计数器方式）

使用redis实现固定窗口比较简单，主要是由于固定窗口同时只会存在一个窗口，所以我们可以在第一次进入窗口时使用pexpire命令设置过期时间为窗口时间大小，这样窗口会随过期时间而失效，同时我们使用incr命令增加窗口计数。

因为我们需要在counter == 1的时候设置窗口的过期时间，为了保证原子性，我们使用简单的lua脚本实现
```golang
package ratelimit

import (
	"context"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8"
)

const fixedWindowLimiterTryAcquireRedisScript = `
-- ARGV[1]: 窗口时间大小
-- ARGV[2]: 窗口请求上限

local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])

-- 获取原始值
local counter = tonumber(redis.call("get", KEYS[1]))

if counter == nil then
	counter = 0
end

-- 若到达窗口请求上限，请求失败
if counter >= limit then
	return 0
end

redis.call("incr", KEYS[1])

if counter == 0 then
	redis.call("pexpire", KEYS[1], window)
end
return 1
`

type FixedWindowLimiter struct {
	limit  int           // 窗口请求上限
	window int           // 窗口时间大小
	client *redis.Client // Redis客户端
	script *redis.Script // TryAcquire脚本
}

func newFixedWindowLimiter(client *redis.Client) *FixedWindowLimiter {
	// // redis过期时间精度最大到毫秒，因此窗口必须能被毫秒整除
	// if window%time.Millisecond != 0 {
	// 	return nil, errors.New("the window uint must not be less than millisecond")
	// }
	// limit 和 window 从配置中心获取
	limit := 2
	window := time.Second
	return &FixedWindowLimiter{
		limit:  limit,
		window: int(window / time.Millisecond),
		client: client,
		script: redis.NewScript(fixedWindowLimiterTryAcquireRedisScript),
	}
}

func (l *FixedWindowLimiter) Limit(ctx context.Context, resource string) bool {
	success, err := l.script.Run(ctx, l.client, []string{resource}, l.window, l.limit).Bool()
	if err != nil {
		fmt.Println("redis script run error: ", err)
		return false
	}
	// 若到达窗口请求上限，请求失败
	if !success {
		return true
	}
	return false
}
```

# 滑动窗口

设计思路：核心是利用list队列左进右出，个数占位推进代替时间推进（空间代替时间推进的转换）

三个参数, key:限流的Api存储key值，count：限流上限个数，windowTime:滑动窗口时间

1. 判断list队列长度是否超过上限count
2. 没超过上限，直接放行，把当前时间戳(秒)放进去队列
3. 超过上限，判断队列最右边占位的时间戳和当前时间戳的差值是否大于windowTime
4. 小于窗口时间，说明在窗口时间内达到上限，限流不放行
5. 大于窗口时间，说明已推进到新窗口，移除最右边的并且放入当前时间戳到最左边，放行

```golang
package ratelimit

import (
	"context"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8"
)

const slidingWindowLimiterTryAcquireRedisScript = `
local key = KEYS[1]
local count = ARGV[1]
local windowTime = ARGV[2]
local time = ARGV[3]
local len = redis.call('llen', key)
if tonumber(len) < tonumber(count) then
	redis.call('lpush', key, time)
	return 1
end

local earlyTime = redis.call('lindex', key, tonumber(len) - 1)
if tonumber(time) - tonumber(earlyTime) < tonumber(windowTime) then
	return 0
end

redis.call('rpop', key)
redis.call('lpush', key, time)
return 1
`

type slidingWindowLimiter struct {
	limit  int           // 窗口请求上限
	window int           // 窗口时间大小
	client *redis.Client // Redis客户端
	script *redis.Script // TryAcquire脚本
}

func newSlidingWindowLimiter(client *redis.Client) *slidingWindowLimiter {
	// // redis过期时间精度最大到毫秒，因此窗口必须能被毫秒整除
	// if window%time.Millisecond != 0 {
	// 	return nil, errors.New("the window uint must not be less than millisecond")
	// }

	limit := 2
	window := 1
	return &slidingWindowLimiter{
		limit:  limit,
		window: window,
		client: client,
		script: redis.NewScript(slidingWindowLimiterTryAcquireRedisScript),
	}
}

func (l *slidingWindowLimiter) Limit(ctx context.Context, resource string) bool {
	now := time.Now().Unix()

	success, err := l.script.Run(ctx, l.client, []string{resource}, l.limit, l.window, now).Bool()
	if err != nil {
		fmt.Println("redis script run error: ", err)
		return false
	}
	// 若到达窗口请求上限，请求失败
	if !success {
		return true
	}
	return false
}
```

go 官方redis限流实现参考地址：

https://github.com/go-redis/redis_rate

```golang
var allowN = redis.NewScript(`
-- this script has side-effects, so it requires replicate commands mode
redis.replicate_commands()

local rate_limit_key = KEYS[1]
local burst = ARGV[1]
local rate = ARGV[2]
local period = ARGV[3]
local cost = tonumber(ARGV[4])

local emission_interval = period / rate
local increment = emission_interval * cost
local burst_offset = emission_interval * burst

-- redis returns time as an array containing two integers: seconds of the epoch
-- time (10 digits) and microseconds (6 digits). for convenience we need to
-- convert them to a floating point number. the resulting number is 16 digits,
-- bordering on the limits of a 64-bit double-precision floating point number.
-- adjust the epoch to be relative to Jan 1, 2017 00:00:00 GMT to avoid floating
-- point problems. this approach is good until "now" is 2,483,228,799 (Wed, 09
-- Sep 2048 01:46:39 GMT), when the adjusted value is 16 digits.
local jan_1_2017 = 1483228800
local now = redis.call("TIME")
now = (now[1] - jan_1_2017) + (now[2] / 1000000)

local tat = redis.call("GET", rate_limit_key)

if not tat then
  tat = now
else
  tat = tonumber(tat)
end

tat = math.max(tat, now)

local new_tat = tat + increment
local allow_at = new_tat - burst_offset

local diff = now - allow_at
local remaining = diff / emission_interval

if remaining < 0 then
  local reset_after = tat - now
  local retry_after = diff * -1
  return {
    0, -- allowed
    0, -- remaining
    tostring(retry_after),
    tostring(reset_after),
  }
end

local reset_after = new_tat - now
if reset_after > 0 then
  redis.call("SET", rate_limit_key, new_tat, "EX", math.ceil(reset_after))
end
local retry_after = -1
return {cost, remaining, tostring(retry_after), tostring(reset_after)}
`)
```