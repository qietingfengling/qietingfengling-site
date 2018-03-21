---
title: "Golang中rate包源码剖析"
date: 2018-02-27T20:24:57+08:00
subtitle: ""
tags: [go]
---
# 背景
最近公司需要对Nginx新版本进行压测，包括http/1.0,http/1.1和http/2.0不同协议版本间的性能对比，也包括各种秘要套件的性能对比。
目前支持http/2.0协议和各种秘要套件的开源压测软件较少，而且性能也比较难压上去。
于是想到用go自研性能压测软件，其中需要用到golang中的`golang.org/x/time/rate`,对其实现方式特别感兴趣，于是对其源码进行了剖析。

# 简介
限制器（Limiter）主要用于控制事件发生的频率。它实现了一个大小为b的"令牌桶"，最初已满并以每秒r个令牌来填充令牌桶。
也就是说，在任何足够长的时间间隔内，限制器（Limiter）将限制每秒只能获取r个令牌，最大突发大小为b个事件。
特殊情况下，如果r==Inf(无限速率｀math.MaxFloat64｀)，则b被忽略。


限制器有三个主要方法：`AllowN()`,`ReserveN()` 和`WaitN()`。三个方法的不同之处在于没有令牌的时候分别表现如下：

1、如果没有令牌可用，`WaitN()`将返回false。

2、如果没有令牌可用，`ReserveN()`将返回未来令牌的预留量以及调用者在使用之前必须等待的时间量

3、如果没有令牌可用，`AllowN()`则等待阻塞，直到可以获得或者其关联的context被取消。


关于令牌桶思想可以查看以下网址：

－https://en.wikipedia.org/wiki/Token_bucket

# 简单使用

## wait/waitN限制每秒发生一个事件
```go
package main

import (
	"fmt"
	"context"
	"time"
	"golang.org/x/time/rate"
)

func main() {
	r := 1
	b := 10
	l := rate.NewLimiter(rate.Limit(r), b)
	c, _ := context.WithCancel(context.TODO())
	for {
		l.Wait(c) //or l.WaitN(c, 1)
		fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
	}
}
```
`WaitN`阻塞直到lim允许n个事件发生。如果n超过限制器的突发大小，Context被取消或者预期的等待时间超过Context的deadline，则返回错误。

## allow/allowN限制每秒发生一个事件
```go
package main

import (
	"fmt"
	"time"
	"golang.org/x/time/rate"
)

func main() {
	r := 1
	b := 10
	l := rate.NewLimiter(rate.Limit(r), b)
	for {
		if l.Allow() { // or l.AllowN(time.Now(), 1) 
			fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
		}
	}
}
```
`AllowN`报告是否有n个事件可能发生。如果您打算删除或者跳过超过速率限制的事件，就使用这个方法。否则使用`Reserve`或`Wait`

## Reserve/ReserveN限制每秒发生一个事件
```go
package main

import (
	"fmt"
	"time"
	"golang.org/x/time/rate"
)

func main() {
	r := 1
	b := 10
	l := rate.NewLimiter(rate.Limit(r), b)
	for {
		rv := l.Reserve() // or ReserveN(time.Now(), 1)
		if !rv.OK() {
			return
		}
		time.Sleep(rv.Delay())
		fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
	}
}
```
`ReserveN`返回一个Reservation对象，表示在n个事件发生之前呼叫者必须等待多久。
限制器在允许将来的事件时考虑此预留。如果n超过限制器的突发大小，则`ReserveN`返回false。
如果您希望根据速率限制等待并放慢速度而不丢弃事件，则使用此方法。
如果您想用context的deadline或者取消延迟，请改为使用`wait`。要放弃或跳过超出速率限制的事件，请使用`allow`。

# 源码剖析
## 两个类型
```go
// Limit 定了事件的最大频率，表示为每秒事件的数据量，0表示无限制
type Limit float64

// Inf 是无限的速率限制；它允许所有事件（即使突发为0）
const Inf = Limit(math.MaxFloat64)

//Reservation中ok为false时，作为Delay的返回
const InfDuration = time.Duration(1<<63 - 1)
```
## 两个主要结构体
```go
type Limiter struct {
	limit Limit   //限制器，每秒可以往令牌桶中放入的令牌数量
	burst int     //令牌桶中令牌总数，既最大允许并发事件数
	mu     sync.Mutex  //限制器锁，属于线程安全
	tokens float64    // 每秒可以从令牌桶中取出令牌的数量
	last time.Time  //tokens更新时的最后时间
	lastEvent time.Time  //限速时间的最新时间（过去或者未来）
}
```

```go
//Reservation保存有关Limiter延迟后发生的事件信息。可以被取消，被取消时Limiter可以允许其他事件执行
type Reservation struct {
	ok        bool  //是否允许事件发生
	lim       *Limiter //限制器的相关信息
	tokens    int  // 每秒可以从令牌桶中取出令牌的数量
	timeToAct time.Time  // 事件允许访问的时间点
	limit Limit  //reservation时候限制的时间数量
}
```

## 三个基本函数

### `durationFromTokens()` 
单位转化函数，将tokens数量转化成需要花费的时间

### `tokensFromDuration()` 
单位转化函数，将时间间隔转化为可以消费的tokens数量

### `advance()` 
其核心思路是通过这一次运行时间now和上一时刻时间last的时间间隔，
通过单位转化函数tokensFromDuration计算出这个时间间隔里可以消耗多少个tokens，
再加上限制器中原有缺失的tokens数，即可计算出时间间隔里新令牌池还有多少令牌数
```go
	// 计算由于时间流逝而产生的新令牌数量。
	delta := lim.limit.tokensFromDuration(elapsed)
	tokens := lim.tokens + delta
```

## 三个主要方法
`reserveN)()` 、`AllowN()` 和`waitN()` 这三个主要方法的简单使用已经在文章开头介绍过了，这里就不在累赘了。
这里主要剖析其是如何实现的。`reserveN()`是三个方法的基础，其他包括`reserve()`、`Allow()` 、
`wait()`、`AllowN()` 和`waitN()`都调用`reserveN()`并用其所计算出的数据。下面先介绍`reserveN()`方法。
###`reserveN()`
通过令牌池中的令牌数减去每秒可以消费的令牌数，
再通过单位转化函数`durationFromTokens()`转化出还需要等待多久才能有令牌可以消费，最后通过`now.Add(waitDuration)`
计算出timeToAct，表示事件需要阻塞到这个时间点才可以继续执行。
```go
	now, last, tokens := lim.advance(now)

	// 计算请求产生的令牌的剩余数量。
	tokens -= float64(n)

	// 计算等待时间。
	var waitDuration time.Duration
	if tokens < 0 {
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

	// 计算结果，注意，这个ok很重要，allow和wait都是通过ok来进行判断是否允许运用或是否阻塞
	ok := n <= lim.burst && waitDuration <= maxFutureReserve

	r := Reservation{
		ok:    ok,
		lim:   lim,
		limit: lim.limit,
	}
	if ok {
		r.tokens = n
		r.timeToAct = now.Add(waitDuration)
	}
```

可以看到上面调用`reserveN()`的例子，例子中通过调用`Delay()`在决策需要sleep多久。
`Delay()`底层是调用`DelayFrom()`，`DelayFrom()`就是将上面计算所得的timeToAct和now做sub计算，计算出需要休眠多少秒。源码如下：

```go
func (r *Reservation) Delay() time.Duration {
	return r.DelayFrom(time.Now())
}

func (r *Reservation) DelayFrom(now time.Time) time.Duration {
	if !r.ok {
		return InfDuration
	}
	delay := r.timeToAct.Sub(now)
	if delay < 0 {
		return 0
	}
	return delay
}
```

###`AllowN()`

`AllowN()`完全使用`reserveN()`方法，不过其返回的是是否可以执行的标志，即`reserveN()`中计算产生的ok。
所以说`AllowN()`在事件超出频率的时候会丢弃或跳过。
```go
func (lim *Limiter) AllowN(now time.Time, n int) bool {
	return lim.reserveN(now, n, 0).ok
}
```

###`waitN()`

`waitN()`底层也是调用`reserveN()`方法，其特殊之处是使用select语法和DelayFrom方法，计算出需要阻塞的时间间隔

```go
	r := lim.reserveN(now, n, waitLimit)
	if !r.ok {
		return fmt.Errorf("rate: Wait(n=%d) would exceed context deadline", n)
	}
	t := time.NewTimer(r.DelayFrom(now))
	defer t.Stop()
	select {
	case <-t.C:
		return nil
	case <-ctx.Done():
		r.Cancel()
		return ctx.Err()
	}
```

# 其他用法
## 如何自定义时间限速
通过`golang.org/x/time/rate`自己提供的`rate.Every()`方法。以下代码实现1分钟的限速：
```go
package main

import (
	"fmt"
	"time"
	"context"
	"golang.org/x/time/rate"
)

func main() {
	b := 10
	l := rate.NewLimiter(rate.Every(60*time.Second), b)
	c, _ := context.WithCancel(context.TODO())
	for {
		l.WaitN(c,1) //or l.WaitN(c, 1)
		fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
	}
}
```
# 使用场景
使用rate做压测工具
https://github.com/qietingfengling/Q-wind







