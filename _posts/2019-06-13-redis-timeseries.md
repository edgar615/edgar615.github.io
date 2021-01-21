---
layout: post
title: redis TimeSeries
date: 2019-06-13
categories:
    - redis
comments: true
permalink: redis-timeseries.html
---

RedisTimeSeries模块支持存储时序数据。

https://oss.redislabs.com/redistimeseries/

**安装**

```
git clone --recursive https://github.com/RedisTimeSeries/RedisTimeSeries.git
cd RedisTimeSeries
make setup
make build
```

**启动**

```
# https://redis.io/topics/modules-intro
# 方式1 命令行
module load /server/RedisTimeSeries/bin/redistimeseries.so
# 方式2 redis.conf
loadmodule /server/RedisTimeSeries/bin/redistimeseries.so
```

RedisTimeSeries 的操作主要有 5 个：

- 用 TS.CREATE 命令创建时间序列数据集合；
- 用 TS.ADD 命令插入数据；
- 用 TS.GET 命令读取最新数据；
- 用 TS.MGET 命令按标签过滤查询数据集合；
- 用 TS.RANGE 支持聚合计算的范围查询。

 **用 TS.CREATE 命令创建一个时间序列数据集合**

```
TS.CREATE device:temperature RETENTION 600000 LABELS device_id 1
```

上面的命令，创建一个 key 为 `device:temperature`、数据有效期为 600s 的时间序列数据集合。也就是说，这个集合中的数据创建了 600s 后，就会被自动删除。最后，我们给这个集合设置了一个标签属性{device_id:1}，表明这个数据集合中记录的是属于设备 ID 号为 1 的数据。

标签可以设置多个

```
TS.CREATE temperature:3:11 RETENTION 60 LABELS sensor_id 2 area_id 32
```

**用 TS.ADD 命令插入数据**

```
TS.ADD device:temperature 1596416700 25.1
(integer) 1596416700
```

**用 TS.GET 命令读取最新数据**

```
TS.GET device:temperature
1) (integer) 1596416700
2) 25.1
```

**用 TS.RANGE 支持需要聚合计算的范围查询**

在对时间序列数据进行聚合计算时，我们可以使用 TS.RANGE 命令指定要查询的数据的时间范围，同时用 AGGREGATION 参数指定要执行的聚合计算类型。RedisTimeSeries 支持的聚合计算类型很丰富，包括求均值（avg）、求最大 / 最小值（max/min），求和（sum）等。

```
127.0.0.1:6379> TS.ADD device:temperature 1596416710 30
(integer) 1596416710
127.0.0.1:6379> TS.ADD device:temperature 1596416890 20.5
(integer) 1596416890
127.0.0.1:6379> TS.ADD device:temperature 1596416711 28.4
(integer) 1596416711
# 按照每个5秒毫秒的时间进行聚合
127.0.0.1:6379> TS.RANGE device:temperature 1596416701 1596416899 AGGREGATION avg 5
1) 1) (integer) 1596416710
   2) 29.2
2) 1) (integer) 1596416890
   2) 20.5
127.0.0.1:6379> TS.RANGE device:temperature 1596416701 -1 AGGREGATION avg 5
1) 1) (integer) 1596416710
   2) 29.2
2) 1) (integer) 1596416890
   2) 20.5

```

**用 TS.MGET 命令按标签过滤查询数据集合**

在保存多个设备的时间序列数据时，我们通常会把不同设备的数据保存到不同集合中。此时，我们就可以使用 TS.MGET 命令，按照标签查询部分集合中的最新数据。在使用 TS.CREATE 创建数据集合时，我们可以给集合设置标签属性。当我们进行查询时，就可以在查询条件中对集合标签属性进行匹配，最后的查询结果里只返回匹配上的集合中的最新数据。

举个例子。假设我们一共用 4 个集合为 4 个设备保存时间序列数据，设备的 ID 号是 1、2、3、4，我们在创建数据集合时，把 device_id 设置为每个集合的标签。此时，我们就可以使用下列 TS.MGET 命令，以及 FILTER 设置（这个配置项用来设置集合标签的过滤条件），查询 device_id 不等于 2 的所有其他设备的数据集合，并返回各自集合中的最新的一条数据。

```
127.0.0.1:6379> TS.CREATE device:temperature:1 RETENTION 600000 LABELS device_id 1
OK
127.0.0.1:6379> TS.CREATE device:temperature:2 RETENTION 600000 LABELS device_id 2
OK
127.0.0.1:6379> TS.CREATE device:temperature:3 RETENTION 600000 LABELS device_id 3
OK
127.0.0.1:6379> TS.ADD device:temperature:1 1596416710 30
(integer) 1596416710
127.0.0.1:6379> TS.ADD device:temperature:2 1596416710 28
(integer) 1596416710
127.0.0.1:6379> TS.ADD device:temperature:3 1596416710 19
(integer) 1596416710
127.0.0.1:6379> TS.MGET FILTER device_id=2
1) 1) "device:temperature:2"
   2) (empty array)
   3) 1) (integer) 1596416710
      2) 28
127.0.0.1:6379> TS.MGET WITHLABELS FILTER device_id=2
1) 1) "device:temperature:2"
   2) 1) 1) "device_id"
         2) "2"
   3) 1) (integer) 1596416710
      2) 28
127.0.0.1:6379> TS.MGET WITHLABELS FILTER device_id=(1,2)
1) 1) "device:temperature:1"
   2) 1) 1) "device_id"
         2) "1"
   3) 1) (integer) 1596416710
      2) 30
2) 1) "device:temperature:2"
   2) 1) 1) "device_id"
         2) "2"
   3) 1) (integer) 1596416710
      2) 28

```



更多用法参考https://oss.redislabs.com/redistimeseries/commands