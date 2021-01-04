---
layout: post
title: 限流（3）- Redis
date: 2020-04-17
categories:
    - 架构设计
comments: true
permalink: ratelimit-redis.html
---

# 1. 简单实现

redis官网介绍了用incr命令实现RateLimit的两种方法

## 1.1. 模式1

```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)
IF current != NULL AND current > 10 THEN
	ERROR "too many requests per second"
ELSE
	MULTI
		INCR(keyname,1)
		EXPIRE(keyname,10)
	EXEC
	PERFORM_API_CALL()
END
```

简单来说，我们对每个IP的每一秒都有一个计数器，但每个计数器都有一个额外的设置：它们都将被设置一个10秒的过期时间。这可以使得当时间已经不是当前秒时（此时该计数器也无效了），能够让redis自动移除它。

需要注意的是，这里我们使用multi和exec命令来确保对每个API调用既执行了incr也同时能够执行expire命令。

	multi命令用于标识一个命令集被包含在一个事务块中，exec保证该事务块命令集执行的原子性。
## 1.2. 模式2

另外的一种实现是采用单一的计数器，但是为了避免race condition（竞态条件），它也更复杂。我们来看几种不同的变体：

### 1.2.1. 单一计数器

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
	ERROR "too many requests per second"
ELSE
	value = INCR(ip)
	IF value == 1 THEN
		EXPIRE(value,1)
	END
	PERFORM_API_CALL()
END
```

该计数器在当前秒内第一次请求被执行时创建，但它只能存活一秒。如果在当前秒内，发送超过10次请求，那么该计数器将超过10。否则它将失效并从0开始重新计数。

在上面的代码中，存在一个race condition。如果因为某个原因，上面的代码只执行了incr命令，却没有执行expire命令，那么这个key将会被泄漏，直到我们再次遇到相同的ip

这种问题也不难处理，可以将incr命令以及另外的expire命令打包到一个lua脚本里，该脚本可以用eval命令提交给redis执行（该方式只在redis版本大于等于2.6之后才能支持）

```
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
	redis.call("expire",KEYS[1],1)
end
```

### 1.2.2. list

当然，也有另一种方式来解决这个问题而不需要动用lua脚本，但需要用redis的list数据结构来替代计数器。这种实现方式将会更复杂，并使用更高级的特性。但它有一个好处是记住调用当前API的每个客户端的IP。这种方式可能很有用也可能没用，这取决于应用需求。

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
	ERROR "too many requests per second"
ELSE
	IF EXISTS(ip) == FALSE
		MULTI
			RPUSH(ip,ip)
			EXPIRE(ip,1)
		EXEC
	ELSE
		RPUSHX(ip,ip)
	END
	PERFORM_API_CALL()
END
```

**rpushx命令只在key存在时才会将值加入list**

仍然需要注意的是，这里也存在一个race condition（但这却不会产生太大的影响）。问题是：exists可能返回false，但在我们执行multi/exec块内的创建list的代码之前，该list可能已被其他客户端创建。然而，在这个race condition发生时，将仅仅只是丢失一个API调用，所以rate limiting仍然工作得很好。

    这里产生race condition不会有大问题的原因在于，else分支使用的rpushx，它不会导致if not than init的问题，并且expire命令将在创建list的时候以原子的形式捆绑执行。不会产生key泄漏，导致永不失效的情况产生。

# 2. 固定窗口算法Fixed window counters

在固定窗口（Fixed Window）算法中，使用n秒的窗口大小（通常使用对人类友好的值，如60秒或3600秒）来跟踪速率。每个传入的请求都会增加窗口的计数器。如果计数器超过阈值，则丢弃请求。窗口通常由当前时间戳的层定义，因此12:00:03的窗口长度为60秒，应该在12:00:00的窗口中。

## 2.1. 示例

lua脚本

```
	local key = "rate.limit:" .. KEYS[1] --限流KEY
	local limit = tonumber(ARGV[1]) --限流大小
	local expire_time = ARGV[2] --过期时间
	local current = tonumber(redis.call('get', key) or "0")
	if current + 1 > limit then --超出限流大小
	  return 0
	else --请求数+1，并设置expire_time秒之后过期
	  redis.call("INCRBY", key, "1")
	  redis.call("EXPIRE", key, expire_time)
	  return 1
	end
```

java代码

```java
	private boolean accessLimit(String ip, int limit, int timeout,
	                          Jedis connection) throws IOException {
	List<String> keys = Collections.singletonList(ip);
	List<String> argv = Arrays.asList(String.valueOf(limit), String.valueOf(timeout));
	
	return 1 == (Long) connection.eval(loadScriptString("limit.lua"), keys, argv);
	}
	
	// 加载Lua代码
	private String loadScriptString(String fileName) throws IOException {
	Reader reader =
	        new InputStreamReader(LimitTest.class.getClassLoader().getResourceAsStream(fileName));
	return CharStreams.toString(reader);
	}
```

测试

```
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
TimeUnit.SECONDS.sleep(3);
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
TimeUnit.SECONDS.sleep(2);
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
```

输入如下：

```
true
true
false
true
true
false
```

## 2.2. 完整代码1：单个限流规则

```
local key = "rate.limit." .. ARGV[1] --限流KEY
local limit = tonumber(ARGV[2]) --限流窗口允许的最大请求数
local interval = ARGV[3] --限流间隔,秒
local now = ARGV[4] --当前Unix时间戳
--计算key
local subject = math.floor(now / interval)
key = key .. '.' .. subject .. '.' .. limit .. '.' .. interval

local current = tonumber(redis.call("incr", key))
if current == 1 then
	redis.call("expire", key, interval)
end

local ttl = redis.call("ttl", key)
local remaining = math.max(limit - current, 0);
--返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
return {limit - current >= 0,  remaining, limit, ttl}
```

## 2.3. 完整代码2：多个限流规则组合

```
local rate_limits = cjson.decode(ARGV[1])
local now = tonumber(ARGV[2])

local prefix_key = "rate.limit." --限流KEY
local result = {}
local passed = true;
--计算请求是否满足每个限流规则
for i, rate_limit in ipairs(rate_limits) do
	local limit = rate_limit[2]
	local interval = rate_limit[3]
	local subject = math.floor(now / interval)
	local limit_key =  prefix_key .. rate_limit[1] .. '.' .. subject .. '.' .. limit .. '.' .. interval
	local requested_num = tonumber(redis.call('GET', limit_key)) or 0 --请求数，默认为0
	if requested_num >= limit then
		passed = false
		--依次为　限流key,限流大小,限流间隔,剩余请求数,是否通过
		table.insert(result, { limit_key, limit, interval,  math.max(limit - requested_num, 0), 0})
	else
		table.insert(result, { limit_key, limit, interval,  math.max(limit - requested_num, 0), 1 })
	end
end

local summary = { }
for key,value in ipairs(result) do
	local limit_key = value[1]
	local limit = value[2]
	local interval = value[3]
	if passed then
		--如果通过, 将所有的限流请求数加1
		local current = tonumber(redis.call("incr", limit_key))
		if current == 1 then
			redis.call("expire", limit_key, interval)
		end
		--添加两个值　是否通过0或1　剩余请求数
		table.insert(summary, 1)
		table.insert(summary, math.max(limit - current, 0))
	else
		local pass = value[5]
		if not pass or pass == 0 then
			table.insert(summary, 0)
			table.insert(summary, 0)
		else
			table.insert(summary, 1)
			table.insert(summary, value[4])
		end
	end
	local ttl = redis.call("ttl", limit_key)
	--添加两个值　最大请求数　重置时间
	table.insert(summary, limit)
	table.insert(summary, ttl)
end

--返回值为: 是否通过0或1, 剩余令牌, 最大请求数, 限流窗口重置时间 ,...
return summary
```

这种算法的优点是，它可以确保处理更多最近的请求，而不会被旧的请求饿死。然而，发生在窗口边界附近的单个流量突发会导致处理请求的速度增加一倍，因为它允许在短时间内同时处理当前窗口和下一个窗口的请求。另外，如果许多消费者等待一个重置窗口，例如在一个小时的顶部，那么他们可能同时扰乱你的API。

## 2.4. **时间边界问题**

上述Redis的实现在实际应用中存在本篇最开始时候描述的问题**“没有很好的处理单位时间的边界”**。假如一个接口每小时值允许240次调用。如果调用方在6:59PM调用200次请求之后，在7:00PM可以再次调用240次请求，再两分钟之内这个接口就被调用了440次。
针对上述的情况，只有一个计数器是不够的。我们需要将整个流量控制（1小时240次调用）看做一个大的计数桶，然后将这个大的桶拆分成一堆小桶，在每个小桶里都有自己的个性计数。我们可以使用1分钟、5分钟或者15分钟的小桶来拆分1小时的桶（这取决于系统需求，更小的桶意味着更多的内存和清理工作）。
例如我们将1小时的水桶拆分为了1分钟的小桶，那么我们会记录6:00PM,6:01PM,6:02PM的调用次数。但当时间变为7:00PM时，我们需要将6:00PM的桶重置为0，并重新标记桶7:00PM。在7:01PM时会对6:01PM和7:01PM的桶做同样的操作。但是这种实现太复杂，很难做到复杂的限制规则。

# 3. 滑动日志算法Sliding window log

滑动日志（Sliding Log）速率限制涉及到跟踪每个使用者请求的时间戳日志。这些日志通常存储在按时间排序的散列集或表中。时间戳超过阈值的日志将被丢弃。当新请求出现时，我们计算日志的总和来确定请求率。如果请求将超过阈值速率，则保留该请求。

<table>
	<thead>
		<tr>
			<th>索引</th>
			<td>0</td>
			<td>1</td>
			<td>2</td>
			<td>3</td>
			<td>4</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>时间戳</th>
			<td>12:34:28</td>
			<td>12:34:26</td>
			<td>12:34:14</td>
			<td>12:33:37</td>
			<td>12:33:35</td>
		</tr>
	</tbody>
</table>

假如我们有两个限流规则：每秒1次请求和每分钟5次请求。那么只需要关注最近的第一次请求和第五次请求即可。如果第一个请求距离现在还未过去1秒，那么请求就无法通过每秒1次请求的规则。同理，如果第五个请求距离现在还未过去1分钟，那么请求就无法通过每分钟5次请求的规则。

在12:34:31的时候第一个请求已经过去了3秒钟，但是第5个请求才过去了56秒钟，所以请求依然无法通过每分钟5次请求的规则。我们必须等到4秒钟之后重试。
在12:34:40的时候第一个请求已经过去了12秒钟，第5个请求已经过去了65秒钟，这是我们就可以允许请求通过，并将请求追加到list中。

如果没有其他的限流规则，那么我们永远不会检查第5个请求后的请求，这时我们可以将整个列表的长度控制在5个元素。同时如果我们最大的限流长度是1分钟，我们还可以将list设置为60秒后过期。

在追加一个新的请求之后，list会变为

<table>
	<thead>
		<tr>
			<th>索引</th>
			<td>0</td>
			<td>1</td>
			<td>2</td>
			<td>3</td>
			<td>4</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>时间戳</th>
			<td>12:34:40</td>
			<td>12:34:28</td>
			<td>12:34:26</td>
			<td>12:34:14</td>
			<td>12:33:37</td>
		</tr>
	</tbody>
</table>
lua代码

```
	--两种规则，1秒钟1次请求，每分钟5次请求，为简单起见，规则未使用参数传入
	local current_timestamp = tonumber(ARGV[1]) -- 当前请求的时间
	local key = "rate.limit2:" .. KEYS[1] --限流KEY
	
	--删除列表中超过1分钟的请求
	redis.pcall('ZREMRANGEBYSCORE', key, 0, current_timestamp - 60)
	
	local passed1 = true
	local passed2 = true
	--获取第一个请求
	local last_drip = redis.pcall('ZREVRANGEBYSCORE', key, '+inf', '-inf', 'LIMIT', 0, 1)
	
	--比较上次的时间戳和当前请求的时间戳
	if last_drip[1] then
	    --上次请求的时间
	    local last_time = tonumber(last_drip[1])
	    passed1 = current_timestamp >=  (last_time + 1)
	end
	
	--获取前5个请求
	local last_drip = redis.pcall('ZREVRANGEBYSCORE', key, '+inf', '-inf', 'LIMIT', 0, 5)
	
	--比较上次的时间戳和当前请求的时间戳
	if last_drip[5] then
	    --第5次请求的时间
	    local last_time = tonumber(last_drip[5])
	    passed2  = current_timestamp >=  (last_time + 60)
	end
	
	if passed1 and passed2 then
	  redis.pcall('ZADD', key, current_timestamp, current_timestamp)
	end
	--设置1分钟过期
	redis.call('EXPIRE', key, 60)
	return {current_timestamp,last_drip, passed1, passed2}
```

测试代码

```
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(61);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
```

输出

```
	[1484551710, [], 1, 1]
	[1484551710, [1484551710], null, 1]
	[1484551711, [1484551710], 1, 1]
	[1484551712, [1484551711, 1484551710], 1, 1]
	[1484551713, [1484551712, 1484551711, 1484551710], 1, 1]
	[1484551714, [1484551713, 1484551712, 1484551711, 1484551710], 1, 1]
	[1484551715, [1484551714, 1484551713, 1484551712, 1484551711, 1484551710], 1, null]
	[1484551776, [], 1, 1]
```

通过输出我们可以看到:

- 第1个请求通过
- 第2个请求没有通过每秒1次请求的规则
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后没有通过每分钟5次请求的规则
- 等待一分钟后请求通过

**但是这个方法在并发大的情况下会对内存有较高的要求，例如每个用户一个队列，每个队列1000个字节，如果有100万个用户的限流规则，那么就会占用1000*100W≈1G的内存**。

参考 https://blog.domaintools.com/2013/04/rate-limiting-with-redis/
https://engineering.classdojo.com/blog/2015/02/06/rolling-rate-limiter/

# 4. 滑动窗口计数Sliding window counters

我们也可以通过将大桶拆分为小桶的方式来处理时间边界问题

```
local subject = "rate.limit." .. ARGV[1] --限流KEY
local limit = tonumber(ARGV[2]) --限流窗口允许的最大请求数
local interval = tonumber(ARGV[3]) --限流间隔,秒
local precision = tonumber(ARGV[4])
local now = tonumber(ARGV[5]) --当前Unix时间戳
--桶的大小不能超过限流的间隔
precision = math.min(precision, interval)

--重新计算限流的KEY，避免传入相同的key，不同的间隔导致冲突
subject = subject .. '.' .. limit .. '.' .. interval .. '.' .. precision

--桶的数量，与位置
local bucket_num = math.ceil(interval / precision)
local oldest_req_key = subject .. ':o'
local bucket_key = math.floor(now / precision)
local trim_before = bucket_key - bucket_num + 1 --需要删除的key
local oldest_req = tonumber(redis.call('GET', oldest_req_key)) or trim_before --最早请求时间，默认为0
trim_before = math.min(oldest_req, trim_before)
--判断当前桶是否存在
local current_bucket = redis.call("hexists", subject, bucket_key)
if current_bucket == 0 then
	redis.call("hset", subject, bucket_key, 0)
end


	
	--请求总数
	local max_req = 0;
	local subject_hash = redis.call("hgetall", subject) or {}
	local last_req = now; --最近的访问时间，计算重置窗口
	for i = 1, #subject_hash, 2 do
	    local ts_key = tonumber(subject_hash[i])
	    if ts_key < trim_before and #subject_hash > 2  then
	        redis.call("hdel", subject, ts_key)
	--        table.insert(dele, ts_key)
	    else
	        local req_num =tonumber(subject_hash[i + 1])
	        max_req = max_req +  req_num
	        if req_num ~= 0 then
	            last_req = math.min(last_req, math.floor(ts_key * precision))
	        end
	    end
	end
	local reset_time =interval - now+  last_req;
	
	if max_req >= limit then
	    --返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
	    return {0, 0, limit, reset_time}
	end
	
-- 当前请求+1
local current = tonumber(redis.call("hincrby", subject, bucket_key, 1))
-- interval+precision之后过期
redis.call("expire", subject, interval+precision)
redis.call("setex", oldest_req_key,interval+precision , now )

--返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
return {1,  limit - max_req - 1, limit, reset_time}

```
**多个滑动窗口组合**

```
	local rate_limits = cjson.decode(ARGV[1])
	local now = tonumber(ARGV[2])
	
	local prefix_key = "rate.limit." --限流KEY
	local result = {}
	local passed = true;
	--计算请求是否满足每个限流规则
	for i, rate_limit in ipairs(rate_limits) do
	    local limit = rate_limit[2]
	    local interval = rate_limit[3]
	    local precision = rate_limit[4]
	    --桶的大小不能超过限流的间隔
	    precision = math.min(precision, interval)
	
	    --重新计算限流的KEY，避免传入相同的key，不同的间隔导致冲突
	    local subject = prefix_key .. rate_limit[1] .. '.' .. limit .. '.' .. interval .. '.' .. precision
	
	    --桶的数量，与位置
	    local bucket_num = math.ceil(interval / precision)
	    local oldest_req_key = subject .. '.o'
	    local bucket_key = math.floor(now / precision)
	    local trim_before = bucket_key - bucket_num + 1 --需要删除的key
	    local oldest_req = tonumber(redis.call('GET', oldest_req_key)) or trim_before --最早请求时间，默认为0
	    trim_before = math.min(oldest_req, trim_before)
	    --判断当前桶是否存在
	    local current_bucket = redis.call("hexists", subject, bucket_key)
	    if current_bucket == 0 then
	        redis.call("hset", subject, bucket_key, 0)
	    end
	
	    --请求总数
	    local max_req = 0;
	    local subject_hash = redis.call("hgetall", subject) or {}
	    local last_req = now; --最近的访问时间，计算重置窗口
	    for i = 1, #subject_hash, 2 do
	        local ts_key = tonumber(subject_hash[i])
	        if ts_key < trim_before and #subject_hash > 2  then
	            redis.call("hdel", subject, ts_key)
	            --        table.insert(dele, ts_key)
	        else
	            local req_num =tonumber(subject_hash[i + 1])
	            max_req = max_req +  req_num
	            if req_num ~= 0 then
	                last_req = math.min(last_req, math.floor(ts_key * precision))
	            end
	        end
	    end
	    local reset_time =interval - now+  last_req;
	    if max_req >= limit then
	        passed = false
	        --依次为　限流key,限流大小,限流间隔,对应的桶,最大请求数,是否成功,重置时间
	        table.insert(result, { subject, limit, interval, bucket_key, max_req, 0, reset_time, precision,oldest_req_key})
	    else
	        table.insert(result, { subject, limit, interval, bucket_key, max_req, 1, reset_time, precision,oldest_req_key})
	    end
	
	end
	
	-- 如果通过，增加每个限流规则
	if passed == true then
	    for key,value in ipairs(result) do
	        -- 当前请求+1
	        local subject = value[1]
	        local interval = value[3]
	        local bucket_key = value[4]
	        local precision = value[8]
	        local oldest_req_key = value[9]
	        local current = tonumber(redis.call("hincrby", subject, bucket_key, 1))
	        -- interval+precision之后过期
	        redis.call("expire", subject, interval+precision)
	        redis.call("setex", oldest_req_key,interval+precision , now )
	    end
	end
	
	-- 返回结果
	local summary = {}
	for key,value in ipairs(result) do
	    local pass = value[6]
	    if not pass or pass == 0 then
	        --添加四个值　是否通过0或1　剩余请求数 最大请求数　重置时间
	        table.insert(summary, 0)
	        table.insert(summary, 0)
	        table.insert(summary, value[2])
	        table.insert(summary, value[7])
	    else
	        table.insert(summary, 1)
	        if passed == true then
	            table.insert(summary, value[2] - value[5] - 1)
	        else
	            table.insert(summary, value[2] - value[5])
	        end
	
	        table.insert(summary, value[2])
	        table.insert(summary, value[7])
	    end
	end
	
	return summary
```

# 5. 令牌桶 Token bucket
可以参考Guava的实现，基于redis实现自己的令牌桶，这个令牌桶有下列属性:

- key: 桶的标识符
- maxAmount: 桶中最大令牌数
- refillTime: 向桶中添加令牌的周期，单位毫秒
- refillAmount: 每次refillTime向桶中添加的令牌数量，默认为1
- value: 桶中可用的令牌数
- lastUpdate: 桶最近一次更新时间（refillTime）


```
local subject = ARGV[1]
local key = "token.bucket." .. subject --桶的标识符
local burst = math.max(tonumber(ARGV[2]), 1) --桶中最大令牌数，最小值1，
local refillTime = tonumber(ARGV[3]) or 1000-- 向桶中添加令牌的周期，单位毫秒
local refillAmount = math.max(tonumber(ARGV[4]), 1) or 1 -- 每次refillTime向桶中添加的令牌数量，默认为1
local tokens_to_take       = tonumber(ARGV[5]) or 1 --当前请求需要的令牌数量
local now = tonumber(ARGV[6]) --当前时间，单位毫秒

local available_tokens = burst --可用的令牌数默认等于桶的容量
local last_update = now --第一次的last_update等于当前时间
local current = redis.call('HMGET', key, 'last_update', 'available_tokens')
if current.err ~= nil then
	redis.call('DEL', key)
	current = {}
	redis.log(redis.LOG_NOTICE, 'Cannot get ratelimit ' .. key)
	return redis.error_reply('Cannot get ratelimit ' .. key)
end


		
		--计算从上次的时间戳与当前时间戳计算应该添加的令牌数
		if current[1] then
		    --上次请求的时间
		    last_update = current[1]
		    local content = current[2]
		    --计算应该生成的令牌数
		    local delta_ms = math.max(now - last_update, 0)
		    local refillCount  = math.floor(delta_ms / refillTime) * refillAmount
		    --如果桶满，直接使用桶的容量
		    available_tokens = math.min(content + refillCount, burst)
		end
		
		-- 计算是否有足够的令牌给调用方
		local enough_tokens = available_tokens >= tokens_to_take
		--之后的请求要偿还之前发生的突发
		
-- 将令牌给调用方之后，桶中剩余的令牌数
if enough_tokens then
	available_tokens = math.min(available_tokens - tokens_to_take, burst)
	last_update = last_update + math.floor(tokens_to_take / refillAmount * refillTime)
else
	redis.log(redis.LOG_NOTICE, 'Cannot get enough tokens, tokens_to_take:' .. tokens_to_take .. ',available_tokens:' .. available_tokens)
	--计算恢复的时间
	local reset_time = last_update - now
	if reset_time < 0 then
		reset_time = now - last_update
	end
	return { 0, math.max(available_tokens, 0), burst, reset_time}
end
--重新设置令牌桶
redis.call('HMSET', key,
	'last_update', last_update,
	'available_tokens', available_tokens)

--如果没有新的请求过来，在桶满之后可以直接将该令牌删除。
redis.call('PEXPIRE', key, math.ceil((burst /  refillAmount) * refillTime))

return {1,  math.max(available_tokens, 0), burst,last_update - now }
```

参考:https://github.com/auth0/limitd/blob/master/lib/db/redis/drip_and_take.lua

# 6. 参考资料

http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/

http://www.cnblogs.com/exceptioneye/p/4783904.html

https://blog.jamespan.me/2015/10/19/traffic-shaping-with-token-bucket/

http://jinnianshilongnian.iteye.com/blog/2305117

https://redis.io/commands/INCR#pattern-rate-limiter

https://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis/155

http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis-part-2/166

http://vinoyang.com/2015/08/23/redis-incr-implement-rate-limit/

https://blog.figma.com/an-alternative-approach-to-rate-limiting-f8a06cf7c94c