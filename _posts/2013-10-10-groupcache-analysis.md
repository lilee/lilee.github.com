---
layout: post
title: "groupcache剖析"
description: ""
category: "海量之道"
tags: ["分布式", "缓存"]
---

在上一篇文章[groupcache简介](2013-10-10-groupcache)中介绍了groupcache的基本概念，和memcached的对比，以及如何使用groupcache。
这篇文章将对groupcache的设计和实现进行较为深入的分析。将从以下几个方面介绍

1. 基本设计思想
2. 运作机制
3. 源码分析
4. 特性 - 自动备份热点数据

## 基本设计思想

groupcache的设计主要满足静态资源的缓存。你可以将他看做是一个分布式的LRU缓存库。

groupcache的设计出人意表，它并不像memcached一样设计成一个可以独立运行的服务(not standalone server)。
而是设计成一个库，用户使用groupcache提供的简单接口，可以快速的开发出自己的业务缓存服务。

值得一题的是groupcache既是服务端库也是客户端库，这就让用户基于groupcache开发的业务缓存系统天生具有分布式特性。

用户使用groupcache的系统架构图如下：

![系统架构图](/media/pic/groupcache-system.png)

## 运作机制

参考系统架构图，使用groupcache建立的系统运作机制是这样的

1. client和任意一个DataService建立连接，并通过DataService提供的接口获取数据
2. DataService调用groupcache的Get方法，获取指定Key的数据

groupcache Get方法运作机制
1. 如果该key命中本地缓存，则直接返回缓存数据
2. 如果本地缓存未命中，根据key值得到其对应的peer，并询问peer该key对应的业务数据，并返回
3. 如果从对应peer中获取失败，则本地加载业务数据到缓存中，并返回

这其中牵连到了**缓存协调填充机制**和**热点数据备份机制**, 后面会有详细的介绍。


## 源码分析

#### Group

Group是groupcache的核心数据结构

{% highlight go %}
// 一个group可以理解为一个namespace
type Group struct {
    // group名
    name       string
    // 用户传入的回调，groupcache调用其访问业务数据
    // type Getter interface {
    //   Get(ctx Context, key string, dest Sink)
    // }
    getter     Getter

    // 保证peers只会初始化一次
    peersOnce  sync.Once

    // 根据key值查找peer的接口，使用一致性hash算法实现
    peers      PeerPicker

    // 包括mainCache和hotCache的缓存容量
    cacheBytes int64

    // mainCache is a cache of the keys for which this process
    // (amongst its peers) is authorative. That is, this cache
    // contains keys which consistent hash on to this process's
    // peer number.
    // 通过PeerPicker中的算法，mainCache只存放属于自己的key
    mainCache cache

    // hotCache contains keys/values for which this peer is not
    // authorative (otherwise they would be in mainCache), but
    // are popular enough to warrant mirroring in this process to
    // avoid going over the network to fetch from a peer.  Having
    // a hotCache avoids network hotspotting, where a peer's
    // network card could become the bottleneck on a popular key.
    // This cache is used sparingly to maximize the total number
    // of key/value pairs that can be stored globally.
    // hotCache存放的是不属于自己的key，存放的是其它peers的热点key
    // 可以有效的避免大量的通过网络从另一个peer取得数据，从而避免网卡瓶颈
    hotCache cache

    // loadGroup ensures that each key is only fetched once
    // (either locally or remotely), regardless of the number of
    // concurrent callers.
    // 在大量并发请求的情况下，保证key对应的缓存只会被加载一次
    // 有效避免cache miss时产生的"惊群现象"
    loadGroup singleflight.Group

    // Stats are statistics on the group.
    Stats Stats
}
{% endhighlight %}

#### Cache

groupcache使用的缓存采用的是[lru](http://baike.baidu.com/view/70151.htm)算法

{% highlight go %}
// Cache is an LRU cache. It is not safe for concurrent access.
// 非线程安全
type Cache struct {
    // MaxEntries is the maximum number of cache entries before
    // an item is evicted. Zero means no limit.
    // 缓存容量
    MaxEntries int

    // OnEvicted optionally specificies a callback function to be
    // executed when an entry is purged from the cache.
    // 缓存被删除时的回调
    OnEvicted func(key Key, value interface{})

    ll    *list.List
    cache map[interface{}]*list.Element
}

// A Key may be any value that is comparable. 
// See http://golang.org/ref/spec#Comparison_operators
type Key interface{}

type entry struct {
    key   Key
    value interface{}
}

{% endhighlight %}

groupcache直接使用的缓存，又在lru的缓存上包装了一层，加了统计信息，锁以及大小限制

{% highlight go %}
// cache is a wrapper around an *lru.Cache that adds synchronization,
// makes values always be ByteView, and counts the size of all keys and
// values.
type cache struct {
    mu         sync.RWMutex
    nbytes     int64 // of all keys and values
    lru        *lru.Cache
    nhit, nget int64
    nevict     int64 // number of evictions
}
{% endhighlight %}

#### PeerPicker接口 
通过该接口可以获得一个key被shard到哪个peer，该接口被HTTPPool实现

{% highlight go %}
// ProtoGetter is the interface that must be implemented by a peer.
type ProtoGetter interface {
	Get(context Context, in *pb.GetRequest, out *pb.GetResponse) error
}

// PeerPicker is the interface that must be implemented to locate
// the peer that owns a specific key.
type PeerPicker interface {
	// PickPeer returns the peer that owns the specific key
	// and true to indicate that a remote peer was nominated.
	// It returns nil, false if the key owner is the current peer.
	PickPeer(key string) (peer ProtoGetter, ok bool)
}
{% endhighlight %}

#### HTTPPool

HTTPPool提供了groupcache peers间的交互方式(http方式)，以及peer选择的实现。它对groupcache.Group来说是透明的


## 自动备份热点数据

groupcache将自动热点数据备份实现的非常简单有效。目前的实现是将其他peer的1/10的key缓存在本地hotCache中。

想过代码代码在从peer获取数据的逻辑中

{% highlight go %}
func (g *Group) getFromPeer(ctx Context, peer ProtoGetter, key string) (ByteView, error) {
	req := &pb.GetRequest{
		Group: &g.name,
		Key:   &key,
	}
	res := &pb.GetResponse{}
	err := peer.Get(ctx, req, res)
	if err != nil {
		return ByteView{}, err
	}
	value := ByteView{b: res.Value}
	// TODO(bradfitz): use res.MinuteQps or something smart to
	// conditionally populate hotCache.  For now just do it some
	// percentage of the time.
	if rand.Intn(10) == 0 {
		g.populateCache(key, value, &g.hotCache)
	}
	return value, nil
}
{% endhighlight %}

作者在TODO中指出以后会用更加智能的方式填充hotCache，比如根据peer的最小QPS，如果最小QPS大于某个阈值则备份到hotCache中。
目前的实现只是取一段时间的百分比。

