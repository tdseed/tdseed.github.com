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

### 这是之前在公司使用redis和Memcache的场景：

redis主要用于保存一些需要长期在内存中的信息，当作内存数据库而不是缓存。保存在redis中的数据有：

1. 每个商家当天的签到数据。类型为zset，key为“ckin商家id"，值为用户id，排序值为时间。见checkin.add_to_redis方法。通过cronjob清空昨天的签到数据。用于当天实时评分，当天首次签到欢迎消息。
2. 每个商家累计的用户数据。类型为zset，key为“UA商家id"，值为用户id，排序值为时间。
2. 每个商家累计的用户总数／女性用户总数。用户总数的key为“suac商家id”。用户总数的计算方法为：昨天累计用户总数（来自mongodb）＋今天新增用户数（来自redis实时统计）。见CheckinShopStat.rb中的相关方法。
3. Resque的异步任务。
4. 登录用户的新浪微博/QQ的token。
5. 经纬度到城市的映射数据, 见gen_gps_city.rb。
6. 百度经纬度纠偏数据，见migrate_offsetbaidu.rb，key前缀为OF。
7. GCJ经纬度纠偏数据，数据类型为hash，key前缀为GCJ。
6. Rails框架的session数据
7. 每个城市分性别最新用户签到有序集合："HOT#{x}U#{self.city}"。用于热点用户列表展示。
8. 每个Wifi签到过的商家id的集合，"BSSID#{bssid}", 用于定位时提供可选商家，类型为zset，score为签到的次数。
9. 每个城市有活跃优惠券的商家，“ACS#{city}”
10. 每个城市有问答系统的商家，“FaqS#{city}”
11. 每个用户的好友的集合，“Frd#{uid}”, 类型为zset, 排序值为加入的顺序。
12. 每个用户的关注的集合，“Fol#{uid}”, 类型为zset, 排序值为加入的顺序。
13. 每个用户的粉丝的集合，“Fan#{uid}”, 类型为zset, 排序值为加入的顺序。
14. 每个用户的领主地点集合,“LORD#{uid}”, 类型为zset, 排序值为加入的顺序。
15. 每个用户创建的地点审核通过的，“LORD2#{uid}”, 类型为set
16. 每个用户的黑名单集合,“BLACK#{uid}”, 类型为zset, 排序值为加入的顺序。
17. 照片的赞,“Like#{photo.id}”, 类型为zset, 排序值为Time.now.to_i的顺序。
18. 用户加入的群组的集合，“GROUP#{uid}”,类型为set。
19. 海外经纬度到国家的映射，"oversea#{xx},#{xx}",见gen_gps_oversea.rb
20. 可信用户的集合 "KxUsers",类型set, 见 KxUser.init_kx_users_redis
21. 每个用户的个人相关信息，"UF#{uid}", 类型为HASH，键值包括:os/ver等。
22. 每张优惠券的下载数量，"CPD#{coupon_id}", 类型为整数字符串。
23. 每日陪聊用户统计, "PL#{date}#{uid}", 类型为set，值为新用户id。
24. 城市／国家code到名称的映射，CityName#{code}"和"CountryName#{code}"。
25. 用户的qq到用户id的映射，"Q:#{openid}"
26. 用户的手机号码到用户id的映射，"P:#{phone}"
27. 用户的微博uid到用户id的映射，"W:#{wb_uid}"
28. 可信用户能随时定位到的地点: "FakeShops", 类型为set. 取代 $fake_shops
29. 群组对应的商家存到redis, 用于查询地点时查找群组: "GSN", 类型为zset
30. 员工与加入的商家: "STAFF#{uid},#{sid}", 类型为set。
30. 用户进入，但不在热点中显示的地点："UnBroadcast",类型为set
31. 游戏地点排名,“Game#{gid}#{sid}”, 类型为zset, 排序值为用户的分数。
32. 内部用户地点漫游,“FCITY#{uid}”, 类型为string, 保存漫游到的城市编码。
33. 可创建手机网站的地点: "MobileShops", 类型为set
34. 授权用户的集合 "CoUsers",类型set
35. 用户最近去过且有发言的3个地点： "LL3#{uid}",类型zset
36. 外部人员跟随地点 "CoShops",类型set

Memcache在两台服务器中都启动，通过客户端实现负载均衡。Memcache中保存的数据：

1. 所有对象的单个find_by_id缓存: "#{self.class.name}#{_id}"
2. 注册流程中查看附近的用户缓存：:cache_path => Proc.new { |c| c.params }
3. 查看热点用户缓存："HOTU#{city}#{sex}#{skip}#{pcount}", :expires_in => 60.minutes
4. 查看个人信息时的缓存："UI#{uid}#{vid}", :expires_in => 12.hours， 待取消（user_info/get接口替换为user_info/basic接口后）
4. 个人照片墙缓存："UP#{uid}-#{pcount}"
5. 用户头像列表缓存："ULOGOS#{uid}"
6. 用户最后出现的地点缓存："LASTL:#{self.id}"
7. 用户最后一次定位的经纬度："LLOC:#{self.id}"
7. 照片墙第一页的缓存："SP#{sid}-#{pcount}"
8. 商家的员工："STF#{sid}"
9. 商家的分享优惠券的文案："SCPT#{sid}"
9. 用户是否在24小时内创建过地点："ADDSHOP#{user_id}"
10. 用户在1小时内发送过好友距离提醒：“Notify#{id}”
11. 用户在12小时内获得过脸脸全部帮助菜单：“HELP#{uid}”
12. 用户在8小时内发图通知过粉丝：“PhotoFan#{uid}”
13. 用户在聊天室发图2个小时内通知一次现场的人：“PhotoRoom#{uid}”
14. 登录商家后台出错，根据首次登录ip判断容许错误的次数：“LE#{错误次数};#{容许错误的次数}”
15. 优惠券重发: "CD#{coupon_down.id}"
16. 用户在最近6小时是否参与了抢地主活动："JOIN-LORD#{uid}"。
17. 签到微博的转发并评论次数"SINAADVT"
18. 新用户用手机号码注册"PHONEREG#{user.id}", 用于路行团带来的用户，首次定位时旅行团排第一位。
19. 新用户"NEWUSER#{user.id}", 用于新用户陪聊的判断，防止一个用户在不同的地点收到重复的陪聊信息。
20. 预置图片的出现次数和时间 “preset_value” => {0;Time.now.to_i}。 预置图片没7天更新一次

http://blog.csdn.net/neosmith/article/details/46811089
