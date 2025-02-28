---
title: 实现redis客户端一致性哈希分片
date: 2025-02-28T19:03:16+08:00
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
哈希方案有以下几种：
- 普通哈希分片    
- 一致性哈希分片  
- 范围哈希分片

分片有以下几种方案：
- redis官方哈希槽分片方案（属于服务端sharding，使用范围哈希）
- 客户端sharding，可以使用普通哈希、一致性哈希
- 代理sharding，使用代理器进行分片，有性能损耗

##### 客户端通过一致性哈希实现分片
```
package main

import (
	"context"
	"github.com/go-redis/redis/v8"
	"github.com/stathat/consistent"
)

// 初始化支持一致性哈希的客户端结构
type ShardingClient struct {
	consistentHash *consistent.Consistent
	clients        map[string]*redis.Client
}

func NewShardingClient(addrs []string) *ShardingClient {
	ch := consistent.New()
	clients := make(map[string]*redis.Client)

	for _, addr := range addrs {
		ch.Add(addr)
		clients[addr] = redis.NewClient(&redis.Options{
			Addr: addr,
		})
	}

	return &ShardingClient{
		consistentHash: ch,
		clients:        clients,
	}
}

// getClient 根据键获取相应的 Redis 客户端
func (sc *ShardingClient) getClient(key string) (*redis.Client, error) {
	addr, err := sc.consistentHash.Get(key)
	if err != nil {
		return nil, err
	}
	return sc.clients[addr], nil
}

// Set 在相应的 Redis 实例上设置键值对
func (sc *ShardingClient) Set(ctx context.Context, key, value string) error {
	client, err := sc.getClient(key)
	if err != nil {
		return err
	}
	return client.Set(ctx, key, value, 0).Err()
}

// Get 在相应的 Redis 实例上获取键值对
func (sc *ShardingClient) Get(ctx context.Context, key string) (string, error) {
	client, err := sc.getClient(key)
	if err != nil {
		return "", err
	}
	return client.Get(ctx, key).Result()
}


func main() {
	addrs := []string{
		"127.0.0.1:6379",
		"127.0.0.1:6380",
		"127.0.0.1:6381",
	}

	// 初始化 Sharding 客户端
	shardingClient := NewShardingClient(addrs)

	ctx := context.Background()

	// 测试连接和操作
	err := shardingClient.Set(ctx, "key", "value")
	if err != nil {
		log.Fatalf("Failed to set key: %v", err)
	}

	val, err := shardingClient.Get(ctx, "key")
	if err != nil {
		log.Fatalf("Failed to get key: %v", err)
	}

	fmt.Printf("key: %s\n", val)
}
```