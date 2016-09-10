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
的集合并进行分析统计对比等。没有做不到，只有想不到。

## 7、Pub/Sub 构建实时消息系统
Redis 的 Pub/Sub 系统可以构建实时的消息系统，比如很多用 Pub/Sub 构建的实时聊天系统
的例子。

## 8、构建队列系统
使用 list 可以构建队列系统，使用 sorted set 甚至可以构建有优先级的队列系统。

##9、缓存
这个不必说了，性能优于 Memcached，数据结构更多样化。
