---
title: 基于redis的分布式锁实现
date: 2025-02-28T19:19:48+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - 架构
showTags: false
hideBackToTop: false
---
#### 分布式锁安全和失效保障
算法只需要具备3个特性就可以实现最低保障的分布式锁
1. 安全属性：独享，任意时刻只有一个客户端持有锁
2. 活性A：无死锁，即便持有锁客户端崩溃或网络分裂，锁仍然可以被获取
3. 活性B：容错，只要大部分redis节点存活，客户端即可获取和释放锁

#### 单redis实例实现
- 使用锁命令
	- NX 不存在key才执行
	- PX超时
	- random_value随机值，确保可以安全释放

```
SET name random_value NX PX 3000
```

- 释放锁

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

#### 单实例配合故障转移实现
在单实例实现中存在单点失败问题，如果增加salve形成主从结构，当主从存在延迟时会导致从并没同步到锁的状态导致安全失效

#### RedLock实现
假设有N个redis master节点，在这种算法情况下客户端获取锁
1. 获取当前unix时间戳，单位为毫秒
2. 依次尝试从N个实例使用相同的key和随机值获取锁，并且获取锁时设置一个网络超时时间，并且这个时间<锁失效时间
3. 客户端使用当前时间减去获取锁时刻时间得到获取锁耗时，只有从大多数节点获取到锁并且使用时间小于锁失效时间，锁才算获取成功
4. 锁的真正有效时间是有效时间减去获取锁所使用的时间
5. 如果获取锁失败，则应尽快在所有实例上释放锁

#### 代码实现
##### redsync.go

```
// redsync.go
package distributedlock

import (
	"math/rand"
	"time"
)

// 最小最大延迟重试时间
const (
	minRetryDelayMilliSec = 50
	maxRetryDelayMilliSec = 250
)

type RedSync struct {
	pools []*Pool
}

func New(pool ...*Pool) *RedSync {
	return &RedSync{pools: pool}
}

func (r *RedSync) NewMutex(name string, options ...MutexOption) *Mutex {
	m := &Mutex{
		name:   name,
		expiry: 8 * time.Second, // 8s过期
		tries:  32, // 最大重试32次
		delayFunc: func(tries int) time.Duration {
			return time.Duration(rand.Intn(maxRetryDelayMilliSec-minRetryDelayMilliSec)+minRetryDelayMilliSec) * time.Millisecond
		}, // 延迟数值计算函数
		genValueFunc:  genValue, // 随机值生成函数
		driftFactor:   0.01, // 到达过期时间漂移因子
		timeoutFactor: 0.05, // 获取锁超时时间漂移因子
		quorum:        len(r.pools)/2 + 1, // 成功获取到锁的节点数
		pools:         r.pools, // redis节点池
	}

	for _, o := range options {
		o(m)
	}

	return m
}
```

##### mutex.go

```
// mutex.go
package distributedlock

import (
	"context"
	"crypto/rand"
	"encoding/base64"
	"errors"
	"fmt"
	"github.com/hashicorp/go-multierror"
	"time"
)

type Mutex struct {
	expiry        time.Duration
	name          string
	timeoutFactor float64
	driftFactor   float64
	tries         int
	pools         []*Pool
	failFast      bool
	quorum        int
	value         string
	until         time.Time
	delayFunc     func(int) time.Duration
	genValueFunc  func() (string, error)
}

func (m *Mutex) Unlock() (bool, error) {
	return m.UnlockContext(context.Background())
}

func (m *Mutex) UnlockContext(ctx context.Context) (bool, error) {
	n, err := m.actOnPoolAsync(ctx, m.release, m.value)
	if n < m.quorum {
		return false, err
	}
	return true, nil
}

func (m *Mutex) Lock() error {
	return m.LockContext(context.Background())
}

func (m *Mutex) LockContext(ctx context.Context) error {
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	var timer *time.Timer
	for i := 0; i < m.tries; i++ {
		// 重试延迟
		if i != 0 {
			if timer == nil {
				timer = time.NewTimer(m.delayFunc(i))
			} else {
				timer.Reset(m.delayFunc(i))
			}
			select {
			case <-ctx.Done():
				timer.Stop()
				return ErrFailed
			case <-timer.C:

			}
		}

		start := time.Now()

		n, err := m.actOnPoolAsync(ctx, m.acquire, value)

		now := time.Now()
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.driftFactor)))
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			return nil
		}

		// 如果没有最终获取锁成功，快速释放掉已经获取的子锁
		m.actOnPoolAsync(ctx, m.release, value)

		// 达到最大尝试次数，并且有报错，直接返回
		if i == m.tries-1 && err != nil {
			return err
		}
	}

	return ErrFailed
}

// 获取锁
func (m *Mutex) acquire(ctx context.Context, pool *Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
	reply, err := conn.SetNX(m.name, value, m.expiry)
	if err != nil {
		return false, err
	}
	return reply, nil
}

var deleteScript = NewScript(1, `
	local val = redis.call("GET", KEYS[1])
	if val == ARGV[1] then
		return redis.call("DEL", KEYS[1])
	elseif val == false then
		return -1
	else
		return 0
	end
`)

// 释放锁，通过脚本执行
func (m *Mutex) release(ctx context.Context, pool *Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
	status, err := conn.Eval(deleteScript, m.name, value)
	if err != nil {
		return false, err
	}

	if status == int64(-1) {
		return false, ErrLockAlreadyExpired
	}

	return status != int64(0), nil
}

// 异步获取各节点的锁
func (m *Mutex) actOnPoolAsync(ctx context.Context, actFn func(context.Context, *Pool, string) (bool, error), value string) (int, error) {

	ctx, cancel := context.WithTimeout(ctx, time.Duration(int64(float64(m.expiry)*m.timeoutFactor)))
	defer cancel()

	type result struct {
		node     int
		statusOK bool
		err      error
	}
	ch := make(chan result, len(m.pools))
	for node, pool := range m.pools {
		go func(node int, pool *Pool) {
			r := result{node: node}
			r.statusOK, r.err = actFn(ctx, pool, value)
			ch <- r
		}(node, pool)
	}

	n := 0
	var err error
	taken := make([]int, 0)

	for range m.pools {
		r := <-ch
		if r.statusOK {
			n++
		} else if r.err == ErrLockAlreadyExpired {
			err = multierror.Append(err, ErrLockAlreadyExpired)
		} else if r.err != nil {
			err = multierror.Append(err, errors.New(fmt.Sprintf("redis error, node: %d err: %v", r.node, r.err)))
		} else {
			taken = append(taken, r.node)
			err = multierror.Append(err, errors.New(fmt.Sprintf("taken error, node: %d err: %v", r.node, r.err)))
		}

		if m.failFast {
			if n >= m.quorum {
				return n, err
			}

			if len(taken) >= m.quorum {
				return n, &ErrTaken{Nodes: taken}
			}
		}
	}

	if len(taken) >= m.quorum {
		return n, &ErrTaken{Nodes: taken}
	}

	return n, err
}

func genValue() (string, error) {
	b := make([]byte, 16)
	_, err := rand.Read(b)
	if err != nil {
		return "", err
	}

	return base64.StdEncoding.EncodeToString(b), nil
}

type MutexOption func(*Mutex)

func WithExpiry(expiry time.Duration) MutexOption {
	return func(m *Mutex) {
		m.expiry = expiry
	}
}
```

##### pool.go

```
// pool.go
package distributedlock

import (
	"context"
	"github.com/go-redis/redis"
)

type Pool struct {
	redisClient *redis.Client
}

func NewPool(redisClient *redis.Client) *Pool {
	return &Pool{redisClient: redisClient}
}

func (p *Pool) Get(ctx context.Context) (*Conn, error) {
	c := p.redisClient
	if ctx != nil {
		c = c.WithContext(ctx)
	}

	return &Conn{c}, nil
}
```

##### conn.go

```
// conn.go
package distributedlock

import (
	"crypto/sha1"
	"encoding/hex"
	"github.com/go-redis/redis"
	"io"
	"strings"
	"time"
)

type Conn struct {
	redisClient *redis.Client
}

func (c *Conn) Close() error {
	return nil
}

type Script struct {
	KeyCount int
	Src      string
	Hash     string
}

func NewScript(keyCount int, src string) *Script {
	h := sha1.New()
	_, _ = io.WriteString(h, src)
	return &Script{
		KeyCount: keyCount,
		Src:      src,
		Hash:     hex.EncodeToString(h.Sum(nil)),
	}
}

func (c *Conn) SetNX(name string, value string, expiry time.Duration) (bool, error) {
	ok, err := c.redisClient.SetNX(name, value, expiry).Result()
	if err != redis.Nil {
		return false, err
	}

	return ok, nil
}

// 执行脚本
func (c *Conn) Eval(script *Script, keysAndArgs ...interface{}) (interface{}, error) {
	keys := make([]string, script.KeyCount)
	args := keysAndArgs

	if script.KeyCount > 0 {
		for i := 0; i < script.KeyCount; i++ {
			keys[i] = keysAndArgs[i].(string)
		}
		args = keysAndArgs[script.KeyCount:]
	}
	v, err := c.redisClient.EvalSha(script.Hash, keys, args...).Result()
	if err != nil && strings.HasPrefix(err.Error(), "NOSCRIPT") {
		v, err = c.redisClient.Eval(script.Src, keys, args...).Result()
	}

	if err != nil {
		return false, err
	}

	return v, nil
}
```

##### error.go

```
// error.go
package distributedlock

import (
	"errors"
	"fmt"
)

var ErrFailed = errors.New("redsync: failed to acquire lock")

var ErrLockAlreadyExpired = errors.New("redsync: failed to unlock, lock already expired")

type ErrTaken struct {
	Nodes []int
}

func (e *ErrTaken) Error() string {
	return fmt.Sprintf("lock already taken, locked nodes: %v", e.Nodes)
}
```