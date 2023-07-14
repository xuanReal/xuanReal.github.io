---
layout: post
title: Redis 一些很实用的场景
date: 2022-12-06
categories: blog
description: 文章金句。
---

Redis 最最常用的就是用来做缓存了，但其实它还能在很多场景下使用。其每一种数据结构都有自己对应的用途。

### String

#### 分布式锁

```shell
127.0.0.1:6379> set user:01 1 ex 10 nx
OK			// 加锁成功，10s后过期
127.0.0.1:6379> set user:01 1 ex 10 nx
(nil)		// 加锁失败
127.0.0.1:6379> del user:01
(integer) 1	// 删除锁成功
127.0.0.1:6379> set user:01 1 ex 10 nx
OK			// 加锁成功
```

#### PV统计（incr自增计数）

Page View（PV）指的是页面浏览量，是用来衡量流量的一个重要标准，也是数据分析很重要的一个依据；统计规则一般是页面被展示一次，就加一。

> 涉及到的命令
> 1. incr：将 key 中储存的数字值增一
> 2. get：获取某个 key 的值

```shell
127.0.0.1:6379> incr pv:page1
(integer) 1
127.0.0.1:6379> incr pv:page1
(integer) 2
127.0.0.1:6379> incr pv:page1
(integer) 3
127.0.0.1:6379> get pv:page1
"3"
127.0.0.1:6379>
```
### List

有序可重复的列表。

#### 作为堆栈使用

> 涉及到的命令
> 1. lpush：从左侧插入列表
> 2. lpop：从列表左侧取出
> 3. rpush：从右侧插入列表
> 4. rpop：从列表右侧取出

1. 堆，先进先出

```shell
127.0.0.1:6379> rpush heap:01 a
(integer) 1
127.0.0.1:6379> rpush heap:01 b
(integer) 2
127.0.0.1:6379> rpush heap:01 c
(integer) 3
127.0.0.1:6379> lpop heap:01
"a"
127.0.0.1:6379> lpop heap:01
"b"
127.0.0.1:6379> lpop heap:01
"c"
127.0.0.1:6379> lpop heap:01
(nil)
127.0.0.1:6379>
```

2. 栈，先进后出

```shell
127.0.0.1:6379> rpush stack:01 a
(integer) 1
127.0.0.1:6379> rpush stack:01 b
(integer) 2
127.0.0.1:6379> rpush stack:01 c
(integer) 3
127.0.0.1:6379> rpop stack:01
"c"
127.0.0.1:6379> rpop stack:01
"b"
127.0.0.1:6379> rpop stack:01
"a"
127.0.0.1:6379> rpop stack:01
(nil)
127.0.0.1:6379>
```

### Set

无序不可重复的集合。

#### 抽奖

很多人都玩过抽奖，让一群用户参与进来，然后随机抽取几个幸运用户给予实物/虚拟的奖品。如果自己去写一个抽奖的算法，那肯定要花点时间；然而用 Redis 提供的 Set 集合，就能轻松实现。

> 涉及到的命令
> 1. sadd key member1 [member2]：添加一个或者多个元素
> 2. srand member KEY [count]：随机返回一个或者多个元素
> 3. spop key [count]：随机返回一个或者多个元素，并删除返回的元素

srandmember 和 spop 主要用于两种不同的抽奖模式，srandmember 适用于一个用户中奖后还可以参与后面的奖项，可以中多次；而 spop 就适用于仅能中一次的场景（一旦中奖，就将用户从用户池中移除，后续的抽奖，就不可能再抽到该用户）； 通常 spop 会用的会比较多。

```shell
127.0.0.1:6379> sadd lucky:draw u1 u2 u3 u4 u5
(integer) 5
127.0.0.1:6379> sadd lucky:draw u6 u7 u8 u9 u10
(integer) 5
127.0.0.1:6379> smembers lucky:draw
 1) "u4"
 2) "u6"
 3) "u3"
 4) "u2"
 5) "u8"
 6) "u1"
 7) "u7"
 8) "u10"
 9) "u5"
10) "u9"
127.0.0.1:6379> spop lucky:draw 2		// 随机抽两个
1) "u4"
2) "u5"
127.0.0.1:6379> spop lucky:draw			// 随机抽一个
"u6"
127.0.0.1:6379> smembers lucky:draw		// u4、u5、u6已不在
1) "u10"
2) "u1"
3) "u8"
4) "u9"
5) "u3"
6) "u2"
7) "u7"
127.0.0.1:6379>
```

#### 点赞/收藏

大部分社交 APP 都有点赞、收藏、喜欢等功能，来提升用户间的互动。

传统实现一般是用户点赞后在数据库插入一条数据，对应统计点赞数或判断用户是否已点赞过都需要频繁查询数据库，而且很不方便。

可以通过 Redis 的 Set 集合，很方便就能实现以上功能。

>涉及到的命令
>1. sadd key member1 [member2]：添加一个或者多个元素（点赞）
>2. scard key：获取集合中元素的数量（点赞数量）
>3. sismember key member：判断元素是否在集合内（是否点赞）
>4. srem key member1 [member2] ：移除一个或者多个元素（取消点赞）

```sh
127.0.0.1:6379> sadd like:post:01 u1 u2 u3
(integer) 3
127.0.0.1:6379> scard like:post:01			// 获取帖子的点赞数
(integer) 3
127.0.0.1:6379> sadd like:post:01 u4		// 点赞
(integer) 1
127.0.0.1:6379> scard like:post:01
(integer) 4
127.0.0.1:6379> sismember like:post:01 u1	// 判断用户是否点过赞
(integer) 1
127.0.0.1:6379> sismember like:post:01 u5
(integer) 0
127.0.0.1:6379> srem like:post:01 u2		// 取消点赞
(integer) 1
127.0.0.1:6379> scard like:post:01
(integer) 3
127.0.0.1:6379>
```

#### 可能认识的人/共同关注

抖音、支付宝等 APP 中经常会给我们推荐一些可能认识的人，被推荐的人一般来说和我们有共同的好友。

比如，A 和 B 是好友，B 和 C 是好友，此时 A 和 C 认识的概率是比较大的，就可以出现在 A 和 C 之间的好友推荐。通过 Set 求差集的方式，可以实现可能认识的人。

> 涉及到的命令：
> 1. sadd key member1 [member2]：添加一个或者多个元素（缓存好友列表）
> 2. sdiff key1 [key2]：求多个集合的差集，找出可以要推荐的用户

```shell
127.0.0.1:6379> sadd friend:01 zs ls			// 01的好友：zs、ls
(integer) 2
127.0.0.1:6379> sadd friend:02 ls ww			// 02的好友：ls、ww
(integer) 2
127.0.0.1:6379> sdiff friend:01 friend:02		// 02可能认识的人：zs
1) "zs"
127.0.0.1:6379> sdiff friend:02 friend:01		// 01可能认识的人：ww
1) "ww"
127.0.0.1:6379>
```

基于此又可以引申出一个新的功能：共同好友/共同关注。

>涉及到的命令：
>1. sadd key member1 [member2]：添加一个或者多个元素（缓存关注列表）
>2. sinter key1 [key2]：求多个集合的交集，找出可以共同关注的用户

```shell
127.0.0.1:6379> sadd follow:01 zs ls ww			// 01关注了：zs、ls、ww
(integer) 3
127.0.0.1:6379> sadd follow:02 ls ww zl			// 02的关注了：ls、ww、zl
(integer) 3
127.0.0.1:6379> sinter follow:01 follow:02		// 共同关注了：ww、ls
1) "ww"
2) "ls"
127.0.0.1:6379>
```

### zset

每个元素关联一个分数（score），可按分数进行排序。

#### 排行榜/热搜

很多应用会有个排行榜功能，用来刺激用户消费；或者出个热搜，让使用者了解当前热门资讯。

>涉及到的命令：
>1. zadd key score1 member1 [score2 member2]：添加并设置 scores，支持一次性添加多个
>2. zincrby key increment member ：在集合元素 member 的分值上加 increment
>3. zrange key start end：获得集合中指定区间成员，按分数递增排序
>4. zrevrange key start end：获得集合中指定区间成员，按分数递减排序

假设统计收礼物排行榜

```shell
127.0.0.1:6379> zincrby rank:gift 1 user01		// 模拟 user01 收到 1 礼物
"1"
127.0.0.1:6379> zincrby rank:gift 5 user02		// 模拟 user02 收到 5 礼物
"5"
127.0.0.1:6379> zincrby rank:gift 66 user02		// 模拟 user02 收到 66 礼物
"71"
127.0.0.1:6379> zincrby rank:gift 1314 user01	// 模拟 user01 收到 1314 礼物
"1315"
127.0.0.1:6379> zincrby rank:gift 999 user03	// 模拟 user03 收到 999 礼物
"999"
127.0.0.1:6379> zrevrange rank:gift 0 -1		// 按收礼物数降序排序，不带分数
1) "user01"
2) "user03"
3) "user02"
127.0.0.1:6379> zrevrange rank:gift 0 -1 withscores	// 按收礼物数降序排序，带分数
1) "user01"
2) "1315"
3) "user03"
4) "999"
5) "user02"
6) "71"
127.0.0.1:6379> zrevrange rank:gift 0 1			// 只展示前两名
1) "user01"
2) "user03"
```

### geo

#### 附近的人

很多交友软件都有附近的人这个功能，通过 Redis 的 geo 也能很快实现。

>涉及到的命令：
>1. geoadd key 经度1 纬度1 成员名称1 经度2 纬度2 成员名称2 ：添加地理坐标
>2. geopos key 成员名称1 成员名称2：返回成员经纬度
>3. geodist key 成员1 成员2 单位：计算成员间距离
>4. georadiusbymember key 成员 值单位 count 数 asc[desc]：根据成员查找附近的成员

```shell
127.0.0.1:6379> geoadd user:pos 113.950051 22.553722 u1 113.918287 22.543508 u2 113.960543 22.545911 u3					// 添加三个用户的坐标 u1、u2、u3
(integer) 3
127.0.0.1:6379> geopos user:pos u1		// 获取 u1 的坐标
1) 1) "113.95005315542221069"
   2) "22.55372201825422707"
127.0.0.1:6379> geopos user:pos u2
1) 1) "113.918285071849823"
   2) "22.54350709198208591"
127.0.0.1:6379> geopos user:pos u3
1) 1) "113.96054059267044067"
   2) "22.54591000764114739"
127.0.0.1:6379> geodist user:pos u1 u2	// 获取 u1 和 u2 之间的距离，单位默认 米
"3455.4599"
127.0.0.1:6379> geodist user:pos u3 u2 km	// 获取 u3 和 u2 之间的距离，单位设置为千米
"4.3490"
127.0.0.1:6379> geodist user:pos u3 u1 km
"1.3840"
127.0.0.1:6379> georadiusbymember user:pos u1 2 km withcoord withdist	// # 获得距离u1 2km以内的用户距离及经纬度
#withcoord：获得经纬度， withdist：获得距离 withhash：获得geohash码
1) 1) "u1"
   2) "0.0000"
   3) 1) "113.95005315542221069"
      2) "22.55372201825422707"
2) 1) "u3"
   2) "1.3840"
   3) 1) "113.96054059267044067"
      2) "22.54591000764114739"
```

### 总结

除了上面这些场景，其实它还可以应用在很多地方，大家可以去发掘。 
