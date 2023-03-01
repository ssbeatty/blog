---
title: 通过redsync学习redlock
date: 2022-09-18T00:22:05+08:00
lastmod: 2022-09-18T00:22:05+08:00
author: sasaba
cover: /img/通过redsync学习redlock.png
images:
  - /img/通过redsync学习redlock.png
categories:
  - 分布式
tags:
  - Redis
draft: true
---

通过redsync学习redis分布式锁。

<!--more-->

## 资料
[redis官网分布式锁](https://redis.io/docs/reference/patterns/distributed-locks/)
[redsync](https://github.com/go-redsync/redsync)


## 什么是分布式锁

- **互斥性**: 任意时刻，只有一个客户端能持有锁。
- **锁超时释放**：持有锁超时，可以释放，防止不必要的资源浪费，也可以防止死锁。
- **可重入性**:一个线程如果获取了锁之后,可以再次对其请求加锁。
- **高性能和高可用**：加锁和解锁需要开销尽可能低，同时也要保证高可用，避免分布式锁失效。
- **安全性**：锁只能被持有的客户端删除，不能被其他客户端删除

## 相关概念
- 1.TTL：Time To Live;只 redis key 的过期时间或有效生存时间

- 2.clock drift:时钟漂移；指两个电脑间时间流速基本相同的情况下，两个电脑（或两个进程间）时间的差值；如果电脑距离过远会造成时钟漂移值过大

[关于漂移因子的设定](https://github.com/mike-marcacci/node-redlock/issues/17)

## 实现

### 初级方案
在网上搜到的方案里一般都会提到使用使用SET的扩展命令（SET EX PX NX）来实现
```shell
SET resource_name lock_value NX PX 30000
```
> NX :表示key不存在的时候，才能set成功，也即保证只有第一个客户端请求才能获得锁，而其他客户端请求只能等其释放锁，才能获取。
> PX milliseconds: 设定key的过期时间，单位为毫秒

这种方式不仅可以保证命令的原子性，同时NX保证同一个时间只能有一个客户端获取到锁，设置TTL可以保证锁自动释放。

但是这种方法也存在如下问题：
1. 锁过期释放掉了但是业务并没有处理完。
2. 锁被其他的客户端获取操作，比如客户端a去释放锁时可能当前锁的时间已经到了，此时b客户端获取到锁后被a释放。


### 单Redis实例的正确方案

为了解决当前锁被其他客户端操作，可以给锁设置一个随机字符的值，上述命令升级为：
```shell
SET resource_name my_random_value NX PX 30000
```
这个随机字符可以是什么？可以看redsync的实现：

```go
func genValue() (string, error) {
	b := make([]byte, 16)
	_, err := rand.Read(b)
	if err != nil {
		return "", err
	}
	return base64.StdEncoding.EncodeToString(b), nil
}
```
随机值用于以安全的方式释放锁，redsync中的实现如下:
```go
var deleteScript = redis.NewScript(1, `
	if redis.call("GET", KEYS[1]) == ARGV[1] then
		return redis.call("DEL", KEYS[1])
	else
		return 0
	end
`)

func (m *Mutex) release(ctx context.Context, pool redis.Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
	status, err := conn.Eval(deleteScript, m.name, value)
	if err != nil {
		return false, err
	}
	return status != int64(0), nil
}
```
这样可以保证客户端自己生成的锁一定是被自己释放，使用lua脚本可以保证命令的一致性。
这样的话对于单机的redis实例就可以保证分布式锁的可用。


### RedLock
上文中单Redis实例的方案对于大多数项目已经可用，但是目前的大型项目中，为了保证高服务的可用性，Redis的实例基本上都不可能是单机的，
无论是哨兵还是主从或者集群方案，上述的锁方案都会存在以下问题：
1. 客户端 A 获取主控中的锁。
2. 主服务器在将Key的写入传输到副本之前崩溃。
3. 副本被提升为主。
4. 客户端 B 获得对同一个资源 A 已经持有锁的锁。

这样就需要RedLock算法，RedLock算法是Redis社区提出的分布式锁算法。为了获取锁，客户端执行以下操作：

1. 以毫秒为单位获取当前时间。
2. 尝试顺序获取所有 N 个实例中的锁，在所有实例中使用相同的键名和随机值。
3. 客户端通过从当前时间中减去步骤 1 中获得的时间戳来计算获取锁所用的时间。当且仅当客户端能够在大多数实例（至少 3 个）中获取锁时，且获取锁的总时间小于锁的有效时间，则认为锁已被获取。
4. 如果获得了锁，则其有效时间被认为是初始有效时间减去经过的时间，如步骤 3 中计算的那样。
5. 如果客户端由于某种原因未能获得锁（它无法锁定 N/2+1 个实例或有效时间为负数），它将尝试解锁所有实例（即使是它认为没有的实例）能够锁定）。

> 在步骤 2 中，当在每个实例中设置锁时，客户端需要使用一个比锁释放时间小的多的超时时间来获取它。
> 例如，如果自动释放时间为 10 秒，则超时可能在 ~ 5-50 毫秒范围内。这可以防止客户端在尝试与已关闭的 Redis 节点通信时长时间保持阻塞：如果一个实例不可用，我们应该尽快尝试与下一个实例通信。

![](/context_img/redlock_imp.png)

redsync 实现如下：
```go
func (m *Mutex) Lock() error {
	return m.LockContext(nil)
}

// LockContext locks m. In case it returns an error on failure, you may retry to acquire the lock by calling this method again.
func (m *Mutex) LockContext(ctx context.Context) error {
	if ctx == nil {
		ctx = context.Background()
	}
    // 获取一个随机值 
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	for i := 0; i < m.tries; i++ {
        // 重试逻辑
		if i != 0 {
			select {
			case <-ctx.Done():
				// Exit early if the context is done.
				return ErrFailed
			case <-time.After(m.delayFunc(i)):
				// Fall-through when the delay timer completes.
			}
		}
        // 1. 获取当前时间
		start := time.Now()
        // 2. 尝试获取所有的节点的锁
		n, err := func() (int, error) {
			ctx, cancel := context.WithTimeout(ctx, time.Duration(int64(float64(m.expiry)*m.timeoutFactor)))
			defer cancel()
			return m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
				return m.acquire(ctx, pool, value)
			})
		}()
		if n == 0 && err != nil {
			return err
		}
        // 3. 实际的锁时间 = 使用锁的过期时间 - 获取所有节点需要的时间 - 时钟偏移时间
		// 时钟偏移时间 = 锁的过期时间 * 漂移系数(默认0.01)
		now := time.Now()
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.driftFactor)))
		// 4. 如果获取到的节点数 > N/2+1节点数 且当前时间仍然小于实际的锁时间则获取成功
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			return nil
		}
		// 5. 如果获取锁失败则释放所有已经获取的锁
		// release 方法可见单Redis实例的释放方法
		_, err = func() (int, error) {
			ctx, cancel := context.WithTimeout(ctx, time.Duration(int64(float64(m.expiry)*m.timeoutFactor)))
			defer cancel()
			return m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
				return m.release(ctx, pool, value)
			})
		}()
		if i == m.tries-1 && err != nil {
			return err
		}
	}

	return ErrFailed
}

// Unlock unlocks m and returns the status of unlock.
func (m *Mutex) Unlock() (bool, error) {
	return m.UnlockContext(nil)
}

// UnlockContext unlocks m and returns the status of unlock.
func (m *Mutex) UnlockContext(ctx context.Context) (bool, error) {
	n, err := m.actOnPoolsAsync(func(pool redis.Pool) (bool, error) {
		return m.release(ctx, pool, m.value)
	})
	if n < m.quorum {
		return false, err
	}
	return true, nil
}
```

### RedLock算法是否是异步算法
可以看成是同步算法；因为 即使进程间（多个电脑间）没有同步时钟，但是每个进程时间流速大致相同；并且时钟漂移相对于TTL叫小，可以忽略，所以可以看成同步算法；（不够严谨，算法上要算上时钟漂移，因为如果两个电脑在地球两端，则时钟漂移非常大）


## 其它问题
1. 关于上述初级方案中提到的锁过期释放掉了但是业务并没有处理完的问题，java中通常使用Redisson框架解决

![](/context_img/redis_session.jpg)

只要线程一加锁成功，就会启动一个watch dog看门狗，它是一个后台线程，会每隔10秒检查一下，如果线程1还持有锁，那么就会不断的延长锁key的生存时间。因此，Redisson就是使用Redisson解决了锁过期释放，业务没执行完问题。