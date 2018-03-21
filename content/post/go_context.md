---
title: "Golang中context包源码剖析"
date: 2018-03-16T08:21:04+08:00
subtitle: ""
tags: [go]
---
# 背景
工作中Golang项目大量使用Context包，但是对其底层实现和具体用法都一知半解，最近抽时间对其源码包进行研读，形成blog分享下。

# 简介

在Go项目中，大量使用goroutine，比如项目中使用context做全局goroutine的终止，
比如官方http包使用context传递请求和上下文数据，比如官方rate包用context包管理wait阻塞终止。
如何做到有效地管理所产生的goroutine的停止和数据传输呢？方法有很多种，例如使用channel的方式，这里不做细化。
本文主要讨论context包的使用和实现原理。

更多实例和解释请查看https://blog.golang.org/context

# 主要方法的简单使用
## `WithCancel()`
主动终止所有goroutine
```go
package main

import (
	"context"
	"time"
	"fmt"
)

func doWork(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("%v is off duty now.\n", name)
			return
		default:
			fmt.Printf("%v working...\n", name)
			time.Sleep(2 * time.Second)
		}

	}
}
func main() {
	//cpuNum := runtime.NumCPU()
	//runtime.GOMAXPROCS(cpuNum)
	boss := context.Background()
	worker, cancel := context.WithCancel(boss)
	go doWork(worker, "[ Work A ]")
	go doWork(worker, "[ Work B ]")
	go doWork(worker, "[ Work C ]")
	time.Sleep(4 * time.Second)
	cancel()
	time.Sleep(5 * time.Second)
}

```
## `WithTimeout()`
实现若干时间后终止所有goroutine
```go
package main

import (
	"context"
	"time"
	"fmt"
)

func doWork(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("%v is off duty now.\n", name)
			return
		default:
			fmt.Printf("%v working...\n", name)
			time.Sleep(2 * time.Second)
		}

	}
}
func main() {
	boss := context.Background()
	worker, _ := context.WithTimeout(boss, 4*time.Second)
	go doWork(worker, "[ Work A ]")
	go doWork(worker, "[ Work B ]")
	go doWork(worker, "[ Work C ]")
	<-worker.Done()
	time.Sleep(5 * time.Second)
}

```

`WithDeadline`的实现和`WithTimeout`大同小异，稍微改变下语法如下，可以实现在特定时间终止所有goroutine
```go
worker, _ := context.WithDeadline(boss, time.Now().Add(4*time.Second))
```

#源码剖析

## 接口`Context`
接口`Context`是context包最主要最基础的接口，context包下的类型`emptyCtx`，结构体`cancelCtx`和`valueCtx`都是继承了Context接口。
```go
type Context interface {
	// 返回一个截止时间，表示这个时间后所有任务应该被取消。截止时间没有设置的时候，返回ok == false。
	// 连续调用截止时间将返回相同的结果
	Deadline() (deadline time.Time, ok bool)

	// 返回一个channel，当当前节点的context应该被取消或终止的时候，该channel会被关闭，
	// 对应的gorountine也应该返回并结束。如果此context无法取消，则可能返回nil。成功调用完成返回相同的值。
	Done() <-chan struct{}

	// 如果Done尚未关闭，则Err返回nil。如果Done被关闭，Err会返回错误说明原因：如果上下文被取消，则返回Canceled，
	// 如果上下文的截止时间已过，则返回DeadlineExceeded。在Err返回错误后，对Err连续调用返回相同的错误。
	Err() error

	// 使用key-Value的形式，纪录上下文共享的数据值，属于协程安全。
	Value(key interface{}) interface{}
}
```
## 接口`canceler`
接口`canceler`是`context`的类型，可以直接被取消。被`cancelCtx`和`timerCtx`继承,
`cancelCtx`和`timerCtx`都实现了自己的`cancel()`方法
```go
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

## 类型`emptyCtx`
`emptyCtx`继承Context接口，对于接口的四个方法都返回nil，是对象`background`和`todo`的类型，通常作为根节点的类型。
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

## 结构体`cancelCtx`
`cancelCtx`同时继承接口`Context`和`canceler`，调用函数`WithCancel()`返回该类型，用于取消上下文
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex  
	// 标识是否已经被cancel，当外部触发cancel或者父节点被cancel的时候，done会被关闭          
	done     chan struct{}         
	// children维护着它的所有子节点，当该节点被触发cancel时，会调用该节点下所有子节点的cancel()来终止自该节点下的所有节点。
	children map[canceler]struct{} 
	err      error                
}
```
## 结构体`timerCtx`
`timerCtx`继承结构体`cancelCtx`，因为`cancelCtx`继承了接口`Context`和`canceler`，因此`timerCtx`也继承了这两
个接口，其实现了自己的`cancel()`和`Deadline()`方法。

使用函数`WithDeadline()`和`WithTimeout()`返回该类型，`timerCtx`保存了一个计时器，用于在一定时间间隔里调用`cancel`函数
```go
type timerCtx struct {
        //继承cancelCtx，使用其Done方法和Err。
	cancelCtx
	// 计时器，用于在一定时间间隔里调用`cancel`函数
	timer *time.Timer 
	//实现自己的Deadline()方法，返回deadline
	deadline time.Time
}
```

## 实现原理

### 函数`propagateCancel()`
函数`propagateCancel`是三个主要使用方法`WithCancel()`、`WithDeadline()`和`WithTimeout()`的主要调用函数。
其主要用于构成整个context的结构树。当parent是根节点时直接返回，不是的时候将child做为对象存入parent的children参数中。
具体实现如下：
```go
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // 根节点直接return
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent 已经被取消
			child.cancel(false, p.err)
		} else {
		        // 构成结构树
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

### 函数`WithCancel()`
`WithCancel()`调用方法`propagateCancel()`构成context结构树，返回`cancelCtx`实现的`cancel()`方法，
使用者可以在适当时候调用。具体实现如下：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
`cancelCtx`实现的`cancel()`，主要做两件事：1、cancel该节点下所有的子节点；2、将子节点从父节点中移除。
当然，整个cancel过程通过加锁使得协程安全的。其具体实现如下：
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	// cancel该节点下所有的子节点
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		//将子节点从父节点中移除。
		removeChild(c.Context, c)

	}
}
```
函数removeChild()实现将子节点从父节点中移除。
```go
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)
	}
	p.mu.Unlock()
}
```

### 函数`WithDeadline()`
`WithDeadline()`通过用户指定的存活时间点d，先通过`time.Until()`计算当前时间离时间存活点d的时间间隔dur
再通过`time.AfterFunc()`返回计时器，再计时器时间到时，调用`timerCtx`的`cancel()`方法，
`timerCtx`的`cancel()`方法和`cancelCtx`的`cancel()`方法大同小异，这里过多介绍。
其具体实现如下：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

### 函数`WithTimeout()`   
`WithTimeout()`调用了`WithDeadline()`,只是将用户传入的超时时间转化为截止时间点作为`WithDeadline()`的参数传入。
其具体实现如下：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

### 函数`WithValue()`
`WithValue()`为结构树以Key-Value的形式存储共享数据,可以通过`Value()`，获取共享数据。其具体实现如下：
```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) String() string {
	return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```





