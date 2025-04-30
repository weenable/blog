---
title: P2P核心技术：Kad分布式路由表算法
date: 2025-02-28T18:51:48+08:00
toc: true
readTime: true
autonumber: true
math: true
categories:
  - web3
showTags: false
hideBackToTop: false
---
### 简介
Kademlia（Kad）是分布式散列表（DHT）算法的一种，是去中心化P2P网络最核心的一种路由寻址技术，可以在无中心服务器（trackerless）的情况下，在网络中快速找到目标节点

早期中心化服务器BtTorrent网络，需要种子服务器来帮助节点之间发现
![image-20252285241628.png](/images/P2P%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%EF%BC%9AKad%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%AF%E7%94%B1%E8%A1%A8%E7%AE%97%E6%B3%95/image-20252285241628.png)

实现Kad协议的P2P网络，每个节点维护一个路由表，仅记录离自己最近的一些节点信息，通过迭代查询来发现其他节点

![image-2025228534197.png](/images/P2P%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%EF%BC%9AKad%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%AF%E7%94%B1%E8%A1%A8%E7%AE%97%E6%B3%95/image-2025228534197.png)
### 核心内容

- Node ID：P2P网络中，节点通过唯一的ID来进行标识，在原始Kad算法中，使用160bit哈希空间来作为Node ID
- Node Distance：每个节点保存自己附近的节点信息，是通过计算得到的逻辑距离来判断的（通过把两个节点的Node ID进行XOR运算，结果越小距离越近）
- K-Bucket：用一个Bucket来保存与当前节点距离在某个范围内的所有节点列表
- Bucket分裂：如果原始Bucket数量不够，需要进行分裂
- Routing Table：记录所有Bucket，每个bucket限制最多k个节点，如下图所示

![image-20252285349459.png](/images/P2P%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%EF%BC%9AKad%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%AF%E7%94%B1%E8%A1%A8%E7%AE%97%E6%B3%95/image-20252285349459.png)

- Update：在节点bootstrap时，需要把连接上的节点更新到自己的routing table中
- LookUp：查找节点，找到与目标节点最近的bucket，如果目标节点在bucket中则直接范围，否则往bucket中节点发送查询请求，这些节点继续迭代查询

### 详细内容

##### 1.Node ID
Kad使用SHA1哈希来计算Node ID，SHA1是一个160bit（20字节）的哈希空间
IPFS使用SHA256哈希来计算Node ID，256bit（32字节）的哈希空间
eth使用SHA3，也是256bit哈希空间

##### 2.Node Distance和XOR
对两个Node ID进行XOR运算，可以得出他们之间的距离
Kad中，根据当前节点和其他节点的Node ID匹配的最多的bit个数来构建一棵二叉树，这里匹配的bit数也叫LCP(longest common prefix)，按照LCP来划分子树

![image-20252285458639.png](/images/P2P%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%EF%BC%9AKad%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%AF%E7%94%B1%E8%A1%A8%E7%AE%97%E6%B3%95/image-20252285458639.png)

对于160bit空间的Node ID来说，一共会有160颗子树，也就是160个bucket
Kad要求每个节点知道其各子树的至少一个节点