---
layout: post
title: kafka生产者基准测试
date: 2020-02-21
categories:
    - kafka
comments: true
permalink: kafka-producer-benchmark.html
---

> 不同硬件环境测试不同，
>
> 以下数据是在三台4核4G的虚拟机上测试
>
> 因为目前内存不足，没有在独立测试机上发送数据，所以下面的测试可能有偏差

# 1. **1个分区，1个副本**

**准备数据**

```
bin/kafka-topics.sh --delete --bootstrap-server 192.168.159.131:9092 --topic test-p1-r1

bin/kafka-topics.sh --create --bootstrap-server 192.168.159.131:9092 --topic test-p1-r1 --partitions 1 --replication-factor 1
```

## 1.1. 消息长度的影响

**消息长度64**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 64  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092
2330499 records sent, 466099.8 records/sec (28.45 MB/sec), 724.4 ms avg latency, 967.0 ms max latency.
3000000 records sent, 518134.715026 records/sec (31.62 MB/sec), 709.04 ms avg latency, 967.00 ms max latency, 759 ms 50th, 942 ms 95th, 959 ms 99th, 965 ms 99.9th.
```

平均吞吐量是31MB/s，即占用248Mb/s左右的带宽，平均每秒能发送518134条消息，平均延时是709毫秒，最大延时是967毫秒，平均有50%的消息发送需要花费759秒，95%的消息发送需要花费942毫秒，99.9%的消息发送需要花费965毫秒。

**消息长度128**

 ```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 128  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092
1767144 records sent, 353428.8 records/sec (43.14 MB/sec), 558.4 ms avg latency, 740.0 ms max latency.
3000000 records sent, 325662.179766 records/sec (39.75 MB/sec), 664.65 ms avg latency, 1532.00 ms max latency, 658 ms 50th, 1403 ms 95th, 1517 ms 99th, 1529 ms 99.9th.
 ```

平均吞吐量是39MB/s，即占用320Mb/s左右的带宽，平均每秒能发送325662条消息，平均延时是664毫秒，最大延时是1532毫秒，平均有50%的消息发送需要花费658秒，95%的消息发送需要花费1403毫秒，99.9%的消息发送需要花费1529毫秒。

**消息长度 256**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 256  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092
560073 records sent, 111635.0 records/sec (27.25 MB/sec), 757.2 ms avg latency, 2228.0 ms max latency.
987346 records sent, 197469.2 records/sec (48.21 MB/sec), 752.1 ms avg latency, 2387.0 ms max latency.
1405684 records sent, 281136.8 records/sec (68.64 MB/sec), 444.7 ms avg latency, 575.0 ms max latency.
3000000 records sent, 196695.515342 records/sec (48.02 MB/sec), 603.76 ms avg latency, 2387.00 ms max latency, 452 ms 50th, 1825 ms 95th, 2299 ms 99th, 2373 ms 99.9th.
```

平均吞吐量是48MB/s，即占用384Mb/s左右的带宽，平均每秒能发送196695条消息，平均延时是603毫秒，最大延时是2387毫秒，平均有50%的消息发送需要花费452秒，95%的消息发送需要花费1825毫秒，99.9%的消息发送需要花费2373毫秒。

**消息长度512**

```
 bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092
398847 records sent, 79705.6 records/sec (38.92 MB/sec), 646.9 ms avg latency, 929.0 ms max latency.
409045 records sent, 81809.0 records/sec (39.95 MB/sec), 768.7 ms avg latency, 1135.0 ms max latency.
268615 records sent, 53723.0 records/sec (26.23 MB/sec), 1206.7 ms avg latency, 2028.0 ms max latency.
334459 records sent, 66891.8 records/sec (32.66 MB/sec), 941.7 ms avg latency, 1112.0 ms max latency.
330894 records sent, 66178.8 records/sec (32.31 MB/sec), 973.0 ms avg latency, 1339.0 ms max latency.
367970 records sent, 73594.0 records/sec (35.93 MB/sec), 866.0 ms avg latency, 1370.0 ms max latency.
494698 records sent, 98939.6 records/sec (48.31 MB/sec), 661.1 ms avg latency, 1156.0 ms max latency.
3000000 records sent, 78159.601907 records/sec (38.16 MB/sec), 797.68 ms avg latency, 2028.00 ms max latency, 778 ms 50th, 1294 ms 95th, 1666 ms 99th, 2011 ms 99.9th.
```

平均吞吐量是38MB/s，即占用304Mb/s左右的带宽，平均每秒能发送78159条消息，平均延时是797毫秒，最大延时是2028毫秒，平均有50%的消息发送需要花费778秒，95%的消息发送需要花费1294毫秒，99.9%的消息发送需要花费2011毫秒。

**短消息对Kafka来说是更难处理的使用方式，可以预期，随着消息长度的增大，records/second会减小，但MB/second会有所提高**

随着消息长度的增加，每秒钟所能发送的消息的数量逐渐减小。但是如果看每秒钟发送的消息的总大小，它会随着消息长度的增加而增加

## 1.2. 压缩的影响

**消息长度64**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 64  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=gzip
3000000 records sent, 685244.403837 records/sec (41.82 MB/sec), 21.46 ms avg latency, 276.00 ms max latency, 4 ms 50th, 140 ms 95th, 178 ms 99th, 180 ms 99.9th.

# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 64  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy
3000000 records sent, 1128243.700639 records/sec (68.86 MB/sec), 46.14 ms avg latency, 360.00 ms max latency, 6 ms 50th, 291 ms 95th, 336 ms 99th, 357 ms 99.9th.
```

**消息长度128**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 128  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=gzip
2897896 records sent, 579579.2 records/sec (70.75 MB/sec), 8.4 ms avg latency, 255.0 ms max latency.
3000000 records sent, 579710.144928 records/sec (70.77 MB/sec), 8.30 ms avg latency, 255.00 ms max latency, 3 ms 50th, 37 ms 95th, 61 ms 99th, 67 ms 99.9th.

# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 128  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy
3000000 records sent, 863060.989643 records/sec (105.35 MB/sec), 171.40 ms avg latency, 528.00 ms max latency, 139 ms 50th, 468 ms 95th, 509 ms 99th, 526 ms 99.9th.
```

**消息长度256**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 256  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=gzip
1504280 records sent, 300856.0 records/sec (73.45 MB/sec), 41.6 ms avg latency, 387.0 ms max latency.
3000000 records sent, 334970.969183 records/sec (81.78 MB/sec), 25.82 ms avg latency, 387.00 ms max latency, 4 ms 50th, 128 ms 95th, 346 ms 99th, 370 ms 99.9th.

# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 256  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy
3000000 records sent, 740923.684860 records/sec (180.89 MB/sec), 693.85 ms avg latency, 970.00 ms max latency, 693 ms 50th, 919 ms 95th, 945 ms 99th, 969 ms 99.9th.
```

**消息长度512**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=gzip
1156725 records sent, 231345.0 records/sec (112.96 MB/sec), 16.0 ms avg latency, 258.0 ms max latency.
1358258 records sent, 271651.6 records/sec (132.64 MB/sec), 6.6 ms avg latency, 39.0 ms max latency.
3000000 records sent, 254065.040650 records/sec (124.06 MB/sec), 10.16 ms avg latency, 258.00 ms max latency, 4 ms 50th, 35 ms 95th, 101 ms 99th, 159 ms 99.9th.

# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy
1751561 records sent, 349822.4 records/sec (170.81 MB/sec), 1282.6 ms avg latency, 1632.0 ms max latency.
3000000 records sent, 446428.571429 records/sec (217.98 MB/sec), 1174.55 ms avg latency, 1632.00 ms max latency, 1110 ms 50th, 1590 ms 95th, 1620 ms 99th, 1631 ms 99.9th.
```

可以看到，开启压缩对吞吐量的提升非常明显

## 1.3. batch.size的影响

**4000**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy batch.size=4000
411017 records sent, 82203.4 records/sec (40.14 MB/sec), 2416.6 ms avg latency, 3036.0 ms max latency.
1543817 records sent, 308763.4 records/sec (150.76 MB/sec), 2079.2 ms avg latency, 2797.0 ms max latency.
3000000 records sent, 232378.001549 records/sec (113.47 MB/sec), 2018.50 ms avg latency, 3036.00 ms max latency, 1879 ms 50th, 2913 ms 95th, 3015 ms 99th, 3030 ms 99.9th.
```

**8192**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy batch.size=8192
1447257 records sent, 289046.7 records/sec (141.14 MB/sec), 1419.3 ms avg latency, 1705.0 ms max latency.
3000000 records sent, 410004.100041 records/sec (200.20 MB/sec), 1231.90 ms avg latency, 1705.00 ms max latency, 1241 ms 50th, 1647 ms 95th, 1691 ms 99th, 1703 ms 99.9th.
```

**20000**

```
# bin/kafka-producer-perf-test.sh --print-metrics --num-records 3000000 --record-size 512  --topic test-p1-r1 --throughput -1 --producer-props bootstrap.servers=192.168.159.131:9092,192.168.159.132:9092,192.168.159.133:9092 compression.type=snappy batch.size=20000
3000000 records sent, 658761.528327 records/sec (321.66 MB/sec), 557.52 ms avg latency, 854.00 ms max latency, 647 ms 50th, 820 ms 95th, 839 ms 99th, 851 ms 99.9th.
```

使用批处理对吞吐量提升明显

这个参数的默认值是16KB，一般可以尝试把这个参数调节大一些，然后利用自己的生产环境发消息的负载来测试一下。

比如说发送消息的频率就是每秒300条，那么如果比如“batch.size”调节到了32KB，或者64KB，是否可以提升发送消息的整体吞吐量。

因为理论上来说，提升batch的大小，可以允许更多的数据缓冲在里面，那么一次Request发送出去的数据量就更多了，这样吞吐量可能会有所提升。

但是这个东西也不能无限的大，过于大了之后，要是数据老是缓冲在Batch里迟迟不发送出去，那么岂不是你发送消息的延迟就会很高。

比如说，一条消息进入了Batch，但是要等待5秒钟Batch才凑满了64KB，才能发送出去。那这条消息的延迟就是5秒钟。

所以需要在这里按照生产环境的发消息的速率，调节不同的Batch大小自己测试一下最终出去的吞吐量以及消息的 延迟，设置一个最合理的参数。

> 未完成





