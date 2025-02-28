---
title: go-redis哨兵模式主从切换源码分析
date: 2025-02-28T19:04:12+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - golang
showTags: false
hideBackToTop: false
---
##### 实现原理
1. 通过哨兵获取主节点信息：在连接 Redis 集群时，`go-redis` 客户端会连接到一个或多个哨兵节点，获取当前的主节点信息。如果主节点发生故障，哨兵会进行故障转移，并通知客户端新的主节点信息。
2. 周期性地从哨兵获取主节点信息：`go-redis` 客户端会周期性地向哨兵节点请求主节点信息，以确保它始终连接到当前的主节点。这个机制可以帮助客户端快速感知主从切换。
3. 在操作失败时重新获取主节点信息：如果客户端在执行操作时遇到连接错误或其他错误，可能会重新从哨兵节点获取主节点的信息，并重试操作。

##### 源码分析
- 初始化
```
//rdb := redis.NewFailoverClient(&redis.FailoverOptions{
//    MasterName:    "mymaster", // 哨兵配置中主节点的名字
//    SentinelAddrs: []string{"127.0.0.1:26379", "127.0.0.1:26380", "127.0.0.1:26381"},
//})


type FailoverOptions struct {
    MasterName    string
    SentinelAddrs []string
    // ... other options
}

func NewFailoverClient(opt *FailoverOptions) *Client {
    sentinel := newSentinel(opt)
    return NewClient(&Options{
        Addr: sentinel.masterAddr(),
        // ... other options
    })
}

func newSentinel(opt *FailoverOptions) *sentinel {
    // Connect to sentinel nodes and get master address
    return &sentinel{
        masterName: opt.MasterName,
        addrs:      opt.SentinelAddrs,
    }
}

func (s *sentinel) masterAddr() string {
    // Get master address from sentinel nodes
}
```

- 周期性刷新master地址
```
func (s *sentinel) periodicallyRefreshMasterAddr() {
    ticker := time.NewTicker(time.Minute)
    for range ticker.C {
        addr := s.masterAddr()
        // Update client's master address
    }
}
```

- 失败时重试和刷新master
```
func (c *Client) doWithRetry(fn func() error) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if isConnectionError(err) {
            c.refreshMasterAddr()
        }
        time.Sleep(retryBackoff)
    }
    return fmt.Errorf("after %d retries, last error: %v", maxRetries, err)
}

func (c *Client) refreshMasterAddr() {
    addr := c.sentinel.masterAddr()
    c.Options().Addr = addr
}
```