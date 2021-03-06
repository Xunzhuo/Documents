---
title: 【负载均衡】Envoy负载均衡原理
description: 本文是对Envoy的负载均衡算法的探究
toc: true
authors:
  - Hugo Authors
tags:
  - Envoy
  - 负载均衡
categories: Envoy
series:
date: '2019-03-11'
lastmod: '2019-03-11'
draft: false
---


## 前言

文章结构是：

1. 负载均衡的定义
2. Envoy负载均衡器的类型
3. Envoy中负载均衡器的基本概念
4. 实现各个负载均衡器的算法原理

## 负载均衡的定义

> Load balancing is a way of distributing traffic between multiple hosts within a single upstream cluster in order to effectively make use of available resources. There are many different ways of accomplishing this, so Envoy provides several different load balancing strategies. At a high level, we can break these strategies into two categories: global load balancing and distributed load balancing.

上面是Envoy官方文档对于负载均衡的一个解释，说通俗点就是一种将流量分布到一个`upstream`集群中的多台机器上的方法。

而实现的方式有很多种，Envoy提供了几种负载均衡的策略，主要是分为`global load balancing`（全局负载均衡）和` distributed load balancing`（分布式负载均衡）两类。

前者是通过一个中心的控制节点来决策流量到底分布到哪些机器，比如通过控制节点来调节权重、优先级、区域等；

而后者则是Envoy自己根据自定义的规则来决定流量到底分布到哪些机器上，比如根据区域来决策、或者根据自己使用的负载均衡算法、又或者是根据机器的健康状况来决策，Envoy是同时支持这两种策略的。

## Envoy负载均衡器的类型

![image-20210129171836925](C:\Users\bitliu\AppData\Roaming\Typora\typora-user-images\image-20210129171836925.png)

上面这张图是Envoy中的负载均衡器的实现类图：

+ 蓝色部分是各个负载均衡器实现所依赖的基类
+ 其他每一种颜色代表一种负载均衡器的实现。

根据它们所实现的基类可以知道这些负载均衡器的类型。在Envoy中大致可以分为五类。

1. 每一个worker线程包含一个负载均衡器实例(`LoadBalancerBase`)
2. 带有区域感知支持的负载均衡器(`ZoneAwareLoadBalancerBase`)
3. 带有权重支持的负载均衡器(`EdfLoadBalancerBase`)
4. 所有线程共享同一个负载均衡器实例(`ThreadAwareLoadBalancerBase`)
5. 自定义的负载均衡器(`LoadBalancer`)

从继承关系反推出，六个负载均衡器的特点：

**Round Robin 和 Least Request**

+ 支持权重
+ 支持区域感知
+ 每一个线程一个负载均衡器实例

**Ring Hash 和 Maglev** 

+ 全局一个负载均衡器实例
+ 不支持区域感知

**Random**

+ 不支持权重
+ 支持区域感知

**Subset**则是按照一套自己的算法

## 负载均衡的基本概念

在正式开始分析Envoy中的负载均衡的时候，我们需要介绍下Envoy关于这个部分的一些基本概念：

Envoy首先会根据指定的路由规则选取集群，而负载均衡的对象就是集群下面的机器列表。

Envoy中有很多概念是为了加强负载均衡机制的，下面我们来一个个介绍下。

首先是**Priority**，一个集群下面可以配置多个**Priority**，每一个**Priority**会存在一些机器，是用来表示一组机器的优先级的，默认从0开始，优先级最高。

下一个概念就是**Locality**，用来表示机器所在的位置，主要的用途就是用来实现区域感知路由。

最后通过一张图来表示下这几个的关系：

<img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/56cb6eddc9b910c0992850ab997e0b54.jpg" alt="cluster-priority.jpg" style="zoom:50%;" />

## 负载均衡算法原理

### Round Robin

这是一个简单的策略，每个健康的上游主机按循环顺序选择。

如果将权重分配给本地的端点，则使用加权 Round Robin（循环）调度，其中较高权重的端点将更频繁地出现在循环中以实现有效权重。

``` c++
/**
 * A round robin load balancer. When in weighted mode, EDF scheduling is used. When in not
 * weighted mode, simple RR index selection is used.
 */

class RoundRobinLoadBalancer : public EdfLoadBalancerBase {
public:
  RoundRobinLoadBalancer(const PrioritySet& priority_set, const PrioritySet* local_priority_set,
                         ClusterStats& stats, Runtime::Loader& runtime,
                         Runtime::RandomGenerator& random,
                         const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config)
      : EdfLoadBalancerBase(priority_set, local_priority_set, stats, runtime, random,
                            common_config) {
    initialize();
  }

private:
  void refreshHostSource(const HostsSource& source) override {
    // insert() is used here on purpose so that we don't overwrite the index if the host source
    // already exists. Note that host sources will never be removed, but given how uncommon this
    // is it probably doesn't matter.
    rr_indexes_.insert({source, seed_});
  }
  double hostWeight(const Host& host) override { return host.weight(); }
  HostConstSharedPtr unweightedHostPick(const HostVector& hosts_to_use,
                                        const HostsSource& source) override {
    // To avoid storing the RR index in the base class, we end up using a second map here with
    // host source as the key. This means that each LB decision will require two map lookups in
    // the unweighted case. We might consider trying to optimize this in the future.
    ASSERT(rr_indexes_.find(source) != rr_indexes_.end());
    return hosts_to_use[rr_indexes_[source]++ % hosts_to_use.size()];
  }

  std::unordered_map<HostsSource, uint64_t, HostsSourceHash> rr_indexes_;
};
```



### LeastRequest

客户端的每一次请求服务在服务器停留的时间都可能会有较大的差异，随着工作时间的加长，如果采用简单的轮循或随机均衡算法，每一台服务器上的连接进程可能会产生极大的不同，这样的结果并不会达到真正的负载均衡。

最少连接数均衡算法对内部中有负载的每一台服务器都有一个数据记录，记录的内容是当前该服务器正在处理的连接数量，当有新的服务连接请求时，将把当前请求分配给连接数最少的服务器，使均衡更加符合实际情况，负载更加均衡。

``` c++
/**
 * Weighted Least Request load balancer.
 *
 * In a normal setup when all hosts have the same weight it randomly picks up N healthy hosts
 * (where N is specified in the LB configuration) and compares number of active requests. Technique
 * is based on http://www.eecs.harvard.edu/~michaelm/postscripts/mythesis.pdf and is known as P2C
 * (power of two choices).
 *
 * When hosts have different weights, an RR EDF schedule is used. Host weight is scaled
 * by the number of active requests at pick/insert time. Thus, hosts will never fully drain as
 * they would in normal P2C, though they will get picked less and less often. In the future, we
 * can consider two alternate algorithms:
 * 1) Expand out all hosts by weight (using more memory) and do standard P2C.
 * 2) Use a weighted Maglev table, and perform P2C on two random hosts selected from the table.
 *    The benefit of the Maglev table is at the expense of resolution, memory usage is capped.
 *    Additionally, the Maglev table can be shared amongst all threads.
 */

class LeastRequestLoadBalancer : public EdfLoadBalancerBase {
public:
  LeastRequestLoadBalancer(
      const PrioritySet& priority_set, const PrioritySet* local_priority_set, ClusterStats& stats,
      Runtime::Loader& runtime, Runtime::RandomGenerator& random,
      const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config,
      const absl::optional<envoy::config::cluster::v3::Cluster::LeastRequestLbConfig>
          least_request_config)
      : EdfLoadBalancerBase(priority_set, local_priority_set, stats, runtime, random,
                            common_config),
        choice_count_(
            least_request_config.has_value()
                ? PROTOBUF_GET_WRAPPED_OR_DEFAULT(least_request_config.value(), choice_count, 2)
                : 2) {
    initialize();
  }

private:
  void refreshHostSource(const HostsSource&) override {}
  double hostWeight(const Host& host) override {
    // Here we scale host weight by the number of active requests at the time we do the pick. We
    // always add 1 to avoid division by 0. It might be possible to do better by picking two hosts
    // off of the schedule, and selecting the one with fewer active requests at the time of
    // selection.
    // TODO(mattklein123): @htuch brings up the point that how we are scaling weight here might not
    // be the only/best way of doing this. Essentially, it makes weight and active requests equally
    // important. Are they equally important in practice? There is no right answer here and we might
    // want to iterate on this as we gain more experience.
    return static_cast<double>(host.weight()) / (host.stats().rq_active_.value() + 1);
  }
  HostConstSharedPtr unweightedHostPick(const HostVector& hosts_to_use,
                                        const HostsSource& source) override;
  const uint32_t choice_count_;
};

```

### Random

随机负载均衡器选择一个随机的健康主机。如果没有配置健康检查策略，则随机负载均衡器通常比 Round Robin 更好。随机选择可以避免主机出现故障后对集合中的主机造成偏见。



``` c++
/**
 * Random load balancer that picks a random host out of all hosts.
 */
class RandomLoadBalancer : public ZoneAwareLoadBalancerBase {
public:
  RandomLoadBalancer(const PrioritySet& priority_set, const PrioritySet* local_priority_set,
                     ClusterStats& stats, Runtime::Loader& runtime,
                     Runtime::RandomGenerator& random,
                     const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config)
      : ZoneAwareLoadBalancerBase(priority_set, local_priority_set, stats, runtime, random,
                                  common_config) {}

  // Upstream::LoadBalancerBase
  HostConstSharedPtr chooseHostOnce(LoadBalancerContext* context) override;
};

---
    
HostConstSharedPtr RandomLoadBalancer::chooseHostOnce(LoadBalancerContext* context) {
  const absl::optional<HostsSource> hosts_source = hostSourceToUse(context);
  if (!hosts_source) {
    return nullptr;
  }

  const HostVector& hosts_to_use = hostSourceToHosts(*hosts_source);
  if (hosts_to_use.empty()) {
    return nullptr;
  }

  return hosts_to_use[random_.random() % hosts_to_use.size()];
}
```

> 在谈RingHash和Maglev之前，我们需要谈谈一致性Hash算法

## 一致性哈希 Consistent Hashing

在数据量较大的场景下，假设我们因为某些原因需要将原本的 n 个 slot 扩容为 m 个 slot，如果仍然使用 Mod-N 哈希，将会有 n/m 份缓存不能正确命中，从而产生大量的数据库请求，可能导致缓存雪崩。

对于传统的哈希映射，添加或者删除一个 slot，会造成哈希表的全量重新映射；而**一致性哈希**的目的是达成增量式的重新映射，即当 slot 的数量发生变化时，降低重新映射的数量，尽量最小化重新映射（minimum disruption）。

一致性哈希算法的设计关键有 4 点：

1. 平衡性 balance：所有的 key 能被均匀地映射到各个 slot 上；
2. 单调性 monotonicity：增加新的 slot 后，原有的 key 应该被映射到原有的 slot，或新的 slot 上，而不是其他旧的 slot ；
3. 分散 spread：服务扩容或者缩容时，尽量减少数据的迁移；
4. 负载 load：尽量降低 slot 的负载；

### RingHash

RingHash 算法是最常用的一种一致性哈希算法，也叫做ketama 算法， 被广泛的应用在数据库，缓存系统和服务框架上，包括但不限于 memcache, redis, dubbo, nginx 等，其步骤是：

1. 对于一个 [0, uint32] 的区间，将其首尾相连，形成顺时针的环；
2. 对 slot 进行哈希，映射到 [0, uint32] 区间上，并将结果标记到环上；
3. 对 key 进行哈希，映射到区间上，沿着环顺时针寻找并将其分配到距其最近的 slot；

举个例子，假设现在有 N0, N1, N2 三个 slot 以及 a, b, c 三个 key，其中 a 会被分配 N1 上，b 和 c 都会被分配到 N2 上；

<img src="https://raw.githubusercontent.com/ZintrulCre/warehouse/master/resources/consistent-hashing/hash-ring-add-1.png" alt="img" style="zoom:50%;" />

现在我们**新增**一个 N3 slot，并将其映射到 [a, N1] 之间，那么 a 和**所有在 [N0, N3] 之间的 key 都会被重新分配到 N3 这个 slot 上，除此之外的其他所有 key 则不会被重新映射**；

<img src="https://raw.githubusercontent.com/ZintrulCre/warehouse/master/resources/consistent-hashing/hash-ring-add-2.png" alt="img" style="zoom:50%;" />

假设我们**移除** N2 slot，那么**所有在 [N1, N2] 之间的 key 都会被重新映射到 N0 上，除此之外的其他所有 key 则不会被重新映射**。

可以发现，ketama 算法达成了在新增或移除 slot 后的**增量式重新映射**（minimum disruption），不会破坏大多数 key 的映射关系；因为要构造出一个环来存储所有 slot 的 key 被映射到的位置，所以其空间复杂度是 O(n)；为了方便地进行查找，可以将环转换成一个有序数组，在其中进行二分查找，时间复杂度是 O(logn)。

该算法基于将所有主机映射到一个环上，使得从主机集添加或移除主机的更改仅影响 1/N 个请求

一致的哈希负载均衡器只有在使用指定哈希值的协议路由时才有效。最小环大小控制环中每个主机的复制因子。例如，如果最小环大小为 1024 并且有 16 个主机，则每个主机将被复制 64 次。

当使用基于优先级的负载均衡时，优先级也通过哈希选择，因此当后端集合稳定时，选定的端点仍将保持一致。

``` c++
class RingHashLoadBalancer : public ThreadAwareLoadBalancerBase,
                             Logger::Loggable<Logger::Id::upstream> {
public:
  RingHashLoadBalancer(
      const PrioritySet& priority_set, ClusterStats& stats, Stats::Scope& scope,
      Runtime::Loader& runtime, Runtime::RandomGenerator& random,
      const absl::optional<envoy::config::cluster::v3::Cluster::RingHashLbConfig>& config,
      const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config);

  const RingHashLoadBalancerStats& stats() const { return stats_; }

private:
  using HashFunction = envoy::config::cluster::v3::Cluster::RingHashLbConfig::HashFunction;

  struct RingEntry {
    uint64_t hash_;
    HostConstSharedPtr host_;
  };

  struct Ring : public HashingLoadBalancer {
    Ring(const NormalizedHostWeightVector& normalized_host_weights, double min_normalized_weight,
         uint64_t min_ring_size, uint64_t max_ring_size, HashFunction hash_function,
         RingHashLoadBalancerStats& stats);

    // ThreadAwareLoadBalancerBase::HashingLoadBalancer
    HostConstSharedPtr chooseHost(uint64_t hash) const override;

    std::vector<RingEntry> ring_;

    RingHashLoadBalancerStats& stats_;
  };
  using RingConstSharedPtr = std::shared_ptr<const Ring>;

  // ThreadAwareLoadBalancerBase
  HashingLoadBalancerSharedPtr
  createLoadBalancer(const NormalizedHostWeightVector& normalized_host_weights,
                     double min_normalized_weight, double /* max_normalized_weight */) override {
    return std::make_shared<Ring>(normalized_host_weights, min_normalized_weight, min_ring_size_,
                                  max_ring_size_, hash_function_, stats_);
  }

  static RingHashLoadBalancerStats generateStats(Stats::Scope& scope);

  Stats::ScopePtr scope_;
  RingHashLoadBalancerStats stats_;

  static const uint64_t DefaultMinRingSize = 1024;
  static const uint64_t DefaultMaxRingSize = 1024 * 1024 * 8;
  const uint64_t min_ring_size_;
  const uint64_t max_ring_size_;
  const HashFunction hash_function_;
};
```



### Maglev

[Maglev](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44824.pdf) 是 Google 研发的一个负载均衡组件，使用了其自研的一致性哈希算法 Maglev Hashing，其主要思路是通过维护两个 table 来将 key 映射到 slot 上；一个表是 lookup table 查找表，用于将 key 映射到 slot 上；另一个表是 permutation table 排列表，用于记录一个 slot 在 lookup table 中的位置序列：

<img src="https://raw.githubusercontent.com/ZintrulCre/warehouse/master/resources/consistent-hashing/maglev-table.png" alt="img" style="zoom:50%;" />

对于 n 个 slot 和一个长度为 m 的映射序列（即 permutation table 和 lookup table 的长度），我们希望为每一个下标为 i 的 slot 都计算出一个数量为 m 的排列，计算时需要使用两个**不同的哈希函数** h1 和 h2，来计算 offset 和 skip 两个值（这里需要保证每一个 slot 的 name 都不相同）：

<img src="https://raw.githubusercontent.com/ZintrulCre/warehouse/master/resources/consistent-hashing/permutation.png" alt="img" style="zoom:67%;" />

举个例子，假设 n = 3，m = 7，对于下标为 0 的 slot，通过某两个哈希函数计算出来的 offset = 3，skip = 4，为其生成一个长度 m = 7 的 permutation：

```python
permutation[i][0] = (3 + 0 * 4) mod 7 = 3
permutation[i][1] = (3 + 1 * 4) mod 7 = 0
permutation[i][2] = (3 + 2 * 4) mod 7 = 4
permutation[i][3] = (3 + 3 * 4) mod 7 = 1
permutation[i][4] = (3 + 4 * 4) mod 7 = 5
permutation[i][5] = (3 + 5 * 4) mod 7 = 2
permutation[i][6] = (3 + 6 * 4) mod 7 = 6
```

再加上另外两个计算好 permutation 的下标为 1 和 2 的 slot，对应的 permutation table：

|  m   |  s0  |  s1  |  s2  |
| :--: | :--: | :--: | :--: |
|  0   |  3   |  0   |  3   |
|  1   |  0   |  2   |  4   |
|  2   |  4   |  4   |  5   |
|  3   |  1   |  6   |  6   |
|  4   |  5   |  1   |  0   |
|  5   |  2   |  3   |  1   |
|  6   |  6   |  5   |  2   |

现在我们让 3 个 slot 轮流地从其 permutation 中，按顺序选择第一个没有被分配的 key，来填充到之后的 lookup table 中，流程是：

1. s0 的 permutation 中的第一个数字 3 没有被分配，选择 3；
2. s1 的 permutation 中的第一个数字 0 没有被分配，选择 0；
3. s2 的 permutation 中的第一个数字 3 已经被分配了，往后遍历到数字 4，选择 4；
4. s0 在 permutation 中往后遍历，0 和 4 都已经被分配了，选择 1；
5. s1 在 permutation 中往后遍历，选择 2；
6. s2 在 permutation 中往后遍历，4 已经被分配了，选择 5；
7. s0 在 permutation 中往后遍历直到选择 6；

于是就有了 lookup table：

|  m   | slot |
| :--: | :--: |
|  0   |  s1  |
|  1   |  s0  |
|  2   |  s1  |
|  3   |  s0  |
|  4   |  s2  |
|  5   |  s2  |
|  6   |  s0  |

这种方法类似于开放寻址法中的双重哈希，通过使用两个无关的哈希函数来生成排列（也可以使用其他生成随机排列的方法，例如 [fisher-yates shuffle](https://en.wikipedia.org/wiki/Fisher–Yates_shuffle)，必须保证方法的的随机性）降低了哈希碰撞的概率；在增加或移除 slot 时，需要为新的 slot 生成 permutation table，再重新生成 lookup table，这会导致部分重新映射，不满足最小化重新映射（minimum disruption）；维护两个表的空间复杂度是 O(n)，查询的时间复杂度是 O(1)；建立表的复杂度可以参考论文的第 3.4 节。

<img src="https://raw.githubusercontent.com/ZintrulCre/warehouse/master/resources/consistent-hashing/minimal-disruption.png" alt="img" style="zoom:67%;" />

一般来说，与环形散列（“ketama”）算法相比，Maglev 具有快得多的查表编译时间以及主机选择时间（当使用 256K 条目的大环时大约分别为 10 倍和 5 倍）。Maglev 的缺点是它不像环哈希那样稳定。当主机被移除时，更多的键将移动位置（模拟显示键将移动大约两倍）。据说，对于包括 Redis 在内的许多应用程序来说，Maglev 很可能是环形哈希替代品的一大优势。高级读者可以使用这个 [benchmark](https://github.com/envoyproxy/envoy/blob/master//test/common/upstream/load_balancer_benchmark.cc) 来比较具有不同参数的环形哈希与 Maglev。

``` c++
class MaglevLoadBalancer : public ThreadAwareLoadBalancerBase {
public:
  MaglevLoadBalancer(const PrioritySet& priority_set, ClusterStats& stats, Stats::Scope& scope,
                     Runtime::Loader& runtime, Runtime::RandomGenerator& random,
                     const envoy::config::cluster::v3::Cluster::CommonLbConfig& common_config,
                     uint64_t table_size = MaglevTable::DefaultTableSize);

  const MaglevLoadBalancerStats& stats() const { return stats_; }

private:
  // ThreadAwareLoadBalancerBase
  HashingLoadBalancerSharedPtr
  createLoadBalancer(const NormalizedHostWeightVector& normalized_host_weights,
                     double /* min_normalized_weight */, double max_normalized_weight) override {
    return std::make_shared<MaglevTable>(normalized_host_weights, max_normalized_weight,
                                         table_size_, stats_);
  }

  static MaglevLoadBalancerStats generateStats(Stats::Scope& scope);

  Stats::ScopePtr scope_;
  MaglevLoadBalancerStats stats_;
  const uint64_t table_size_;
};
```

> 以上是对Envoy的负载均衡算法的探究