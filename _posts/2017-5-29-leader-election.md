---
layout: post
title: 分布式锁和选举
categories: [distribution lock, leader election]
tags: [distribution lock, leader election]
---

# 分布式锁

项目中实现了有状态服务的`leader election`功能，这就涉及到了分布式锁的实现。
分布式锁的原理很简单，如下：

![distribution lock]({{ "/img/posts/2017-5-29-leader-election.md/distribution_lock.png" | prepend: site.baseurl }})

打个简单的比方，分布式锁的竞争就像两个人A和B想往一个保险箱safe里放自己的签名（signature）。保险箱的size只有1，所以同时只能容下一个人的签名。并且这个签名是带有有效期的（expiration），因此有效期内不被再次刷新就作废，保险箱重新被腾出来。

所以，当A和B同时往保险箱里放签名，假设A获胜，于是A把自己的签名放进保险箱，并约定一个过期时间，同时，A会定时去更新下过期时间，直到他不再需要这个签名有效，或者由于其他原因无法更新为止。
而B由于在竞争中失败，这时他会一直去监视这个保险箱里A签名的过期状态，一旦发现他过期了，B则重新开始一轮新的竞争。

# 选举实现

从上面的例子可以提取出一个选举算法的流程图：

![leader election]({{ "/img/posts/2017-5-29-leader-election.md/leader_election.png" | prepend: site.baseurl }})

每一个竞选者都是一个candidate的实现，每个candidate会实现自己的`onElected`和`onDefeated`方法，作为选举成功或从成功转为失败的callback。每个candidate都会对应一个同样的key，这个key标识了一个分布式锁的id。并且每个candidate都有自己的特有signature，可以是id，也可以是hostname。

在选举开始后，每个candidate都会去创建一个带自己signature的，id为key的这么一个记录，并制定过期时间TTL。因为id唯一，所以只会有一个成功。

1. 如果成功，则表明获得了锁，就可以去调用candidate的`onElected`的callback，执行具体业务，并且启动一个线程去定时刷新signature的TTL。如果刷新TTL失败，就说明失去了锁，则调用candidate的`onDefeated`callback，结束相关业务。之后可以继续重新参与竞选。

2. 如果失败，且错误不是`duplicated key`，则重新开始竞选流程，这时可能是创建记录的时候出现的不可预期的错误。如果错误是`duplicated key`，则表明一个具有key的记录已经存在，可以继续尝试替换记录的signature。

3. 如果替换成功，则进入`步骤1`。否则开始监控记录的signature，一旦发现其过期，或者更甚，连记录都不存在了，则开始新的竞选流程。

## KV storage

因为KV storage自带TTL功能，所以实现起来会比较简单，其中key就是storage中的path。需要注意的是，在对记录进行操作的时候需要采用CAS方式。

## 数据库

一般数据库的实现一来于单个SQL语句的原子性。数据库表至少包括如下三个字段：

| 字段 | 描述 |
| -- | ----- |
| Key | PK |
| Signature | 每个candidate自己的特征，如hostname，id等 |
| UpdatedTime | 记录被更新的时间 |

`Key`和`Signature`之前描述过，`UpdatedTime`是当前记录被更新的时间。因为数据库没有TTL的功能，所以这里用时间戳来描述：每次执行SQL对记录进行修改时（如更新TTL，尝试替换Signature等），都需要带上当前的时间`now`，如果`now - UpdatedTIme > TTL`则视当前记录的签名已经过期。

| action | SQL |
| -- | ----- |
| check unexpired | select * from [table] where key=[key] and [TTL] > timestampdiff(second, updated_time, [current_time]) |
| insert key | insert into <table> (key, signature, updated_time) values ([key], [signature], [current_time]) |
| try replace | update [table] set updated_time = [current_time], signature = [signature] where key = [key] and [TTL] <= timestampdiff(second, updated_time, [current_time]) |
| refresh TTL | update [table] set updated_time = [current_time] where key = [key] and signature = [signature] and [TTL] >= timestampdiff(second, updated_time, [current_time]) |
| release lock | delete from [table] where key = [key] and signature = [signature] and [TTL] < timestampdiff(second, updated_time, [current_time]) |
