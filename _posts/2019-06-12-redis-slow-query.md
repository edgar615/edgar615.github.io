---
layout: post
title: redis����ѯ
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-slow-query.html
---

Redis�ͻ���ִ��һ�������Ϊ4�����֣�

1. ��������
2. �����Ŷ�
3. ����ִ��
4. ���ؽ��

����ѯֻͳ�Ʋ���3��ʱ�䣬����û������ѯ��������ͻ���û�г�ʱ����

# ������

```
	# The following time is expressed in microseconds, so 1000000 is equivalent to one second. Note that a negative number disables the slow log, while a value of zero forces the logging of every command.
	# ��ֵ����λ΢�룬Ĭ��ֵ1�룬Ϊ0ʱ���¼�������Ϊ-1ʱ�����¼��������
	slowlog-log-slower-than 10000 
	
	# ���洢����������ѯ��־��Ĭ��ֵ128��redisʹ��һ��list���ʹ洢��־
	# There is no limit to this length. Just be aware that it will consume memory. You can reclaim memory used by the slow log with SLOWLOG RESET.
	slowlog-max-len 128 
```

# ����
**��������ѯ**

```
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 5
OK
127.0.0.1:6379> CONFIG REWRITE
```

**��ѯ����ѯ**  `slowlog get [n]` ����nָ������

```
127.0.0.1:6379> slowlog get 3
1) 1) (integer) 3 //��ʶID
   2) (integer) 1510128238 //����ʱ���
   3) (integer) 38 //�����ʱ
   4) 1) "slowlog" //ִ������
	  2) "get"
	  3) "2"
   5) "127.0.0.1:42638" //�ͻ���
   6) ""
2) 1) (integer) 2
   2) (integer) 1510128181
   3) (integer) 3
   4) 1) "CONFIG"
	  2) "REWRITE"
   5) "127.0.0.1:42638"
   6) ""
3) 1) (integer) 1
   2) (integer) 1510128172
   3) (integer) 8
   4) 1) "config"
	  2) "set"
	  3) "slowlog-max-len"
	  4) "5"
   5) "127.0.0.1:42638"
   6) ""
```

**����ѯ�б���** `slowlog len`

```
127.0.0.1:6379> slowlog len
(integer) 5
```

**����ѯ����** `slowlog reset`

```
127.0.0.1:6379> slowlog reset
OK
127.0.0.1:6379> slowlog len
(integer) 1
```

# ���ý���

- slowlog-log-slower-than ������10���룬�����ݲ���������
- slowlog-max-len 1000 ���Ͽ��Ե����б��ȣ���Ϊredis���Զ��ضϹ����ĳ��ȣ�����ռ�ù����ڴ�
	
����ѯֻ��¼����ִ��ʱ�䣬�������������ŶӺ����紫��ʱ�䣬��˿ͻ���ִ�������ʱ����������ʵ��ִ��ʱ�䡣
��Ϊ����ִ���Ŷӻ��ƣ�����ѯ�ᵼ�������������������˵��ͻ��˳�������ʱʱ����Ҫ����ʱ����Ƿ��ж�Ӧ������ѯ��

��������ѯ��־��һ���Ƚ��ȳ����У�������ѯ�Ƚ϶������£����ܻᶪʧ��������ѯ���Ϊ�˷�ֹ����������������Զ���ִ��slow get�������ѯ��־�־û��������洢��

# �ο�����

��Redis��������ά��
