---
layout: post
title: "groupcache简介"
description: ""
category: "海量之道"
tags: ["分布式", "缓存"]
---

groupcache是google开源的分布式缓存库，用于开发提供缓存功能的系统(目前只支持Go语言)。groupcache已经实际运用到了google的多个产品中，如: dl.google.com

以[官方文档](https://github.com/golang/groupcache/blob/master/README.md)的说明
    
> groupcache is a caching and cache-filling library, intended as a replacement for memcached in many cases.

groupcache意在在某些情况下替换memcached的使用，这里有两层含义

1. 某些情况下替换。表示groupcache在功能上不能完全覆盖memcached，某些特定业务场景下不能替换使用。后面会具体分析groupcache适用的业务场景；
2. 意在替换memcached。表示groupcache解决了memcached中的一些问题，值得我们替换使用。后面会对groupcache针对memcached的一些改进予以详细介绍；


## 业务场景

作者在[文档](https://github.com/golang/groupcache/blob/master/README.md)中这样描述

> does not support versioned values. If key "foo" is value "bar", key "foo" must always be "bar". 
> There are neither cache expiration times, nor explicit cache evictions. 
> Thus there is also no CAS, nor Increment/Decrement. 

简单的将就是，groupcache不支持缓存的过期删除和更新机制。一个key的值写进去，就不会变动。

groupcache的设计并未考虑做一个类似memcached的通用的缓存系统，而是只考虑提供静态资源的缓存，以提升静态资源服务的效率。

如：apt get，cdn等典型的静态资源服务

启示：从具体业务出发，专注对一个问题的解决的设计是google的一贯作风，这样可以大幅简化设计及实现，并能将问题解决的完美切优雅。

## compare with memcached

groupcache和memcached在分布式上都是通过key的一致性hash来分片(shard)，根据key来选择对应的peer。

memcached目前有两个问题：

1. 如果存在大量访问一个key的情况（在cdn服务中很常见），memcached会出现一个节点过热的情况，导致服务性能下降。
   
   groupcache通过auto-mirror(自动镜像)机制来解决，将过热(super-hot) items，自动镜像到其他节点。

2. 如果访问的key未命中memcached缓存，用户程序会查询数据库(或其他组件)，并填充memcached，
   如果查询这个未命中缓存的key频率太高，在缓存填充阶段，会出现数据库(或其他组件)的惊群现象。

   groupcache引入协调缓存填充机制(coordinate cache filling mechanism)，协调缓存填充，
   如果缓存正在被填充，另一个对同一个key的请求会被阻塞，等待缓存被填充完毕，而不会去再次查询数据库

   ![缓存填充机制](/media/pic/groupcache-cache-filling.png)

另外，groupcache使用Go语言编写(作者不考虑将它移植到其他语言)，它不像memcached一样是独立的服务(standalone server)。
它是一个库(library)，即是服务器端库，又是客户端库，基于这种设计，用户可以很容易的在产品中加入分布式缓存功能。

## 简单实例

首先安装groupcache

    go get github.com/golang/groupcache


下面是一个简单的例子代码

{% highlight go %}

package main

import (
    "fmt"
    "github.com/golang/groupcache"
    "log"
    "net/http"
    "os"
    "strings"
)

func main() {
    port := os.Args[1]

    // 集群配置
    peers := groupcache.NewHTTPPool("http://localhost:" + port)
    peers.Set("http://localhost:8081", "http://localhost:8082", "http://localhost:8083")

    // 创建cache
    cache := groupcache.NewGroup("hello", 1024*1024*1024*16, groupcache.GetterFunc(
        func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
            log.Println("fetch " + key)
            dest.SetString(key + " from " + port)
            return nil 
    })) 
 
    log.Println("group: ", cache.Name())
    http.HandleFunc("/s/", func(w http.ResponseWriter, r *http.Request) {
        parts := strings.SplitN(r.URL.Path[len("/s/"):], "/", 1)
        if len(parts) != 1 { 
            http.Error(w, "Bad Request", http.StatusBadRequest)
            return
        }

        // 通过cache获取数据
        var data []byte
        cache.Get(nil, parts[0], groupcache.AllocatingByteSliceSink(&data))
        w.Write(data)
        
        // 打印统计
        log.Printf("%+v", cache.Stats);
    })  
    http.ListenAndServe(":" + port, nil)
}

{% endhighlight %}

使用方法很简单，大体都是以下三个步骤：

1. 配置集群
   
   集群的配置非常简单，只需要设置自己的地址，和集群内各peer（包括自己）的地址即可，groupcache使用http协议

   部署集群也非常简单，不需要针对不同节点进行额外配置

2. 创建cache

   使用NewGroup创建，创建时必须指定三个参数：

   1. group名
   2. cache大小
   3. 获取key对应本地数据的对调

3. 使用Get方法获取数据

想要了解groupcache较为复杂一点的例子看一参考[Playing With Groupcache](http://www.capotej.com/blog/2013/07/28/playing-with-groupcache/)

