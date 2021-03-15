+++
title = "如何实现重试"
date = 2021-03-15T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

**为什么要重试？**

重试是在工程实践中用来提高系统容错能力的一种手段，在微服务的部署架构中，一个请求链路往往需要经过多个服务进行处理返回，而每个服务之间的信息交换依赖于网络通信，网络大部分情况下是稳定的，请求通常能够正常达到和返回到每个服务，但实际情况下可能因为各种原因网络状态会出现波动，使请求可能无法正常达到服务端，或是服务端没有及时返回给客户端使请求链路发生超时，最终导致请求失败，当请求链路中的服务数量越来越多时，意味着请求更容易受到这种影响而失败，而这种情况下导致的失败往往可以通过重试的方式解决，因此实现服务间的自动重试某种意义上提高了整个系统故障恢复能力。

典型的重试场景如下：

1. 网络抖动问题造成的请求失败，通过重试提高成功率。
2. 由于系统负载高原因导致请求变慢，导致请求超时，通过重试提高成功率。
3. 由于系统故障或服务不可用导致请求没能成功，通过重试保证数据落地。

首先，是不是所有请求都可以重试呢？显然不是，通常重试依赖于服务端接口的幂等性，假设对于一个接口，多次进行调用会违反了数据约束，那么自动重试则可能会使结果在我们的预期之外。那么，对于幂等的接口，是不是所有请求都可以立即发起重试呢？对于第一种单纯因为网络抖动引起的失败我们通常能简单地发起重试，但是对于其他两种场景，似乎不太适合直接发起重试了，对于系统负载高导致的超时，此时进行重试往往会进一步提高系统的负载，进而导致后续请求发生雪崩，降低了系统的可用性，此时重试便失去了其意义；对于系统当前发生故障的情况下，如果系统没能及时恢复对外提供服务能力，那么失败后的立即重试就变成了“无用功”，如果当前的请求需要保证成功时，此时重复的重试就变成了忙等待，占用着服务端资源。由此可见，不是所有请求都适合立即处理重试。

因此，对于是否可以重试上可以划分出：「可重试」与「不可重试」，在是否可以立即重试上可以划分出：「同步重试」和「异步重试」，最后对重试的等待间隔还需要制定一个「重试策略」。

**「可重试」与「不可重试」**

对于是否可以重试，常用的方式是通过错误码来区分哪些调用可以重试，哪些不可以重试，例如在配置重试规则时，可以指定需要重试的错误码，重试接口依据错误码来决定是否进行重试：

 - allowlist：指定可以重试的错误码，不在列表中的错误码将不进行重试
 - denylist：指定不需要进行重试的错误码，不在列表中的错误码将都进行重试

或者依据重试命令（RPC Command）来决定是否进行重试：

 - allowlist：指定可以重试的命令，不在列表中的命令将不进行重试
 - denylist：指定不需要进行重试的命令，不在列表中的命令将都进行重试

当上游服务过载时，处理速度往往会变慢，此时重试会加大服务的压力，因此需要采集历史的耗时和成功率数据，定义发送重试的阈值，避免对被调用服务的冲击，根据组件采集到的历史数据（平均耗时和成功率）进行计算，配置进行重试的数据阈值：

 - 耗时：当前调用命令耗时大于 n（n为最近平均调用耗时的百分位：0~100），不再进行重试
 - 成功率：当前调用命令成功率小于 n（n为最近平均调用的成功率：0~100），不再进行重试

**重试策略**

对于重试间隔等待的策略，可以简单地划分为以下几种：

 - 简单重试：指定重试次数，失败后立即进行重试，直到重试次数超过指定重试次数
 - 线性退避：指定重试的间隔，失败后等待固定间隔，进行重试操作
 - 随机退避：指定重试的间隔，失败后等待区间的随机间隔，进行重试操作
 - 指数退避：指定重试的间隔，和上限等待时间，失败后等待 2^x * n（n为重试间隔，x为重试次数），进行重试操作，当超过上限等待时间后，不再进行重试
 - 自定义间隔：指定重试的间隔窗口，失败后等待 period(x)（x为重试次数），进行重试操作，当重试超过窗口大小后，不再进行重试

![retry-pocliy](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/retry_policy.png)

**「同步重试」与「异步重试」**

对于最普遍的重试方式，也就是同步重试，很好理解，其过程如下所示：

![sync-retry](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sync_retry.png)

1. 当前服务向外部服务发起一个 RPC 请求
2. 如果成功直接返回，否则进入重试步骤
3. 获取重试策略，判断是否应该进行重试
4. 如果应该重试且重试次数没达到最大重试次数，等待一个间隔时间继续进行重试

在这种方式下，重试流程会贯穿在请求的整个生命周期中，因此重试过程会随着请求终止而随之终止。那么现在假如整个请求链路中的每一层都实现了这种重试机制，会发生什么呢？

假设当前有 S1->S2->S3 三个服务，最大重试次数为3次，当 S2 调用 S3 失败时，会重试调用 S3 3次，如果都失败的话，那么 S2 会给 S1 返回失败，这时候 S1 又会重试调用 S2，假如后续请求都失败的话，那么这个链路中会发起3x3=9次请求，足足放大了9倍，假如服务数量或者最大重试次数更多之后，这种现象就会更加明显，甚至影响了整个系统。对于重试风暴，需要以一种合适的方式及时中断重试，防止重试次数以线性方式增长，比如用指数退避的方式进行重试，同时需要为整个链路的重试次数设置一个硬上限，避免重试次数过多，另外当上层服务已经发生超时返回错误的话，底层服务因为无法感知到请求已经失败而继续进行重试，如果这时候重试成功则会出现不一致的情况，因此底层服务应该及时中断重试，避免更多无意义调用。

而对于那些不需要立即返回状态的调用，或者短时间内不能立即成功的重试，将重试持久化起来，异步进行重试是一个不错的选择，在这种方案下，可以引入一个独立的重试服务，接受重试消息将其重试持久化下来，并且及时通知使用重试的使用方进行重试，如下所示：

![async-retry](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/async_retry.png)

1. S1 对 S2 发起一个 RPC 请求
2. 如果成功直接返回，否则进入重试步骤
3. S1 调用重试 API 向重试服务发起一个重试请求
4. 重试服务向延迟队列中插入一个带有调度时间的重试请求
5. 重试服务获取到期的重试请求，将回调写入到 S1 的消息队列中
6. S1 消费重试请求并发起重试，成功后将向重试服务发起一个响应重试的请求
7. 重试服务收到请求后将重试从调度表中删除

**实现一个异步重试服务**

有了以上基础之后，接下来就可以根据这些理论来编写一个简单的异步重试服务，目标如下：

 - 基于多个失败重试的场景，抽象出通用的消息重试服务以及组件
 - 提供统一重试 API，易于开发和接入
 - 提供多种重试策略，使用方按需选择
 - 保证组件的性能和可靠性

主要实现如下：

1.接收重试请求：

接收一个重试请求后，首先根据请求的重试策略获得重试的下一次调度时间以及是否应该重试，如果不需要重试则直接返回，否则将重试持久化，并写入到延迟队列，到期设置为下一次调度时间。

```go
func Retry(req *retrypb.RetryRequest) error {
	if err := validateRetry(req); err != nil {
		return err
	}
	retryTime, shouldRetry := getReqNextRetryTime(req)
	if !shouldRetry {
		return errors.New("should not retry")
	}
    queueName := formatQueueName(req)
	startQueueWorkerChan <- queueName

	if err := store.StoreRetry(req.GetRetryId(), req); err != nil {
		return err
	}
	if err := store.ScheduleRetry(queueName, req.GetRetryId(), retryTime); err != nil {
		return err
	}
	return nil
}
```

2.处理重试：

处理重试时，可以批量从延迟队列中获取到期的重试请求，取的一个重试请求后，判断重试次数是否超过最大上限，如果超过则从调度表中删除该重试并返回，否则将重试次数加1，更新到数据库中，接下来把重试通知写入到消息队列，回调给使用方，若发送失败，则直接跳过该请求，下一次处理时继续进行重试，发送成功后，继续获取重试下一个周期的调度时间，更新到延迟队列。

```go
func handleExpiredRetries(queueName string) error {
	retryIDs, err := store.GetExpiredRetries(queueName, fetchRetriesSize)
	if err != nil {
		return err
	}
	for _, retryID := range retryIDs {
		req, err := store.GetRetry(retryID)
		if err != nil {
			continue
		}
		msg := req.GetMsg()
		retryCnt := msg.GetRetryCnt()
		if retryCnt > maxRetryCnt {
			store.RemoveRetrySchedule(queueName, retryID)
			continue
		}
		msg.RetryCnt = retryCnt + 1

		store.StoreRetry(retryID, req)
		if err := mq.SendMessage(req.Marshal()); err != nil {
			continue
		}
		retryTime, shouldRetry := getReqNextRetryTime(req)
		if !shouldRetry {
			store.RemoveRetrySchedule(queueName, retryID)
			continue
		}
		store.ScheduleRetry(queueName, retryID, retryTime)
	}
	return nil
}
```

3.删除重试：

当接收到使用方的重试回应时，应删除延迟队列中的重试请求。

```go
func Remove(req *retrypb.RetryRequest) error {
	if err := validateRetry(req); err != nil {
		return err
	}
	queueName := formatQueueName(req)
	return store.RemoveRetrySchedule(queueName, req.GetRetryId())
}
```

4.重试策略

简单重试：重试间隔应该永远返回0（代表立即重试）

```go
func (p *simplePeriodPolicy) NextPeriod() (time.Duration, bool) {
	if p.retryCnt >= p.policy.GetMaxRetryCnt() {
		return 0, false
	}
	return 0, true
}
```

线性退避：

```go
func (p *linearPeriodPolicy) NextPeriod() (time.Duration, bool) {
	return time.Duration(p.policy.GetPeriod()) * time.Millisecond, true
}
```

随机退避：

```go
func (p *randomPeriodPolicy) NextPeriod() (time.Duration, bool) {
	if p.policy.GetMinPeriod() >= p.policy.GetMaxPeriod() {
		return 0, false
	}
	return time.Duration(rand.Intn(int(p.policy.GetMaxPeriod()-p.policy.GetMinPeriod()))+int(p.policy.GetMinPeriod())) * time.Millisecond, true
}
```

指数退避：

```go
func (p *exponentialBackOffPeriodPolicy) NextPeriod() (time.Duration, bool) {
	attempts := p.retryCnt
	exp := math.Pow(2, float64(attempts))
	result := p.policy.GetWaitExponentialMultiplier() * uint32(exp)
	if result > p.policy.GetWaitExponentialMax() {
		return 0, false
	}
	return time.Duration(result) * time.Millisecond, true
}
```

自定义间隔：

```go
func (p *customPolicy) NextPeriod() (time.Duration, bool) {
	attempts := p.retryCnt
	retryPeriods := p.policy.GetPeriods()
	if attempts >= uint32(len(retryPeriods)) {
		return 0, false
	}
	retryPeriod := retryPeriods[attempts]
	return time.Duration(retryPeriod) * time.Millisecond, true
}
```

**总结**

要实现正确的重试并不是易事，落实到操作上还有许多细节要考虑，本文主要介绍了重试这一提高容错能力的手段，以及利用其中一种重试方式去编写一个可用的服务，仅供参考和拓展思路。
