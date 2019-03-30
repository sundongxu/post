---
title: "new 与 make 的差别"
date: 2019-03-30T19:02:11+08:00
categories:
    - Golang
tags: 
    - Golang
    - tips
---

Go语言中new和make是内建的两个函数，主要用来创建分配类型内存

## new(T) 返回的是 T 的指针
这是 `new` 函数的注释:

```
// The new built-in function allocates memory. The first argument is a type,
// not a value, and the value returned is a pointer to a newly
// allocated zero value of that type.
func new(Type) *Type
```

`new(T)` 为一个 `T` 类型新值分配空间并将此空间初始化为 `T` 的**零值**，返回的是新值的地址，也就是 `T` 类型的指针 `*T`，该指针指向 `T` 的新分配的零值

> 对于有哪些零值可以看看 [之前的博客](https://zhaolion.com/post/golang/upgrade/memory.layout/) 里面有基础类型的零值

## make
`make` 函数的注释:
```
The make built-in function allocates and initializes an object of type slice, map, or chan (only). Like new, the first argument is a type, not a value. Unlike new, make's return type is the same as the type of its argument, not a pointer to it. The specification of the result depends on the type:

Slice: The size specifies the length. The capacity of the slice is
equal to its length. A second integer argument may be provided to
specify a different capacity; it must be no smaller than the
length. For example, make([]int, 0, 10) allocates an underlying array
of size 10 and returns a slice of length 0 and capacity 10 that is
backed by this underlying array.
Map: An empty map is allocated with enough space to hold the
specified number of elements. The size may be omitted, in which case
a small starting size is allocated.
Channel: The channel's buffer is initialized with the specified
buffer capacity. If zero, or the size is omitted, the channel is
unbuffered.

func make(t Type, size ...IntegerType) Type
```

`make` 只能用于 `slice`，`map`，`channel` 三种类型，`make(T, args)` 返回的是初始化之后的 `T` 类型的值，这个新值并不是 `T` 类型的零值，也不是指针 `*T`，是经过初始化之后的 `T` 的引用

## make vs new
[之前的博客](https://zhaolion.com/post/golang/upgrade/memory.layout/)里面有提到 `make` 和 `new` 在内存排布上差别，直接贴出来:

new:

```
+---------+
| pointer |     s = new([3]int)
+---------+
    |
    +---+---+---+
    | 0 | 0 | 0 | [3]int
    +---+---+---+
```

make:

```
+---------+---------+---------+
| pointer |  len=1  |  cap=3  | slice=make([]int,1,3) 
+---------+---------+---------+
    |
    +---+---+---+ 
    | 0 | 0 | 0 | [3]int
    +---+---+---+

+---------+
| pointer | map = make(map[string]int); 实际返回的是⼀一个指针包装对象。
+---------+
    |
    ..................
    .                .
    . hashmap.c Hmap . 
    .                . 
    ..................

+---------+
| pointer | channel = make(chan int); 实际返回的是⼀一个指针包装对象。 
+---------+ 
    |
    ................ 
    .              . 
    . chan.c Hchan . 
    .              .
    ................
```

总结:

- 二者都是内存的分配（堆上），但是 `make` 只用于 `slice`、`map` 以及 `channel`的初始化（非零值
- `new` 用于类型的内存分配，并且内存置为零
- `make` 返回的还是这三个引用类型本身；而 `new` 返回的是指向类型的指针
