---
layout: post
title: "浅析groupcache架构哲学"
description: ""
category: "海量之道"
tags: ["分布式", "缓存"]
---

出于对golang的浓厚兴趣，研究了一些使用golang编写的开源项目，groupcache是其中较为简单的一个项目。

groupcache是memcached的作者Brad Fitzpatrick用Go语言开发的memcached替代版本，主要用于Google的下载服务。本文主要从和大家熟悉的memcached对比来介绍groupcache的特性和架构。

## 一、特性对比
这里说groupcache是memcached的替代版本着实有点言过其实，至少从开源代码的版本来看（我不知道Google内部是否还有另外一个版本），在功能特性上groupcache和memcached有较大的差异。

google内部使用golang重构下载服务（dl.google.com）。groupcache的项目的目的便是为下载服务提供静态资源缓存，以提升服务效率。认识这点很重要，因为groupcache的架构设计和接口设计都是围绕这一业务场景进行的（不会考虑非静态资源的场景）。熟悉memcached的同学都知道memcached的使用场景显然更加通用，它通常作为业务访问数据库的缓存（数据库内容可变），当然也可以作为静态资源缓存。

虽然都是缓存系统，但groupcache更窄的应用场景，导致了它和memcached在架构设计上的迥异。关注应用场景在google的后台架构设计中一直得到了非常好的实践。大名鼎鼎的GFS（Google File System）开始就是为搜索索引系统设计的，它只关注存放像索引文件这样的大尺寸文件，所以定义每个chunk的大小为64M，这直接导致了单Master的设计给整个架构设计带来的简洁性。这点值得我们好好揣摩学习。

## 二、接口对比
groupcache仅用于提供静态资源缓存。那么我们可以这样定义它的Get("key")方法（以下是官方文档的描述）

> does not support versioned values. If key "foo" is value "bar", key "foo" must always be "bar". There are neither cache expiration times, nor explicit cache evictions. Thus there is also no CAS, nor Increment/Decrement.

简单的讲就是，groupcache不支持缓存的过期删除和更新机制。一个key的值写进去，将不会变动。（当然Key会被LRU策略淘汰）,这个接口上的约定会大大简化groupcache的设计

所以groupcache只有Get接口，而没有Put，Delete等接口。这和我们所熟知的memcached接口有很大的不同。下面是使用groupcache实现静态资源缓存的代码片段

{% highlight go %}
// ...
// 创建cache
cache := groupcache.NewGroup("thumbnail", 16 << 30, groupcache.GetterFunc(
    func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
        data := readThumbnailFromGFS(key)
        dest.SetBytes(data)
        return nil 
}))

// 通过cache获取数据
var data []byte
cache.Get(nil, "1.png", groupcache.AllocatingByteSliceSink(&data))
// ...
{% endhighlight %}


接口使用是这样的：

1. 创建一个名为thumbnail的groupcache，第二个参数指定了缓存总大小，第三个参数是个回调函数，定义了从后端存储获取真实数据的逻辑
2. 通过Get接口获取指定key对应的数据

很容易理解上面代码片段的缓存工作原理。比如要访问1.png这样的图片资源，先去缓存中查询是否存在，如果存在则直接返回缓存数据，如果不存在，则调用注册的回调函数去GFS中获取，填充缓存后将图片数据返回。

## 三、架构对比
上一节的接口设计只简要的说明了groupcache单机工作的原理。对一个缓存系统来说，分布式是一个非常重要特性（保证可扩容性，服务可靠性）。groupcache如何比memcached更加优雅的实现分布式？既然groupcache牺牲了更广泛的应用场景，它是用的哪些更有用的特性做的这笔交易？这是本节重点讨论的话题。

首先说说groupcache在分布式特性上和memcached的相同点。它们都使用key的一致性hash来进行shard，来决定访问哪个缓存节点。

再来看看他们的之间设计的不同点：

1. groupcache和memcached架构设计上最大的不同是groupcache本身不作为一个独立的系统部署到额外的机器上去，它仅仅是一个库（既是客户端库也是服务端库）,只要写非常少量的代码就可以为业务提供分布式缓存的功能。memcached和groupcache的架构对比如下图所示：
![groupcache vs memcached](/media/pic/groupcache-vs-memcached.png)

2. 对于静态资源来讲，很可能同一时刻有上千个请求访问同一资源。如果这时没有命中缓存，memcached会直接告诉client: "Sorry, cache miss"，直到client从后端存储中读取到数据并设置缓存为止，这就很容导致后端存储系统发生"惊群"现象，造成系统不稳定。

   正如我们前面所了解的，groupcache的读取缓存和读取后端存储都在同一个模块中被调度，就可以非常方便的做到对同一个key的多个请求，只会请求一次后端存储。这从本质上避免了cache失效对后端系统造成的"惊群"现象。这个是在[singleflight](https://github.com/golang/groupcache/tree/master/singleflight)模块中实现的

   两者之间的对比如下图所示：
   ![memcache-thundering-herd](/media/pic/memcache-thundering-herd.png)

3. 对于热点资源的访问，Memcached没有进行特殊的处理。如果一个资源过热，那么其所在的memcached服务器可能成为访问瓶颈。而groupcache会自动镜像(automatic mirroring)热点资源(super-hot items)，如果一个资源过热，那么它会在本地也缓存一份作为镜像。这就能使对热点资源的访问能分散到不同缓存服务器上，均分负载。groupcache在实现上并没有获取一个Key的负载情况，判断其是否过热，而是直接将路由到其他peer的1/10的key放到自己本地的热点缓存中（如果一个资源过热，那么他有极大可能在这1/10的集合中）。如下代码所示：
{% highlight go %}
// TODO(bradfitz): use res.MinuteQps or something smart to
// conditionally populate hotCache.  For now just do it some
// percentage of the time.
if rand.Intn(10) == 0 {
    g.populateCache(key, value, &g.hotCache)
}
{% endhighlight %}


## 四、结束语
groupcache并不是一个复杂的系统，但是为具体业务设计并深度优化的思想在这个简单的系统中被体现的淋漓尽致。系统的设计必定是基于业务特性的，不能凭空想象的得到一个看似通用实则无用的系统。 

本文没有全面的剖析groupcache的设计，也没有逐行分析源代码，你当然不会凭我三言两语就对这个项目有了完整的认识。这时候请相信一句话

> talk is cheap, f\*\*king the source code