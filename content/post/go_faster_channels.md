---
title: "Go中更快的Channels"
date: 2018-04-24T10:24:47+08:00
subtitle: "使用非阻塞数据结构的技术来改进阻塞的Channels"
tags: [go]
---
# 简介
Go语言中的Channels是在go开发中用来构建并发的常用方法。GO中的channel的api是以CSP（Communicating sequential processes）描述的方式
来支持编程。关于CSP，可以参考Hoare 1978论文，Donovan and Kernighan在2015也讲述过CSP与Go的关系，
详细描述请查看https://godoc.org/github.com/thomas11/csp。 
Go中的Channels具有固定的缓存区大小b，
这使得只有b个发送者可以在没有将值传递给相应的接收者的情况下就可以返回。下面是一段发送和接收操作的
伪代码。我们的重点是发送和接收，这里没有涉及到select或者close。close添加起来也很方便，select可以
通过接收者使用等待机制的形式来实现。虽然这添加起来并不困难，但是和WaitGroup的实现相比会降低速度。

发送者伪代码
```go
send(c: chan T, item: T)
  atomically do:
    if the buffer is full
      block
    append an item to the buffer
    if there were any receivers blocked
      wake the first one up
```

接收者伪代码
```go
receive(c: chan T) -> T
  atomically do:
      begin:
      if there are items in the buffer
        result = head of buffer
        advance the buffer head
        if there are any senders waiting
          wake the first sender up
        return result
      if the buffer is empty
        block
        goto begin
```

Go的channels目前要求goroutines在执行对channels的操作前需要申请一个锁。
这使得对这个锁的竞争成为可扩展性的瓶颈；尽管获取这个锁的速度非常快，但是这也意味
着每次只有一个线程可以对队列进行操作。该论文描述一种新型的channel算法实现，
允许不同发送者和接受着并行完成操作。

我们首先回顾最近有关非阻塞队列的文献。然后，我们继续介绍Go中快速的无边界channel的实现；
这个算法可能是个人的兴趣。最后，我们将扩展此设计来提供Go的channells的有限制含义。
我们还提供算法的性能指标报告。

# 无阻塞队列
最接近无边界channel概念的数据结构是FIFO队列。一个队列支持入队和出队操作，如果队列中没有元素，
则会出现出队失败的错误。有许多并发队列的算法在改进和一致性方面提供不同层面的保障，但我们将
重点介绍非阻塞队列，因为它的文献是最接近可伸缩的并发数据结构。

通俗地说，如果一个线程可以执行一个操作并且要求它在无限时间内阻塞任何其他线程，那么我们说数据结构是非阻塞的。
因此，线程在获取队列中数据前不需要先获取锁，这个队列可以是非阻塞的：一个线程可以获取一个锁，然后在任何时间段
内取消调度，从而阻止其他所有线程去争夺锁。非阻塞算法通常用于像Compare-And-Swap（CAS）这样的原子指令来避免
不同线程间的上下文切换。非阻塞操作可以由以下几种不同程度定义来展示：

* 无阻塞：如果只有一个线程执行操作，那么该操作将以有限的步骤完成；
* 无锁定：无论并行执行操作的线程数量如何，至少有一个线程将以有限数量的步骤完成操作；
* 无等待：任何执行操作的线程都将保证以有限的步骤完成

非阻塞算法不是万能的。线程在完成操作所需要的时间上限是一定的，并不意味着该算法在实践中会表现得更好。
虽然某些嵌入式系统或者实时系统需要这种免等待的数据结构的强大保障，但是通常有阻塞算法在吞吐量性能方面
比无锁或无等待表现得更好。尽管如此，非阻塞算法可以在高竞争系统设置中表现得更加闪耀。少量得CAS操作可能
比获取锁得开销个更少，而更细粒度的并发加上进度保证可以减少竞争。

# 使用Fetch-and-Add来减少竞争

原子操作Fetch-and-Add (F&A)指令是将值添加到整数中去，并返回这个整数的旧值或者新值。下面是Go操作的基本语法。
```go
//atomically
func AtomicFetchAdd(src *int, delta int) { 
    *src += delta
    return *src
}
```
虽然F&A指令的硬件支持不如CAS的通用，但是F&A在x86上实现。在x86操作系统的机器上，F&A要比CAS快得多，并且成功率也高。
这样具有双重得影响，允许代码明智地选择F&A并不仅仅依赖于CAS来得到更高效和容易推理。举个例子，使用F&A来获取数组中的索引，
然后使用常规的技术来写入该索引。这样，它对操作各种数据结构时减少竞争是很有帮助的。

## 无限数组中的非阻塞队列

为了说明这一点，我们在无限数组的基础上，用Go的伪代码实现两个不同的非阻塞队列。这两种设计都是使用了只能在头尾指针进行元素地操作来实现。
Queue1是基于CAS算法来实现；Queue2是基于Yang and Mellor-Crummey在2016的伪代码中呈现的非阻塞队列。这里大概说明下面两个实现的
CompareAndSwap(v,a,b)的含义，其中v为内存值，a为旧的预期值，b为要修改的新值。当且仅当预期值a和内存值v相同时，将内存值v修改为b，
否则什么都不做。
```go
type Queue1 struct {
    head, tail *T
    data [∞]T
}
func (q *Queue1) Enqueue(elt T) {
    for{
        newTail := atomic.LoadPointer(&q.tail) + 1
        if atomic.CompareAndSwapT(newTail, nil, elt) {
             atomic.CompareAndSwap(&q.tail, q.tail, newTail)
             break
        }
    }
}
func (q *Queue1) Dequeue() T {
    for{   
        curHead := atomic.LoadPointer(&q.head)
        curTail := atomic.LoadPointer(&q.tail)
        if curHead == curTail {
            return nil
        }
        if atomic.CompareAndSwapPointer(&q.head, curHead, curHead+1) {
            return *curHead
        }
    
    }
}
```

第二个队列假设类型T不仅仅可以取nil值，而且也有一个明确的哨兵，用于确保用户不能传值到Enqueue函数。这个哨兵用于将索引标记为不可用，
表示线程在调用Enqueue函数的时候发生冲突应该再重试一次。

```go
type Queue2 struct {
	head, tail uint
	data     [∞]T
}

func (q *Queue2) Enqueue(elt T) {
	for {
		myTail := atomic.AddUint(&q.tail) - 1
		if atomic.CompareAndSwapT(&q.data[myTail], nil, elt) {
			break
		}
	}
}

func (q *Queue2) Dequeue() T {
	for {
		myHead := atomic.AddUint(&q.head) - 1
		curTail := atomic.LoadUint(&q.tail)
		if !atomic.CompareAndSwapPointer(&q.data[myHead], nil, SENTINEL) {
			return atomic.LoadT(&q.data[myHead])
		}
		if myHead == curTail {
			return nil
		}
	}
}
```

Queue1和Queue2的核心算法思想基本相同。Enqueueing线程加载尾指针并尝试在尾指针之后的一个元素进行CAS操作；Dequeueing线程则反过来操作
来推进头指针。Queue1和Queue2的实际差异是Queue2中线程执行Enqueue和Dequeue时优先移动头部和尾部索引。这意味着两个并发入队的操作总是在
不同的队列位置元素上尝试CAS操作。因此，队列操作只需要关心自己的除队操作，将头部指针增加到与尾部指针相同的值。

Queue2方法的确定是Queue能够做到无锁的，而Queue2只是无阻塞的。对于每一对入队/出队线程，每次操作都可以不断增加相等的头指针索引和尾指针索引，
而在入队者导致活锁之前，出队者的CAS操作总是能够成功的。活锁是一个或多个线程永不阻塞（即它们不断地改变它们各自的状态），但是线程仍然无限期地
无法取得进展的场景。


## Channels的课程
上面的Queue2是Yang and Mellor-Crummey实现快速无等待队列算法的核心。这也是我们设计更具有扩展性的channel使用的基本思想。我们设置的三个
类似的问题也在他们算法的其余部分得到解决。

1. 用有限的内存模拟一个无限数组。这里作者实现了一个固定长度数组的链表（称为段或单元）；线程在需要更多空间时增加这个数组。

2. 从无阻塞到无等待。这是一个缓慢的过程，这个过程既包括了上面的Dequeue和Enqueue算法的不断迭代，也包括使用帮助机制来帮助争用线程来完成其
未完成的操作。这种帮助机制是一种标准，可使无阻塞或者无锁算法进化为无等待。

3. 内存回收。也许在一个非阻塞队列中进行内存回收是一个正常不过的设置，但这的确是一项非常艰巨的任务。

虽然本文中第3点的解决方案是最有趣和有效的，但是这里我们是依赖Go的gc来解决这个问题。对于第1点，我们将采用与本文基本相同的算法，但是
对内存分配进行额外地优化。对于第2点，我们将慢慢实现channel的阻塞链表。

# 低竞争的无限制Channel
我们先考虑实现无限制channel的情况。当这个channel被阻塞：
* 因为Go的channel提供了同步机制，所以它对阻塞必须有一定的容忍度；
* 它只是在需要的时候才被阻塞（即对于尚没有相应发送的接收），并且当它进行时最多2个线程被阻塞，发送/接收成对的组件。
我们从下面代码所定义的类型开始：
```go
type Elt *interface{} type index uint64

// 数组链表的大小
const segShift = 12
const segSize = 1 << segShift


// channel缓存区的存储大小是segsize的固定大小数组链表。
// ID是一个单调递增的标志符，对应改数组链表缓冲区第一个元素的索引去除以segSize
type segment struct {
    ID index // index of Data[0] / segSize 
    Next *segment
    Data [segSize]Elt
}

 // Load自动加上数组链表的索引i处的元素
func (s *segment) Load(i index) Elt {
    return Elt(atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&s.Data[i]))))
}

// queue是channel的全局状态。它包含channel的头部索引和尾部索引，以及用于避免过度分配的备用段链表
type queue struct{ H, T index }

// Thread-local state for interacting with an unbounded channel
// 线程局部状态用于与无限制channel来进行交互
type UnboundedChan struct {
    // 指向全局状态的指针
    q *queue
    // pointer into last guess at the true head and tail segments 
    head, tail *segment
}
```
我们使用queue结构来保持一个唯一的全局数据结构状态，这个queue数据结构保存着队列的头指针索引和尾指针索引。在无限制channel中使用指针指向数据
本身主要考虑以下两个原因：

1. 它减少了因为更新共享头指针或者尾指针而引起的任何可能的争用。
2. 如果单个线程已经更新了本地的头指针和尾指针，那么当所有线程不再持有引用时，gc将能够清理所有已使用的段。

我们注意到，这种设计有一个缺点，那就是如果持有这种句柄的线程是处于非活跃状态，可能会导致空间泄漏，因为它会引起一个段空间的长期死亡。

用户使用channel，首先先创建一个初始值，然后使用NewHandle克隆该值和其他派生属性。


```go
// New初始化一个新的队列并返回该队列的初始句柄。所有其他的操作通过调用NewHandle()来进行分配
func New() *UnboundedChan {
    segPtr := &segment{} // 这里允许为0值
    q := &queue{H: 0, T: 0}
    h := &UnboundedChan{q: q, head: segPtr, tail: segPtr} return h
}

// NewHandle 为给定channel创建一个新句柄
func (u *UnboundedChan) NewHandle() *UnboundedChan {
    return &UnboundedChan{q: u.q, head: u.head, tail: u.tail}
}
```

## 发送和接收
数据入队（或者出队）算法是以原子方式递增尾部（Tail）索引的，尝试对队列中的数据进行CAS操作，并在CAS失败时唤醒被阻塞的线程。我们先从Enqueue代码开始，
然后解释它调用的代码。

```go
func (u *UnboundedChan) Enqueue(e Elt) {
    u.adjust()
    myInd := index(atomic.AddUint64((*uint64)(&u.q.T), 1) - 1)
    cell, cellInd := myInd.SplitInd()
    seg := u.q.findCell(u.tail, cell)
    if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])),unsafe.Pointer(nil), unsafe.Pointer(e)) {
        return
    }
    wt := (*waiter)(atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd]))))
    wt.Send(e)
}

func (u *UnboundedChan) Dequeue() Elt {
    u.adjust()
    myInd := index(atomic.AddUint64((*uint64)(&u.q.H), 1) - 1)
    cell, cellInd := myInd.SplitInd()
    seg := u.q.findCell(u.head, cell)
    elt := seg.Load(cellInd)
    wt := makeWaiter() 
    if elt == nil && atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])),unsafe.Pointer(nil), unsafe.Pointer(wt)) {
        return wt.Recv() 
    }
    return seg.Load(cellInd) 
}
    
```
以上代码第2行adjust()方法是原子加载头部（Head）索引和尾部（Tail）索引，然后添加索引使u.head和u.tail指向它们的单元格。第3行的atomic.Add
获取队列中的索引。第4行SplitInd()方法返回单元格的ID和与MyInd相对应的单元格所在的索引。由于尾部（Tail）只能增加，所以只有进行
Dequeueing的线程，在获取头部（Head）索引赋值给myInd时，才有可能对这个单元格进行争夺。这体现在第6行和21-22行的cas操作，如果入队的cas失败
了，这意味着Dequeue线程将转化为waiter，如果它成功了，那么它意味着Enqueuer可以返回，同时处以竞争的Dequeuer可以加载到值给cellInd赋值。

## 阻塞
那么waiter是什么？它就像缓冲区大小只有1的通道，或者Haskell中的MVar，它只允许在其上发送一个元素。我们目前基于单个值加上WaitGroup在实现的。
Go中sync包的WaitGroups允许goroutine向WaitGroup的计数器添加一个整数值并等待该计数器达到零（即可以用它来控制goroutine的安全退出）。如果
计数器小于零，那么但前的WaitGroup会出现异常，这样可以确保只有一个Send或者Recv在waiter上，所以这对调试很有帮助。

```go

type waiter struct { 
    E Elt
    Wgroup sync.WaitGroup
}

func makeWaiter() *waiter {
    wait := &waiter{}
    wait.Wgroup.Add(1)
    return wait
}

func (w *waiter) Send(e Elt) { 
    atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&w.E)), unsafe.Pointer(e)) 
    w.Wgroup.Done() // The Done method just calls Add(-1)
}

func (w *waiter) Recv() Elt { 
    w.Wgroup.Wait()
    return Elt(atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&w.E)))) 
}

```
我们实现阻塞策略有两个重要部分组成。无论是Enqueuer还是Dequeuer，如果Enqueuer在Dequeuer开始之前完成，都不会阻塞。事实上，他们必须执行的
唯一全局同步是单个F&A和单个非竞争的CAS操作（除非是队列的扩充）。其次，如果Enqueuer没有运行到被waiter阻塞，则waiter基本不会有争用，因为
只能有一个其他线程与其交互。

## 队列的增长和分配
现在我们来描述findCell方法的实现。该算法从传参进来的段指针（start）开始，一直寻找段的next指针直到段的ID等于传参进来给定的cellID。
如果findCell找到段的末尾还是无法找到符合条件的ID，它会尝试分配一个新的段并将其放到原来段的末尾。
下面是实现的代码：
```go

func (q *queue) findCell(start *segment, cellID index) *segment {
    cur:= start
    for cur.ID != cellID {
        next := (*segment)(atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&cur.Next)))) 
        if next==nil {
            q.Grow(cur)
            continue
        }
        cur = next
    }
    return cur
}

func (q *queue) Grow(tail *segment) {
    curTail := atomic.LoadUint64((*uint64)(&tail.ID))
    newSegment := &segment{ID: index(curTail + 1)}
    if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&tail.Next)),unsafe.Pointer(nil), unsafe.Pointer(newSegment)) {
        return
    } 
}

```

这里值得注意，我们可以在Grow函数中只执行一次CAS操作后就离开，因为如果我们的CAS操作失败了，这意味着其他线程会成功并且只有唯一一个id为tail.ID+1
的新段会放置在tail.ID后面。但是，这种实现存在一个问题：这是非常浪费的。在高争用的情况下，很多线程都有可能分配一个新的段，但是只有一个
线程会成功。其他失败的线程所分配的空间将无法访问，并将被GC垃圾回收。在我们的实验中，当channel大小大于1024时，channel操作的速度是最快的，因此
任何无效的空间分配对吞吐量都会产生实际的影响。这种影响在我们的性能测试中很明显。

我们的解决方案是在这个队列结构中保存一个无锁链表。线程在执行Grow的时候首先尝试从列表中pop出一个段，然后再CAS操作。只有当这个pop操作失败了，
他们才会分配一个新的段。对应的，如果CAS失败，则线程会尝试将一个段推送到列表上。该列表使用一个最佳计数器来保存列表的长度，
并且不允许这个计数器的增长超过最大长度；这使得我们可以避免在实现这个队列的时候出现内存泄露。Grow的全部代码，请参阅附录A。

# 扩展到有限制channel的情况

Go的channels没有无限制的变种。尽管本文上面提到的数据结构可能是有用的，但是在某些情况下，选择有限制的channel会是更好的选择。无缓冲区通道
允许更多的同步编程模型，它在Go中同步两个协作线程是很常见的；这个级别的同步是有用的。本节描述基于无限制channel的基础实现的有限制的channel。

##准备知识
我们复用了q和segment类型，以及findCell和Grow机制。几乎所有的区别都在Enqueue和Dequeue方法中实现。但是要复杂很多。这种复杂性是因为senders
和receivers被赋予新的功能：

* Senders必须决定是否自己被阻塞然后等待更多的receivers到达；
* Receivers必须唤醒所有应该醒来的waiters，如果他们成功地从队列中pop出元素。

和以前一样，此协议的实现方式是避免阻塞的，除非channel语法需要阻塞。这意味着Enqueue和Dequeue方法必须考虑无限制channel协议和新的阻塞协议
地任意交错。有限制的channel（BoundedChan）绑定一个额外的整数字段，该字段指示在没有没有对应的receiver下，最大数量的senders可以允许被返回。

我们还引入了一个不可变的全局哨兵指针，用于接收线程以发信号通知sender不应该被阻塞。这种设计导致的结果是，所有原来需要从nil指针到其他指针左CAS
操作都变成需要从哨兵指针到其他指针做CAS操作。我们约定不会出现哨兵变为nil的情况，所以下面的tryCas方法能够保证返回值seg.Data[segInd]既不会是nil
也不会是哨兵
##（另请参阅）段中元素的可能历史纪录
在无限制的情况下，队列中的元素基本上有两个可能的历史值：

| Events               | History         |
| -------------------- | --------------- |
| Sender, Receiver     | nil → Elt       |
| Receiver, Sender     | nil → *waiter   |

上面这些可以被视为实现无限制channel是强制执行的固定变量。有限制的channel的情况下，有更多可能的历史值。下面这些（并且只有这些）都可能出现，
牢记下面这些规则有助于我们理解这种协议：

| Events                           | History                     |
| -------------------------------- | --------------------------- |
| Sender, Receiver                 | nil → Elt                   |
| Receiver, Sender                 | nil → *waiter               |
| Waker, Sender, Receiver          | nil → sentinel → Elt        |
| Waker, Receiver, Sender          | nil → sentinel → *waiter    |
| Sender†, Waker, Sender, Receiver | nil → *weakWaiter → Elt     |
| Sender†, Waker, Receiver, Sender | nil → *weakWaiter → *waiter |

其中，Sender表示sender到达，但必须阻止更多receivers完成，而Waker是成功唤醒阻塞的Sender的任何线程。 
下面的章节将详细介绍一下weakWaiter是什么以及究竟谁扮演的是“Waker”角色。
## 入队
我们首先介绍tryCas和Enqueue的源代码：
```go
func tryCas(seg *segment, segInd index, elt unsafe.Pointer) bool {
    return atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[segInd])),
        unsafe.Pointer(nil), elt) ||
        atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[segInd])),sentinel, elt)
}

//Enqueue把e发送到b。如果已经有>=bound的goroutines阻塞了，
//则enqueue将阻塞，直到有足够多的元素被received。
func (b *BoundedChan) Enqueue(e Elt) {
    b.adjust()
    startHead := index(atomic.LoadUint64((*uint64)(&b.q.H)))
    myInd := index(atomic.AddUint64((*uint64)(&b.q.T), 1) - 1)
    cell, cellInd := myInd.SplitInd()
    seg := b.q.findCell(b.tail, cell)
    if myInd > startHead && (myInd-startHead) > index(uint64(b.bound)) {
        // 这是一个我们可以阻塞的机会
        var w interface{} = makeWeakWaiter(2)
        if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])),unsafe.Pointer(nil), unsafe.Pointer(Elt(&w))) {
            // 在w里我们成功交换了。没有人可以覆盖这个位置，除非他们已经发送了w，我们阻塞
            w.(*weakWaiter).Wait()
            if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])),unsafe.Pointer(Elt(&w)), unsafe.Pointer(e)) {
                return
            } // someone put a waiter into this location. We need to use the slow path
        } else if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])),sentinel, unsafe.Pointer(e)) {
            //从startHead到这里的代码，有足够的增量让我们不应该再阻塞
            return
        }
    } else {
        // 正常情况。 我们知道我们不必阻止，因为b.q.H只能增加。
        if tryCas(seg, cellInd, unsafe.Pointer(e)) {
            return
        }
    }
    ptr:= atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[cellInd])))
    w := (*waiter)(ptr) 
    w.Send(e)
    return
}
```
Enqueue首先加载b.q.H（队列的head）的值，然后获取myInd（b.q.T+1）。 请注意，队列状态不是一直不变的，因为H可能在加载它和增加
myInd（第12-13行）时出现移动。 但是，H（Head）只会增加！ 如果myInd-startHead超出b.bound的范围，则意味着随着T（Tail）的增长，
H（Head）已经远远落后于T。 在这种情况下，我们可以简单地尝试在e中进行CAS操作（第40行）。 如果失败了，它只能意味着receiver已经在该索
引中放置了waiter，所以我们唤醒了接收方并返回（第44-46行）。

如果我们有机会进行阻塞，那么我们分配一个新的weakWaiter。weakWaiter就像一个waiter，除了它不包含值，但它确实允许接收多个消息。 
在Go中有很多方法来实现这样的构造，下面是一个WaitGroup的实现：

```go
type weakWaiter struct { 
    OSize, Size int32 
    Wgroup sync.WaitGroup
}

func makeWeakWaiter(i int32) *weakWaiter {
    wait := &weakWaiter{Size: i, OSize: i} 
    wait.Wgroup.Add(1)
    return wait
}

func (w *weakWaiter) Signal() {
    newVal := atomic.AddInt32(&w.Size, -1) 
    orig := atomic.LoadInt32(&w.OSize)
    if newVal+1 == orig {
        w.Wgroup.Done() 
    }
}
```

在这种情况下，我们可能会阻塞，我们构造一个缓冲区大小为2的weakWaiter，因为可能有两个出队线程同时尝试唤醒队列线程（参见下文）。 
如果发送方成功地完成将w加入到适当的位置的CAS操作（第19行），那么它会在唤醒时等待并尝试其余的无限制channel协议。 
如果CAS失败，有两种可能的情况：

1. 在存储w之前，channel中已经存储有b.bound个元素，这时receiver尝试唤醒sender。
2. receiver已经开始在这个位置上等待了

第29行的CAS操作决定了这是哪种情况。如果时（1）这种情况，那么CAS操作将失败并且sender必须唤醒等待的receiver线程（第46行）。如果是（2）这种
情况，则CAS操作将成功并且元素e将成功入队。

## 出队
Dequeue的实现有效地反映了Enqueue的实现。但是，有一些实现特别巧妙，让我们来看看具体实现：
```go
func (b *BoundedChan) Dequeue() Elt {
    b.adjust()
    myInd := index(atomic.AddUint64((*uint64)(&b.q.H), 1) - 1)
    cell, segInd := myInd.SplitInd()
    seg := b.q.findCell(b.head, cell)
    // 如果Enqueuers由于缓冲区大小而等待去完成，我们将负责唤醒d.b.H+b.bound所在索引的线程。 如果bound为零，那只是唤醒当前线程。 
    // 否则，我们必须做一些额外的工作。 我们唤醒的线程所在索引赋值给buddy。
    var (
        bCell, bInd index
        bSeg *segment
    )
    if b.bound>0{
        buddy := myInd + index(b.bound)
        bCell, bInd = buddy.SplitInd()
        bSeg = b.q.findCell(b.head, bCell)
    }
    w := makeWaiter()
    var res Elt
    if tryCas(seg, segInd, unsafe.Pointer(w)) {
        res = w.Recv()
    }else{
        // tryCas失败，这意味着根据历史表的可能性，这必须是Elt，waiter或者weakWaiter。
        // 因为我们是唯一被允许将数据放到这个位置的线程，所以它不可能是waiter。 因此，它可能是weakWaiter或Elt。 
        // 如果它是一个weakWaiter的话，那么我们必须在w存储之前发送它，否则另一个线程可能会饿死。 
        // 如果这是一个正常的Elt，我们会执行协议的其余部分。 这也意味着我们可以安全地从段（seg）中载入一个Elt，
        // 这并不总是这样，因为哨兵（sentinel）不是Elt。 
        // 第一步：我们没有把我们的waiter赋值给Ind，这意味着要不我们有一个值在那里，或者有一个weakWaiter在那里。 
        // 无论哪种方式，这些都是有效的elt，我们可以有效地用类型断言给他们区分开来
        elt := seg.Load(segInd)
        res = elt
        if ww, ok := (*elt).(*weakWaiter); ok {
            ww.Signal()
            if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&seg.Data[segInd])
                unsafe.Pointer(elt), unsafe.Pointer(w)) {
                res = w.Recv()
            } else {
                // someone cas'ed a value from a weakWaiter, could only have been our 
                // friend on the dequeue side
                res = seg.Load(segInd)
            }
        }
    }
    for b.bound > 0{//最多运行两次
        // 我们已经成功地从我们的单元格中获得了值。 现在我们必须确保我们的buddy在他们等待的时候被唤醒，或者他们不会睡眠。 
        // 如果bElt不是nil，则它有一个Elt或一个weakWater。 如果它有一个weakWaiter，那么我们需要发送它唤醒buddy。 
        // 如果不是，那么我们试图将buddy索引放入到哨兵中。 如果我们失败了，那么这个buddy可能会channel中等待CAS操作，
        // 所以我们必须重新开始。 然而，这只会发生一次。
        bElt := bSeg.Load(bInd)
        if bElt != nil {
            if ww, ok := (*bElt).(*weakWaiter); ok { 
                ww.Signal()
            }
            // 在bSeg.Data[bInd],中有一个真正的队列值，因此，buddy不会被阻塞
            break
        }
        // 让buddy知道他们不必阻塞
        if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&bSeg.Data[bInd])), unsafe.Pointer(nil), sentinel) {
            break
        } 
    }
    return res 
}
```
现在聊一聊精妙之处。 dequeuer可能不得不唤醒多个等待发送线程：一个在myInd处等待，另一个在myInd + bound（或bInd）处等待。 
这可能看起来很奇怪，因为dequeuer接收myInd-bound时应该唤醒任何待处理的sender。 问题是我们没有保证这个dequeuer已经返回。 
发生这种情况的可能性大小取决于缓冲区的大小，但当bound很小时，发生会越来越频繁。

第二个是Go的一个特点。 在第87行中有一个类型转换，它将Elt转换为一个类型为interface {}的值。 interface{}包含一个指向某个
运行时的有关结构体指针的实际类型的信息。如果elt是指向weakWaiter的指针，（* weakWaiter）会执行语法查询。 这是一件安全的事情，
因为weakWaiter是一个包私有类型：外部调用者是不能传入指向weakWaiter的Elt除非我们从包中提供公共方法作为返回，但是我们是不会这样做。

这很复杂，因为*waiters实际上是直接存储在队列中的，而没有隐藏在接口值的后面（例如，在第90行）。 这是因为额外的间接层是不必要的：
因为总是可以确定一个给定的Elt或*waiter在CAS的失败和成功的位置。

# 性能

我们对入队/出队队列中的5个独立频道进行了基准测试：
* Bounded0：缓冲区大小为0的BoundedChan
* Bounded1K：缓冲区大小为1024的BoundedChan
* Unbounded：无界的chan
* Chan0：一个无缓冲的原生Go的channel
* Chan1K：缓冲区大小为1024的原生Go的channel
* Chan10M：缓冲区大小为$ 10 ^ 7 $的原生Go的channel，这是在基准测试过程中排入channel的元素总数。

我们包含两种情况的基准测试结果：一种是我们为每个处理器分配一个goroutine（其中处理器使用Go运行时的GOMAXPROCS过程设置），
另一个分配5000个goroutine，而不考虑GOMAXPROCS的当前值。 我们包括这两个原因有两个。 首先，在运行的Go程序中有数千个goroutine处于活
动状态并不罕见，因此这必须考虑处理器以这种方式过度订阅的情况。 其次，我们注意到，在核心超额订阅的情况下，性能往往更好。 虽然与直觉相反，
但这可能是由于不可预知的调度程序性能以及在同一个内核上执行的两个goroutine之间的同步开销较低所致。

未完待续。。。。。。















