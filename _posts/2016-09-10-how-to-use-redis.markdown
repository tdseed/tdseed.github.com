---
layout: post
title:  "早会分享redis相关"
date:   2016-09-10 10:10:09
categories: technology
author: tank
---

## 1、取最新 N 个数据的操作
比如典型的取你网站的最新文章，通过下面方式，我们可以将最新的 5000 条评论的 ID 放在
Redis 的 List 集合中，并将超出集合部分从数据库获取。
使用 LPUSH latest.comments<ID>命令，向 list 集合中插入数据
插入完成后再用 LTRIM latest.comments 0 5000 命令使其永远只保存最近 5000 个 ID
然后我们在客户端获取某一页评论时可以用下面的逻辑

```javascript
    FUNCTION get_latest_comments(start,num_items):
        id_list = redis.lrange("latest.comments",start,start+num_items-1)
        IF id_list.length < num_items
            id_list = SQL_DB("SELECT ... ORDER BY time LIMIT ...")
        END
        RETURN id_list
    END
```

如果你还有不同的筛选维度，比如某个分类的最新 N 条，那么你可以再建一个按此分类的
List，只存 ID 的话，Redis 是非常高效的。

## 2、排行榜应用，取 TOP N 操作
这个需求与上面需求的不同之处在于，前面操作以时间为权重，这个是以某个条件为权重，
比如按顶的次数排序，这时候就需要我们的 sorted set 出马了，将你要排序的值设置成 sorted
set 的 score，将具体的数据设置成相应的 value，每次只需要执行一条 ZADD 命令即可。

## 3、需要精准设定过期时间的应用
比如你可以把上面说到的 sorted set 的 score 值设置成过期时间的时间戳，那么就可以简单
地通过过期时间排序，定时清除过期数据了，不仅是清除 Redis 中的过期数据，你完全可以
把 Redis 里这个过期时间当成是对数据库中数据的索引，用 Redis 来找出哪些数据需要过期
删除，然后再精准地从数据库中删除相应的记录。

## 4、计数器应用
Redis 的命令都是原子性的，你可以轻松地利用 INCR，DECR 命令来构建计数器系统。

## 5、Uniq 操作，获取某段时间所有数据排重值
这个使用 Redis 的 set 数据结构最合适了，只需要不断地将数据往 set 中扔就行了，set 意为
集合，所以会自动排重。

## 6、实时系统，反垃圾系统
通过上面说到的 set 功能，你可以知道一个终端用户是否进行了某个操作，可以找到其操作
的集合并进行分析统计对比等。

## 7、Pub/Sub 构建实时消息系统
Redis 的 Pub/Sub 系统可以构建实时的消息系统，比如很多用 Pub/Sub 构建的实时聊天系统
的例子。

## 8、构建队列系统
使用 list 可以构建队列系统，使用 sorted set 甚至可以构建有优先级的队列系统。

## 9、缓存
性能优于 Memcached，数据结构更多样化。

## 简单分析：
redis有自己的api支持java、ruby等很多语言
每种语言都可能封装库来让redis的操作更加方便，如ruby圈有redis-rb，而ohm又做了层抽象，Object-Hash Mapping。。
redis有不同的数据类型，灵活的选择使用，strings, lists, hashes, sets, sorted sets
还有redis常用命令、高级用法等

当时的有一款基于地理位置的社交app，后端设计很多不同的redis key，
比如把附近的地点信息放在了redis中，在进行搜索自动补全时使用，加快自动补全的速度
把redis直接用作搜索还是少的

### 1.你可以用Redis的lists做许多有趣的事，例如你可以：
在社交网络中建立一个时间线模型，使用LPUSH去添加新的元素到用户时间线中，使用LRANGE去检索一些最近插入的条目。
你可以同时使用LPUSH和LTRIM去创建一个永远不会超过指定元素数目的列表并同时记住最后的N个元素。
列表可以用来当作消息传递的基元（primitive），例如，众所周知的用来创建后台任务的Resque Ruby库。
你可以使用列表做更多事，这个数据类型支持许多命令，包括像BLPOP这样的阻塞命令。

### 2.你可以用Redis的sets做很多有趣的事，例如你可以：
用集合跟踪一个独特的事。想要知道所有访问某个博客文章的独立IP？只要每次都用SADD来处理一个页面访问。那么你可以肯定重复的IP是不会插入的。
Redis集合能很好的表示关系。你可以创建一个tagging系统，然后用集合来代表单个tag。接下来你可以用SADD命令把所有拥有tag的对象的所有ID添加进集合，这样来表示这个特定的tag。如果你想要同时有3个不同tag的所有对象的所有ID，那么你需要使用SINTER.
使用SPOP或者SRANDMEMBER命令随机地获取元素。

### 3.使用sorted sets你可以很好地完成，很多在其他数据库中难以实现的任务。
在一个巨型在线游戏中建立一个排行榜，每当有新的记录产生时，使用ZADD 来更新它。你可以用ZRANGE轻松地获取排名靠前的用户， 你也可以提供一个用户名，然后用ZRANK获取他在排行榜中的名次。 同时使用ZRANK和ZRANGE你可以获得与指定用户有相同分数的用户名单。 所有这些操作都非常迅速。
有序集合通常用来索引存储在Redis中的数据。 例如：如果你有很多的hash来表示用户，那么你可以使用一个有序集合，这个集合的年龄字段用来当作评分，用户ID当作值。用ZRANGEBYSCORE可以简单快速地检索到给定年龄段的所有用户。
有序集合或许是最高级的Redis数据类型，所以花些时间查看完整的有序集合（Sorted sets）命令列表去探索你能用Redis干些什么吧！

Key-Value Store 更加注重对海量数据存取的性能、分布式、扩展性支持上，并不需要传统
关系数据库的一些特征，例如：Schema、事务、完整 SQL 查询支持等等，因此在分布式环
境下的性能相对于传统的关系数据库有较大的提升。

http://blog.csdn.net/neosmith/article/details/46811089
