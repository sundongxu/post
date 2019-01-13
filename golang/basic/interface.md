---
title: Go Interface 从理解到深入
date: 2018-09-13T18:24:50+08:00
categories:
    - Golang
tags: 
    - golang
    - interface
---

# Go Interface 从理解到深入
如果说 goroutine 和 channel 是 Go 并发的两大基石，那么接口是 Go 语言编程中数据类型的关键。在 Go 语言的实际编程中，几乎所有的数据结构都围绕接口展开，接口是 Go 语言中所有数据结构的核心

Go 不是一种典型的 OO 语言，它在语法上不支持类和继承的概念

没有继承是否就无法拥有多态行为了呢？答案是否定的，Go 语言引入了一种新类型—Interface，它在效果上实现了类似于 C++ 的 “多态” 概念，虽然与 C++ 的多态在语法上并非完全对等，但至少在最终实现的效果上，它有多态的影子

虽然 Go 语言没有类的概念，但它支持的数据类型可以定义对应的 method(s)。本质上说，所谓的 method(s) 其实就是函数，只不过与普通函数相比，这类函数是作用在某个数据类型上的，所以在函数签名中，会有个 receiver(接收器) 来表明当前定义的函数会作用在该 receiver 上


Go 语言支持的除 Interface 类型外的任何其它数据类型都可以定义其 method（而并非只有 struct 才支持 method），只不过实际项目中，method(s) 多定义在 struct 上而已。
从这一点来看，我们可以把 Go 中的 struct 看作是不支持继承行为的轻量级的 “类”，这一点比较类似 Abstract Class

从语法上看，Interface 定义了一个或一组 method(s)，这些 method(s) 只有函数签名，没有具体的实现代码（有没有联想起 C++ 中的虚函数？）。若某个数据类型实现了 Interface 中定义的那些被称为 "methods" 的函数，则称这些数据类型实现（implement）了 interface。这是我们常用的 OO 方式

我们先看一个例子感受一下，接口是如何使用的，再深入了解。

```
type Stringer interface {
    String() string
}

type S struct {
     i int
}

func (s *S) String() string {
    return fmt.Sprintf("%d", s.i)
}

func Print(s Stringer) {
    println(s.String())
}

func DynamicPrint(any interface{}) {
   if s, ok := any.(Stringer); ok {
       Print(s)
   }
}

func main() {
   var s S
   s.i = 123456789
   Print(&s)
   DynamicPrint(&s)
}
```

## 1. interface 实现

interface 的实现位于 [runtime/runtime2.go](https://golang.org/src/runtime/runtime2.go)

根据 interface 是否包含有 method，底层实现上用两种 struct 来表示：iface 和 eface

空接口 interface{} 通过 eface 结构体实现，对于 Golang 中的大部分数据类型都可以抽象出来 _type 结构，同时针对不同的类型还会有一些其他信息

```
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```

相比于 empty interface，non-empty 要包含一些 method，底层实现使用 iface。method 的具体实现存放在 itab.fun 变量里。

```
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

### 1.1 eface 解析

空接口 (eface) 有两个域，所指向对象的类型信息 (_type) 和数据指针 (data)。先看看 **_type** 字段

`_type` 也位于 runtime/runtime2.go 文件内，所有类型信息结构体的公共部分

```
type _type struct {
    size       uintptr  // 类型的大小
    ptrdata    uintptr  // size of memory prefix holding all pointers
    hash       uint32   // 类型的 Hash 值
    tflag      tflag    // 类型的 Tags
    align      uint8    // 结构体内对齐
    fieldalign uint8    // 结构体作为 field 时的对齐
    kind       uint8    // 类型编号 定义于 runtime/typekind.go
    alg        *typeAlg // 类型元方法 存储 hash 和 equal 两个操作。map key 便使用 key 的_type.alg.hash(k) 获取 hash 值
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte   //  GC 相关信息
    str       nameOff // 类型名字的偏移
    ptrToThis typeOff
}
```

_type 是 go 所有类型的公共描述，里面包含 GC，反射等需要的细节，它决定 data 应该如何解释和操作。

各个类型所需要的类型描述是不一样的，比如 chan，除了 chan 本身外，还需要描述其元素类型，而 map 则需要 key 类型信息和 value 类型信息等。

```
// src/runtime/type.go
// ptrType represents a pointer type.
type ptrType struct {
    typ  _type  // 指针类型
    elem *_type // 指针所指向的元素类型
}
type chantype struct {
    typ  _type  // channel 类型
    elem *_type // channel 元素类型
    dir  uintptr
}
type maptype struct {
    typ           _type
    key           *_type
    elem          *_type
    bucket        *_type // internal type representing a hash bucket
    hmap          *_type // internal type representing a hmap
    keysize       uint8  // size of key slot
    indirectkey   bool   // store ptr to key instead of key itself
    valuesize     uint8  // size of value slot
    indirectvalue bool   // store ptr to value instead of value itself
    bucketsize    uint16 // size of bucket
    reflexivekey  bool   // true if k==k for all keys
    needkeyupdate bool   // true if we need to update key on an overwrite
}
```

这些类型信息的第一个字段都是 `_type` (类型本身的信息)，接下来是一堆类型需要的其它详细信息 (如子类型信息)，这样在进行类型相关操作时，可通过一个字 `(typ *_type)` 即可表述所有类型，然后再通过_type.kind 可解析出其具体类型，最后通过地址转换即可得到类型完整的 `_type tree`，参考 reflect.Type.Elem() 函数:

```
func (t *rtype) Elem() Type {
    switch t.Kind() {
    case Array:
        tt := (*arrayType)(unsafe.Pointer(t))
        return toType(tt.elem)
    case Chan:
        tt := (*chanType)(unsafe.Pointer(t))
        return toType(tt.elem)
    case Map:
        tt := (*mapType)(unsafe.Pointer(t))
        return toType(tt.elem)
    case Ptr:
        tt := (*ptrType)(unsafe.Pointer(t))
        return toType(tt.elem)
    case Slice:
        tt := (*sliceType)(unsafe.Pointer(t))
        return toType(tt.elem)
    }
    panic("reflect: Elem of invalid type")
}
```

## 1.2 iface 解析

iface 结构体表示非空接口

```
// runtime/runtime2.go
// 非空接口
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// 非空接口的类型信息
type itab struct {
    inter  *interfacetype // 接口定义的类型信息
    _type  *_type         // 接口实际指向值的类型信息
    link   *itab
    bad    int32
    inhash int32
    fun    [1]uintptr // 接口方法实现列表，即函数地址列表，按字典序排序
}

// runtime/type.go
// 非空接口类型，接口定义，包路径等。
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod // 接口方法声明列表，按字典序排序
}

// 接口的方法声明
type imethod struct {
    name nameOff // 方法名
    ityp typeOff // 描述方法参数返回值等细节
}
```

非空接口 (iface) 本身除了可以容纳满足其接口的对象之外，还需要保存其接口的方法，因此除了 data 字段，iface 通过 tab 字段描述非空接口的细节，包括接口方法定义，接口方法实现地址，接口所指类型等。iface 是非空接口的实现，而不是类型定义，iface 的真正类型为 interfacetype，其第一个字段仍然为描述其自身类型的 `_type` 字段

```
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
```

为了提高查找效率，runtime 中实现 (interface_type, concrete_type) -> itab(包含具体方法实现地址等信息) 的 hash 表。并不是每次接口赋值都要去检查一次对象是否符合接口要求，而是只在第一次生成 itab 信息，之后通过 hash 表即可找到 itab 信息。

下面只是一些简单的描述，更详细的可以看 [runtime/iface.go](https://golang.org/src/runtime/iface.go)

```
// runtime/iface.go
const (
   hashSize = 1009
)
var (
   ifaceLock mutex // lock for accessing hash
   hash      [hashSize]*itab
)
// 简单的 Hash 算法
func itabhash(inter *interfacetype, typ *_type) uint32 {
   h := inter.typ.hash
   h += 17 * typ.hash
   return h % hashSize
}

// 根据 interface_type 和 concrete_type 获取或生成 itab 信息
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
    ...
    // 算出 hash key
    h := itabhash(inter, typ)
    var m *itab
    ...
    // 遍历 hash slot 链表
    for m = (*itab)(atomic.Loadp(unsafe.Pointer(&hash[h]))); m != nil; m = m.link {
        // 如果在 hash 表中找到则返回
        if m.inter == inter && m._type == typ {
            if m.bad {
                if !canfail {
                    additab(m, locked != 0, false)
                }
                m = nil
            }
            ...
            return m
        }
      }
    }
    // 如果没有找到，则尝试生成 itab(会检查是否满足接口)
    m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
    m.inter = inter
    m._type = typ
    additab(m, true, canfail)
    if m.bad {
        return nil
    }
    return m
}

// 检查 concrete_type 是否符合 interface_type 并且创建对应的 itab 结构体 将其放到 hash 表中
func additab(m *itab, locked, canfail bool) {
   inter := m.inter
   typ := m._type
   x := typ.uncommon()
   ni := len(inter.mhdr)
   nt := int(x.mcount)
   xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
   j := 0
   for k := 0; k < ni; k++ {
      i := &inter.mhdr[k]
      itype := inter.typ.typeOff(i.ityp)
      name := inter.typ.nameOff(i.name)
      iname := name.name()
      ipkg := name.pkgPath()
      if ipkg == "" {
         ipkg = inter.pkgpath.name()
      }
      for ; j < nt; j++ {
         t := &xmhdr[j]
         tname := typ.nameOff(t.name)
         // 检查方法名字是否一致
         if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
            pkgPath := tname.pkgPath()
            if pkgPath == "" {
               pkgPath = typ.nameOff(x.pkgpath).name()
            }
            // 是否导出或在同一个包
            if tname.isExported() || pkgPath == ipkg {
               if m != nil {
                    // 获取函数地址，并加入到 itab.fun 数组中
                  ifn := typ.textOff(t.ifn)
                  *(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
               }
               goto nextimethod
            }
         }
      }
      // didn't find method
      if !canfail {
         if locked {
            unlock(&ifaceLock)
         }
         panic(&TypeAssertionError{"", typ.string(), inter.typ.string(), iname})
      }
      m.bad = true
      break
   nextimethod:
   }
   if !locked {
      throw("invalid itab locking")
   }
   // 加到 Hash Slot 链表中
   h := itabhash(inter, typ)
   m.link = hash[h]
   m.inhash = true
   atomicstorep(unsafe.Pointer(&hash[h]), unsafe.Pointer(m))
}
```

## 1.3 接口赋值

```
type MyInterface interface {
   Print()
}
type MyStruct struct{}
func (ms MyStruct) Print() {}
func main() {
   a := 1
   b := "str"
   c := MyStruct{}
   var i1 interface{} = a
   var i2 interface{} = b
   var i3 MyInterface = c
   var i4 interface{} = i3
   var i5 = i4.(MyInterface)
   fmt.Println(i1, i2, i3, i4, i5)
}
```

对于接口间的赋值，将 iface 赋给 eface 比较简单，直接提取 eface 的 interfacetype 和 data 赋给 iface 即可。而反过来，则需要使用接口断言，接口断言通过 assertE2I, assertI2I 等函数来完成，这类 assert 函数根据使用方调用方式有两个版本

```
i5 := i4.(MyInterface)      // call conv.assertE2I
i5, ok := i4.(MyInterface)  // call conv.AssertE2I2
```

下面看一下几个常用的 conv 和 assert 函数实现:

```
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    // TODO: We allocate a zeroed object only to overwrite it with actual data.
    // Figure out how to avoid zeroing. Also below in convT2Eslice, convT2I, convT2Islice.
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}

func assertE2I(inter *interfacetype, e eface) (r iface) {
    t := e._type
    if t == nil {
        // explicit conversions require non-nil interface value.
        panic(&TypeAssertionError{"","", inter.typ.string(), ""})
    }
    r.tab = getitab(inter, t, false)
    r.data = e.data
    return
}
```

在 assertE2I 中，我们看到了 getitab 函数，即 i5=i4.(MyInterface) 中，会去判断 i4 的 concretetype(MyStruct) 是否满足 MyInterface 的 interfacetype，由于前面我们执行过 var i3 MyInterface = c，因此 hash[itabhash(MyInterface, MyStruct)] 已经存在 itab，所以无需再次检查接口是否满足，从 hash 表中取出 itab 即可 (里面针对接口的各个方法实现地址都已经初始化完成)。

而在 go1.10 中，有一些优化:

convT2x 针对简单类型 (如 int32,string,slice) 进行特例化优化 (避免 typedmemmove):

```
convT2E16, convT2I16
convT2E32, convT2I32
convT2E64, convT2I64
convT2Estring, convT2Istring
convT2Eslice, convT2Islice
convT2Enoptr, convT2Inoptr
```

优化了剩余对 convT2I 的调用:

由于 itab 由编译器生成, 可以直接由编译器将 itab 和 elem 直接赋给 iface 的 tab 和 data 字段，避免函数调用和 typedmemmove

## 1.4 类型反射

类型反射无非就是将 eface{} 的 `_type` 和 `data` 字段取出进行解析，针对 TypeOf 的实现很简单:

```
func TypeOf(i interface{}) Type {
    eface := *(*emptyInterface)(unsafe.Pointer(&i))
    return toType(eface.typ)
}

```

reflect.Type.Elem() 仅对复合类型有效 (Array,Ptr,Map,Chan,Slice)，取出其中的子类型。之前 Elem 的实现已经说明。

reflect.ValueOf 则要复杂一些，因为它需要根据 type 来决定数据应该如何被解释，因此实际上 reflect.Value 也包含类型信息，并且通过一个 flag 字段来标识只读属性，是否为指针等

```
type Value struct {
    // 值的类型
    typ *rtype
    // 立即数或指向数据的指针
    ptr unsafe.Pointer
    // type flag uintptr
    // 指明值的类型，是否只读，ptr 字段是否是指针等
    flag
}

func ValueOf(i interface{}) Value {
    if i == nil {
        return Value{}
    }
    escapes(i)
    return unpackEface(i)
}

// 将数据从 interface{} 解包为 reflec.Value
// unpackEface converts the empty interface i to a Value.
func unpackEface(i interface{}) Value {
    e := (*emptyInterface)(unsafe.Pointer(&i))
    // NOTE: don't read e.word until we know whether it is really a pointer or not.
    t := e.typ
    if t == nil {
        return Value{}
    }
    f := flag(t.Kind())
    if ifaceIndir(t) {
        f |= flagIndir
    }
    return Value{t, e.word, f}
}


// 将数据由 reflect.Value 打包为 interface{}
// packEface converts v to the empty interface.
func packEface(v Value) interface{} {
    t := v.typ
    var i interface{}
    e := (*emptyInterface)(unsafe.Pointer(&i))
    // First, fill in the data portion of the interface.
    switch {
    case ifaceIndir(t):
        if v.flag&flagIndir == 0 {
            panic("bad indir")
        }
        // Value is indirect, and so is the interface we're making.
        ptr := v.ptr
        if v.flag&flagAddr != 0 {
            // TODO: pass safe boolean from valueInterface so
            // we don't need to copy if safe==true?
            c := unsafe_New(t)
            typedmemmove(t, c, ptr)
            ptr = c
        }
        e.word = ptr
    case v.flag&flagIndir != 0:
        // Value is indirect, but interface is direct. We need
        // to load the data at v.ptr into the interface data word.
        e.word = *(*unsafe.Pointer)(v.ptr)
    default:
        // Value is direct, and so is the interface.
        e.word = v.ptr
    }
    // Now, fill in the type portion. We're very careful here not
    // to have any operation between the e.word and e.typ assignments
    // that would let the garbage collector observe the partially-built
    // interface value.
    e.typ = t
    return i
}

// reflect.Value 的 Elem() 方法仅对引用类型 (Ptr 和 Interface{}) 有效，返回其引用的值
// Elem returns the value that the interface v contains
// or that the pointer v points to.
// It panics if v's Kind is not Interface or Ptr.
// It returns the zero Value if v is nil.
func (v Value) Elem() Value {
    k := v.kind()
    switch k {
    case Interface:
        var eface interface{}
        if v.typ.NumMethod() == 0 {
            eface = *(*interface{})(v.ptr)
        } else {
            eface = (interface{})(*(*interface {
                M()
            })(v.ptr))
        }
        x := unpackEface(eface)
        if x.flag != 0 {
            x.flag |= v.flag.ro()
        }
        return x
    case Ptr:
        ptr := v.ptr
        if v.flag&flagIndir != 0 {
            ptr = *(*unsafe.Pointer)(ptr)
        }
        // The returned value's address is v's value.
        if ptr == nil {
            return Value{}
        }
        tt := (*ptrType)(unsafe.Pointer(v.typ))
        typ := tt.elem
        fl := v.flag&flagRO | flagIndir | flagAddr
        fl |= flag(typ.Kind())
        return Value{typ, ptr, fl}
    }
    panic(&ValueError{"reflect.Value.Elem", v.kind()})
}
```

## 2. 为什么用 interface

主要是以下几点理由：
- (伪) 泛型编程
- 隐藏具体实现，隔离依赖

### 2.1. 泛型编程

严格来说，在 Golang 中并不支持泛型编程。在 C++ 等高级语言中使用泛型编程非常的简单，所以泛型编程一直是 Golang 诟病最多的地方。但是使用 interface 我们可以实现泛型编程，如下是一个参考示例：[sort](https://golang.org/src/sort/sort.go)

```
// Package sort provides primitives for sorting slices and user-defined
// collections.
package sort

// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

```
package main

import (
    "fmt"
    "sort"
)

type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// ByAge implements sort.Interface for []Person based on
// the Age field.
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
    people := []Person{
        {"Bob", 31},
        {"John", 42},
        {"Michael", 17},
        {"Jenny", 26},
    }

    fmt.Println(people)
    // There are two ways to sort a slice. First, one can define
    // a set of methods for the slice type, as with ByAge, and
    // call sort.Sort. In this first example we use that technique.
    sort.Sort(ByAge(people))
    fmt.Println(people)

    // The other way is to use sort.Slice with a custom Less
    // function, which can be provided as a closure. In this
    // case no methods are needed. (And if they exist, they
    // are ignored.) Here we re-sort in reverse order: compare
    // the closure with ByAge.Less.
    sort.Slice(people, func(i, j int) bool {
        return people[i].Age > people[j].Age
    })
    fmt.Println(people)

}
```

Sort 函数的形参是一个 interface，包含了三个方法：Len()，Less(i,j int)，Swap(i, j int)。使用的时候不管数组的元素类型是什么类型（int, float, string…），只要我们实现了这三个方法就可以使用 Sort 函数，这样就实现了 ** 泛型编程 **。

### 2.2 隐藏具体实现
隐藏具体实现，这个很好理解。比如我设计一个函数给你返回一个 interface，那么你只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。

例如我们常用的 context 包，就是这样的，context 最先由 google 提供，现在已经纳入了标准库，而且在原有 context 的基础上增加了：cancelCtx，timerCtx，valueCtx

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```

WithCancel 函数返回的还是一个 Context interface，但是这个 interface 的具体实现是 cancelCtx struct

```
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields
    done     chan struct{}         // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})
    }
    d := c.done
    c.mu.Unlock()
    return d
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}
```

尽管内部实现上下面三个函数返回的具体 struct （都实现了 Context interface）不同，但是对于使用者来说是完全无感知的。

```
// 返回 cancelCtx
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 返回 timerCtx
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
// 返回 valueCtx
func WithValue(parent Context, key, val interface{}) Context
```

## 3. interface 与 nil 的比较

我们先来看个例子

```
package main

import "fmt"

func Foo(x interface{}) {
    if x == nil {
        fmt.Println("empty interface")
        return
    }
    fmt.Println("non-empty interface")
}

func main() {
    var x *int = nil
    Foo(x)
}
```

你猜，输出是什么，我们来运行一下 [5v8a-tVXNXN](https://play.golang.org/p/5v8a-tVXNXN)

```
non-empty interface
```

wtf? 为什么 nil 传值进入 Foo 结果竟然不是 nil ？？


但是，我们在深入探讨 interface 的实现后，应该明白一点:

一个 interface{} 类型的变量包含了 2 个指针，一个指针指向值的类型，另外一个指针指向实际的值

对一个 interface{} 类型的 nil 变量来说，它的两个指针都是 0；

但是 x *int 传进去后，指向的类型的指针不为 0 了，因为有类型了， 所以比较为 false。 interface 类型比较， 要是 两个指针都相等， 才能相等
