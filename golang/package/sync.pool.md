---
title: "sync.Pool 的实现原理和适用场景"
date: 2018-11-13T22:52:14+08:00
categories:
    - Golang
tags: 
    - Golang
    - sync
---

# sync.Pool 的实现原理和适用场景

Golang GC 大大减少了程序编程负担，但 GC 是一把双刃剑，带来了编程的方便但同时也增加了运行时开销，使用不当甚至会严重影响程序的性能。因此性能要求高的场景不能任意产生太多的垃圾，如何解决呢？

那就是 ** 重用对象 **

我们可以简单的使用一个 chan 把这些可重用的对象缓存起来，但如果很多 goroutine 竞争一个 chan 性能肯定是问题，为了避免大家重造车轮，因此官方统一出了一个包 [`sync.Pool`](https://golang.org/src/sync/pool.go)

首先来看下官方给的注释，已经解释的非常到位:

```

// A Pool is a set of temporary objects that may be individually saved and
// retrieved.
//
// Any item stored in the Pool may be removed automatically at any time without
// notification. If the Pool holds the only reference when this happens, the
// item might be deallocated.
//
// A Pool is safe for use by multiple goroutines simultaneously.
//
// Pool's purpose is to cache allocated but unused items for later reuse,
// relieving pressure on the garbage collector. That is, it makes it easy to
// build efficient, thread-safe free lists. However, it is not suitable for all
// free lists.
//
// An appropriate use of a Pool is to manage a group of temporary items
// silently shared among and potentially reused by concurrent independent
// clients of a package. Pool provides a way to amortize allocation overhead
// across many clients.
//
// An example of good use of a Pool is in the fmt package, which maintains a
// dynamically-sized store of temporary output buffers. The store scales under
// load (when many goroutines are actively printing) and shrinks when
// quiescent.
//
// On the other hand, a free list maintained as part of a short-lived object is
// not a suitable use for a Pool, since the overhead does not amortize well in
// that scenario. It is more efficient to have such objects implement their own
// free list.
//
// A Pool must not be copied after first use.
```

sync.Pool 最常用的两个函数 Get/Put，再来给个 demo:

```
package main

import(
    "fmt"
    "sync"
)

func main() {
    p := &sync.Pool{
        New: func() interface{} {
            return 0
        },
    }

    a := p.Get().(int)
    p.Put(1)
    b := p.Get().(int)

    fmt.Println(a, b)
}
```

对象池在 Get 的时候没有里面没有对象会返回 nil，所以我们需要 New function 来确保当获取对象对象池为空时，重新生成一个对象返回，pool.Get 的功能是从池中获取一个 interface{} 类型的值，pool.Put 的作用则是把一个 interface{} 类型的值放置于池中

如果一个临时对象池的 Put 方法未被调用过，且它的 New 字段也未曾被赋予一个非 nil 的函数值，那么它的 Get 方法返回的结果值就一定会是 nil。Get 方法返回的不一定就是存在于池中的值。不过，如果这个结果值是池中的，那么在该方法返回它之前就一定会把它从池中删除掉

对象池使用是较简单的，但原生的 sync.Pool 不是万精油，存在一些问题：

- 我们不能自由控制 Pool 中元素的数量
- 放进 Pool 中的对象每次 GC 发生时都会被清理掉

这些小问题使得 sync.Pool 做简单的对象池还可以，但做连接池就有点心有余而力不足了，比如：在高并发的情景下一旦 Pool 中的连接被 GC 清理掉，那每次连接 DB 都需要重新三次握手建立连接，这个代价就较大了。

## 缓存对象的开销和过程
如何在多个 goroutine 之间使用同一个 pool 做到高效呢？官方的做法就是尽量减少竞争。

因为 sync.pool 为每个 P（对应 cpu，不了解的童鞋可以去看看 golang 的调度模型介绍）都分配了一个子池。

当执行一个 pool.get 或者 put 操作的时候都会先把当前的 goroutine 固定到某个 P 的子池上面，然后再对该子池进行操作。每个子池里面有一个私有对象和共享列表对象，私有对象是只有对应的 P 能够访问，因为一个 P 同一时间只能执行一个 goroutine，因此对私有对象存取操作是不需要加锁的。共享列表是和其他 P 分享的，因此操作共享列表是需要加锁的。

获取对象过程是：
1. 固定到某个 P，尝试从私有对象获取，如果私有对象非空则返回该对象，并把私有对象置空；
2. 如果私有对象是空的时候，就去当前子池的共享列表获取（需要加锁）
3. 如果当前子池的共享列表也是空的，那么就尝试去其他 P 的子池的共享列表偷取一个（需要加锁）
4. 如果其他子池都是空的，最后就用用户指定的 New 函数产生一个新的对象返回

可以看到一次 get 操作最少 0 次加锁，最大 N（N 等于 MAXPROCS）次加锁。

归还对象的过程：
1. 固定到某个 P，如果私有对象为空则放到私有对象
2. 否则加入到该 P 子池的共享列表中（需要加锁）
3. 可以看到一次 put 操作最少 0 次加锁，最多 1 次加锁

由于 goroutine 具体会分配到那个 P 执行是 golang 的协程调度系统决定的，因此在 MAXPROCS>1 的情况下，多 goroutine 用同一个 sync.Pool 的话，各个 P 的子池之间缓存的对象是否平衡以及开销如何是没办法准确衡量的。但如果 goroutine 数目和缓存的对象数目远远大于 MAXPROCS 的话，概率上说应该是相对平衡的。

## 总结
sync.Pool 的定位不是做类似连接池的东西，它的用途仅仅是增加对象重用的几率，减少 gc 的负担，而开销方面也不是很便宜的。

推荐阅读 [fmt](https://golang.org/pkg/fmt/)，这是 sync.Pool 的最佳实践
