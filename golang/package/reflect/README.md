# 初步了解 golang reflect pkg
[TOC]

阅读这篇文章之前，建议先熟悉官方文档 [pkg/reflect](https://golang.org/pkg/reflect/)

Golang 语言实现了反射，反射机制就是在运行时动态的调用对象的方法和属性，官方自带的 reflect 包就是反射相关的，只要包含这个包就可以使用。实际使用中可以先不考虑使用 reflect 对性能的影响，先实现功能，再利用 benchmark test 去优化。

** 什么时候应该用 reflect**

1. 首先你得确认你会使用 reflect pkg，并且不是乱用
2. 更好的抽象和约束，减少 bug 几率
3. 提升代码的易读性
4. 提高开发效率


## 1 理解 Type & Kind & Value
reflect package 提供了 3 个重要的结构体 [Type](https://golang.org/pkg/reflect/#Type)、[Kind](https://golang.org/pkg/reflect/#Kind) 和 [Value](https://golang.org/pkg/reflect/#Value):
- Type: 就是 Go concrete type, 例如 int/string/bool/customStruct ...
- Kind: 就是 Go static type(固定的)，例如 Boot/Int/Struct ...
- Value: 也就是 Go value, 承载变量 / 常量的值


> `type Kind uint`
> `Kind` 用途: 用于 runtime 和 compiler 为变量分配变量内存布局和函数分配堆栈

Type 和 Kind，在真正的使用中是隐藏在背后的，并且在不阅读相关文档的情况下，其实会不明白有什么差别。举个例子来说明其中的差别:

** 例子:**
参考一个典型的 struct:

```golang
type User struct {
    Name string
}
```

如果实例化一个 `User` 结构体 `user1 := User{"zhaolion"}` 那么 `user1` 对象的 Type 和 Kind 分别是:
- Type: `pkg.User`
- Kind: `Struct`

如果实例化一个 `User` 结构体 `user2 := &User{"zhaolion"}` 那么 `user2` 对象的 Type 和 Kind 分别是:
- Type: `*pkg.User`
- Kind: `Ptr`

运行结果可以参考这份代码 [V6VgdLLeBWb](https://play.golang.org/p/V6VgdLLeBWb)

```golang
package main

import (
	"fmt"
	"reflect"
)

type User struct {
    Name string
}


func main() {
	user1 := User{"zhaolion"}
	user2 := &User{"zhaolion"}

	fmt.Printf("user1 kind: %s, type: %s\n", reflect.ValueOf(user1).Kind(), reflect.TypeOf(user1))
	fmt.Printf("user2 kind: %s, type: %s", reflect.ValueOf(user2).Kind(), reflect.TypeOf(user2))
}
```

输出:

```
user1 kind: struct, type: main.User
user2 kind: ptr, type: *main.User
```

那么有什么样的方式可以让我们直接获取到变量内部的信息 (Type/Value) 呢？

```
/ ValueOf returns a new Value initialized to the concrete value
// stored in the interface i.  ValueOf(nil) returns the zero
func ValueOf(i interface{}) Value {...}
```

```
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {...}
```

这样就能拿到 `Object` 的 `Type` 和 `Value`，做自己想做的事情

更详细的细节，可以去看源码，这里简单介绍一下 `Type` 和 `Value`:

```golang
type Type interface {
    // 变量的内存对齐，返回 rtype.align
    Align() int

    // struct 字段的内存对齐，返回 rtype.fieldAlign
    FieldAlign() int

    // 根据传入的 i，返回方法实例，表示类型的第 i 个方法
    Method(int) Method

    // 根据名字返回方法实例，这个比较常用
    MethodByName(string) (Method, bool)

    // 返回类型方法集中可导出的方法的数量
    NumMethod() int

    // 只返回类型名，不含包名
    Name() string

    // 返回导入路径，即 import 路径
    PkgPath() string

    // 返回 rtype.size 即类型大小，单位是字节数
    Size() uintptr

    // 返回类型名字，实际就是 PkgPath() + Name()
    String() string

    // 返回 rtype.kind，描述一种基础类型
    Kind() Kind

    // 检查当前类型有没有实现接口 u
    Implements(u Type) bool

    // 检查当前类型能不能赋值给接口 u
    AssignableTo(u Type) bool

    // 检查当前类型能不能转换成接口 u 类型
    ConvertibleTo(u Type) bool

    // 检查当前类型能不能做比较运算，其实就是看这个类型底层有没有绑定 typeAlg 的 equal 方法。
    // 打住！不要去搜 typeAlg 是什么，不然你会陷进去的！先把本文看完。
    Comparable() bool

    // 返回类型的位大小，但不是所有类型都能调这个方法，不能调的会 panic
    Bits() int

    // 返回 channel 类型的方向，如果不是 channel，会 panic
    ChanDir() ChanDir

    // 返回函数类型的最后一个参数是不是可变数量的，"..." 就这样的，同样，如果不是函数类型，会 panic
    IsVariadic() bool

    // 返回所包含元素的类型，只有 Array, Chan, Map, Ptr, Slice 这些才能调，其他类型会 panic。
    Elem() Type

    // 返回 struct 类型的第 i 个字段，不是 struct 会 panic，i 越界也会 panic
    Field(i int) StructField

    // 跟上边一样，不过是嵌套调用的，比如 [1, 2] 就是说返回当前 struct 的第 1 个 struct 的第 2 个字段，适用于 struct 本身嵌套的类型
    FieldByIndex(index []int) StructField

    // 按名字找 struct 字段，第二个返回值 ok 表示有没有
    FieldByName(name string) (StructField, bool)

    // 按函数名找 struct 字段，因为 struct 里也可能有类型是 func 的嘛
    FieldByNameFunc(match func(string) bool) (StructField, bool)

    // 返回函数第 i 个参数的类型，不是 func 会 panic
    In(i int) Type

    // 返回 map 的 key 的类型，不是 map 会 panic
    Key() Type

    // 返回 array 的长度，不是 array 会 panic
    Len() int

    // 返回 struct 字段数量，不是 struct 会 panic
    NumField() int

    // 返回函数的参数数量，不是 func 会 panic
    NumIn() int

    // 返回函数的返回值数量，不是 func 会 panic
    NumOut() int

    // 返回函数第 i 个返回值的类型，不是 func 会 panic
    Out(i int) Type
}
```

```golang
type Value struct {
    // 反射出来此值的类型，rtype 是啥往上看，但可别弄错了，这 typ 是未导出的，从外部调不到 Type 接口的方法
    typ *rtype

    // 数据形式的指针值
    ptr unsafe.Pointer

    // 保存元数据
    flag
}
```

Value 的方法太多了，参考开头的官方文档吧，挑几个重点：

```golang
// 前提 v 是一个 func，然后调用 v，并传入 in 参数，第一个参数是 in[0]，第二个是 in[1]，以此类推
func (v Value) Call(in []Value) []Value

// 返回 v 的接口值或者指针
func (v Value) Elem() Value

// 前提 v 是一个 struct，返回第 i 个字段，这个主要用于遍历
func (v Value) Field(i int) Value

// 前提 v 是一个 struct，根据字段名直接定位返回
func (v Value) FieldByName(name string) Value

// 前提 v 是 Array, Slice, String 之一，返回第 i 个元素，主要也是用于遍历，注意不能越界
func (v Value) Index(i int) Value

// 判断 v 是不是 nil，只有 chan, func, interface, map, pointer, slice 可以用，其他类型会 panic
func (v Value) IsNil() bool

// 判断 v 是否合法，如果返回 false，那么除了 String() 以外的其他方法调用都会 panic，事前检查是必要的
func (v Value) IsValid() bool

// 前提 v 是个 map，返回对应 value
func (v Value) MapIndex(key Value)

// 前提 v 是个 map，返回所有 key 组成的一个 slice
func (v Value) MapKeys() []Value

// 前提 v 是个 struct，返回字段个数
func (v Value) NumField() int

// 赋值
func (v Value) Set(x Value)

// 类型
func (v Value) Type() Type
```

## 2 根据 Type 创建变量
如何根据类型创建变量?

- 我们想要拿到变量的类型信息，那就需要使用 `reflect.Type`
- 再根据类型创建一个对应的 · 零值 (Zero values)·，也就是未初始化的变量

```
`Type` => `Value` => `interface{}`
```

### 2.1 创建基础类型变量

对于这些 Golang 的基础类型可以使用 `reflect.Zero` 可以创建对应的变量值而不需要做任何处理:

```
        Bool
        Int
        Int8
        Int16
        Int32
        Int64
        Uint
        Uint8
        Uint16
        Uint32
        Uint64
        Uintptr
        Float32
        Float64
        Complex64
        Complex128
        String
        UnsafePointer
```

可以使用下面这个函数来创建基础类型的变量对应的值变量，如果想要真正使用，只需要提取一些类型信息包装一下就行。

```golang
func Create(t reflect.Type) reflect.Value {
  return reflect.Zero(t)
}
```

我们平时使用的对象需要具体的类型，因此需要从 `reflect.Value` 提取 (Extract) 具体的类型和值。

golang 提供了许多方法来为基础类型提取对象的值和类型

#### 2.1.1 Extract Integer values
Golang 有 5 种 Integer:

```
        Int
        Int8
        Int16
        Int32
        Int64
```

每种 `Integer` 都是不同的类型，因此需要转换成对应的类型。

reflect package 提供了 `reflect.Type > func (v Value) Int() int64` 来做这件事。

为什么返回值是 `int64` ?
> 这是因为其他几种 `Integer` 类型都可以在 `int64` 中进行编码

只需要进行一些基本的类型转换就可以获取想要的类型对象：

```golang
// Extract Int64:
func extractInt64(v reflect.Value) (int64, error) {
  if reflect.Kind() != reflect.Int64 {
    panic("invalid input")
  }
  var intVal int64
  intVal = v.Int()
  return intVal, nil
}
// Extract Int32
func extractInt32(v reflect.Value) (int32, error) {
  if reflect.Kind() != reflect.Int32 {
    return int32(0), errors.New("Invalid input")
  }
  var intVal int64
  intVal = v.Int()
  return int32(intVal), nil
}
// Extract Int16
func extractInt16(v reflect.Value) (int16, error) {
  if reflect.Kind() != reflect.Int16 {
    panic("invalid input")
  }
  var intVal int64
  intVal = v.Int()
  return int16(intVal), nil
}
// Extract Int8
func extractInt8(v reflect.Value) (int8, error) {
  if reflect.Kind() != reflect.Int8 {
    panic("invalid input")
  }
  var intVal int64
  intVal = v.Int()
  return int8(intVal), nil
}
// Extract Int
func extractInt(v reflect.Value) (int, error) {
  if reflect.Kind() != reflect.Int {
    panic("invalid input")
  }
  var intVal int64
  intVal = v.Int()
  return int(intVal), nil
}
```

#### 2.1.2 Extract Unsigned Integers
Golang 有 5 种 Unsigned Integer types:

```
        Uint
        Uint8
        Uint16
        Uint32
        Uint64
```

reflect package 提供了 `reflect.Type > func (v Value) Uint() uint64` 来做这件事。

为什么返回值是 `uint64` ?
> 这是因为其他几种 `Unsigned Integer` 类型都可以在 `uint64` 中进行编码

只需要进行一些基本的类型转换就可以获取想要的类型对象：

```golang
// Extract Uint64
func extractUint64(v reflect.Value) (uint64, error) {
  if reflect.Kind() != reflect.Uint64 {
    panic("invalid input")
  }
  var uintVal uint64
  uintVal = v.Uint()
  return uintVal, nil
}
// Extract Uint32
func extractUint32(v reflect.Value) (uint32, error) {
  if reflect.Kind() != reflect.Uint32 {
    panic("invalid input")
  }
  var uintVal uint64
  uintVal = v.Uint()
  return uint32(uintVal), nil
}
// Extract Uint16
func extractUint16(v reflect.Value) (uint16, error) {
  if reflect.Kind() != reflect.Uint16 {
    panic("invalid input")
  }
  var uintVal uint64
  uintVal = v.Uint()
  return uint16(uintVal), nil
}
// Extract Uint8
func extractUint8(v reflect.Value) (uint8, error) {
  if reflect.Kind() != reflect.Uint8 {
    panic("invalid input")
  }
  var uintVal uint64
  uintVal = v.Uint()
  return uint8(uintVal), nil
}
// Extract Uint
func extractUint(v reflect.Value) (uint, error) {
  if reflect.Kind() != reflect.Uint {
    panic("invalid input")
  }
  var uintVal uint64
  uintVal = v.Uint()
  return uint(uintVal), nil
}
```

#### 2.1.3 Extract Boolean values
Boolean values 可以通过内建常量表示

reflect package 提供了 `reflect.Type > func (v Value) Bool() bool` 来做这件事。

```golang
// Extract Bool
func extractBool(v reflect.Value) (bool, error) {
  if reflect.Kind() != reflect.Bool {
    panic("invalid input")
  }
  return v.Bool(), nil
}
```

#### 2.1.4 Extract Floating Point Numbers
Golang 有 2 种 Floating Point Number types:

```
        Float32
        Float64
```

reflect package 提供了 `reflect.Type > func (v Value) Float() float64` 来做这件事。

只需要进行一些基本的类型转换就可以获取想要的类型对象：

```golang
// Extract Float64
func extractFloat64(v reflect.Value) (float64, error) {
  if reflect.Kind() != reflect.Float64 {
    panic("invalid input")
  }
  var floatVal float64
  floatVal = v.Float()
  return floatVal, nil
}
// Extract Float32
func extractFloat32(v reflect.Value) (float32, error) {
  if reflect.Kind() != reflect.Float32 {
    panic("invalid input")
  }
  var floatVal float64
  floatVal = v.Float()
  return float32(floatVal), nil
}
```

#### 2.1.5 Extract Complex Values
Golang 有 2 种 Complex Values:

```
        Complex64
        Complex128
```

reflect package 提供了 `reflect.Type > func (v Value) Complex() complex128` 来做这件事。

只需要进行一些基本的类型转换就可以获取想要的类型对象：

```golang
// Extract Complex128
func extractComplex128(v reflect.Value) (complex128, error) {
  if reflect.Kind() != reflect.Complex128 {
    panic("invalid input")
  }
  var complexVal complex128
  complexVal = v.Complex()
  return complexVal, nil
}
// Extract Complex64
func extractComplex64(v reflect.Value) (complex64, error) {
  if reflect.Kind() != reflect.Complex64 {
    panic("invalid input")
  }
  var complexVal complex128
  complexVal = v.Complex()
  return complex64(complexVal), nil
}
```

#### 2.1.6 Extract string values
Boolean values 也是通过内建常量 `String` 表示

reflect package 提供了 `reflect.Type > func (v Value) String() string` 来做这件事。

```golang
// Extract String
func extractString(v reflect.Value) (string, error) {
  if reflect.Kind() != reflect.String {
    panic("invalid input")
  }
  return v.String(), nil
}
```

#### 2.1.7 Extract Pointer values
Golang 有 2 种 Pointer values:

```
        Uintptr
        UnsafePointer
```

`Uintptr` and `UnsafePointer` 都是 `uint` 值，指向一块内存地址。

 不同之处在于:

 - `Uintptr` 被 `runtime` 进行过类型确认
 - `UnsafePointer` 则没有

> `UnsafePointer` 被用来进行内存排布相同之间的类型转换

获取这两种类型对象的内部函数有一点难理解，不过先接受吧 (可以仔细了解 runtime 再来理解):

- `Uintptr`: `reflect.Type > func (v Value) Addr() Value`
- `UnsafePointer`: `reflect.Type >  func (v Value) UnsafeAddr() uintptr`

```golang
// Extract Uintptr
func extractUintptr(v reflect.Value) (uintptr, error) {
  if reflect.Kind() != reflect.Uintptr {
    panic("invalid input")
  }
  var ptrVal uintptr
  ptrVal = v.Addr()
  return ptrVal, nil
}
// Extract UnsafePointer
func extractUnsafePointer(v reflect.Value) (unsafe.Pointer, error) {
  if reflect.Kind() != reflect.UnsafePointer {
    panic("invalid input")
  }
  var unsafeVal unsafe.Pointer
  unsafeVal = unsafe.Pointer(v.UnsafeAddr())
  return unsafeVal, nil
}
```


### 2.2 根据基本类型创建 '基本复合类型' 变量
自定义一些复合类型，如果类型的 Kind 是以下基本类型的话，那和基本类型的方法是差不多的

```
        Bool
        Int
        Int8
        Int16
        Int32
        Int64
        Uint
        Uint8
        Uint16
        Uint32
        Uint64
        Uintptr
        Float32
        Float64
        Complex64
        Complex128
        String
        UnsafePointer
```

假设自定义一个复合类型 `type Name string` 那么 `Name` 的 `Kind` 是 `string`，`Type` 是 `Name`，可以通过零值进行构造

```
type Name string

func CreateObject(t reflect.Type) reflect.Value {
    return reflect.Zero(t)
}

func extractObject(v reflect.Value) (Name, error) {
    if v.Type().String() != "Name" {
        panic("invalid input")
    }

    return v.String(), nil
}
```

### 2.3 根据复合类型创建复合类型变量
复合类型是包含了其他类型，例如 `Map``Struct` `Array` ...

复合类型对应的 Kind 列表:

```
        Array
        Chan
        Func
        Interface
        Map
        Ptr
        Slice
        Struct
```

对于以下复合类型，也能够像基本类型一样使用 Zero value 来创建空值变量，但是需要一些额外的初始化步骤。

```golang
        Array
        Chan
        Func
        Interface
        Map
        Ptr
        Slice
        Struct
```

#### 2.3.1 根据类型创建 Array
空的 `Array` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
 return reflect.Zero(t)
}
```

获得 `reflect.Value` 之后，可以通过 `reflect.Type > func ArrayOf(count int, elem Type) Type` 创建一个包含对应类型及数量的 `Array`

```golang
func CreateArray(t reflect.Type, length int) reflect.Value {
  var arrayType reflect.Type
  arrayType = reflect.ArrayOf(length, t)
  return reflect.Zero(arrayType)
}
```

`Array` 的元素类型是通过传入参数 `reflect.Type` 定义的，因此如果你想要将批量对象导出为 `Array`，最佳选择是处理 `Array` 为 `interface{}`(因为没有泛型，需要写很多导出函数）

```golang
func extractArray(v reflect.Value) (interface{}, error) {
  if v.Kind() != reflect.Array {
    panic("invalid input")
  }
  var array interface{}
  array = v.Interface()
  return array, nil
}
```

还有一种做法是 `reflect.Value > func (v Value) Slice(i, j int) Value` 来导出 `Array`，但是这个函数前提是 `reflect.Value` 已经被处理为一个 `addressable array`，最好还是使用 `Interface()` 函数，这样会更简单

#### 2.3.2 根据类型创建 Channel

空的 `Channel` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
 return reflect.Zero(t)
}
```

reflect package 提供了两个方法来组合创建 `Channel`，`ChanOf` 函数创建带函数签名的 `Channel` 类型，`MakeChan` 用来分配内存。

- `reflect.Type > func ChanOf(dir ChanDir, t Type) Type`
- `reflect.Value > func MakeChan(typ Type, buffer int) Value`

> ChanDir:
>
> - SendDir
> - RecvDir
> - BothDir

```golang
func CreateChan(t reflect.Type, buffer int) reflect.Value {
 chanType := reflect.ChanOf(reflect.BothDir, t)
 return reflect.MakeChan(chanType, buffer)
}
```

导出为具体类型的 `Channel`，也需要使用 `Interface()` 函数

```golang
func extractChan(v reflect.Value) (interface{}, error) {
  if v.Kind() != reflect.Chan {
    panic("invalid input")
  }
  var ch interface{}
  ch = v.Interface()
  return ch, nil
}
```

#### 2.3.3 根据类型创建 Function Object
FBI warning: 无法使用 `reflect.Zero` 创建 `Function Object`

reflect package 提供了两个方法来组合创建 `Function Object`，`FuncOf` 函数创建带函数签名的 `Function Object` 类型，`MakeFunc(Type, func(args []Value) (results []Value)) Value` 用来分配内存。

- `reflect.Type > func FuncOf(in, out []Type, variadic bool) Type`
- `reflect.Value > func MakeFunc(typ Type, fn func(args []Value) (results []Value)) Value`

看个例子:

```golang
func CreateFunc(
    fType reflect.Type,
    f func(args []reflect.Value) (results []reflect.Value)
) (reflect.Value, error)
{
    if fType.Kind() != reflect.Func {
        panic("invalid input")
    }

    var ins, outs *[]reflect.Type
    ins = new([]reflect.Type)
    outs = new([]reflect.Type)

    for i:=0; i<fType.NumIn(); i++ {
        *ins = append(*ins, fType.In(i))
    }

    for i:=0; i<fType.NumOut(); i++ {
        *outs = append(*outs, fType.Out(i))
    }
    var variadic bool
    variadic = fType.IsVariadic()
    return AllocateStackFrame(*ins, *outs, variadic, f), nil
}

func AllocateStackFrame(
    ins []reflect.Type,
    outs []reflect.Type,
    variadic bool,
    f func(args []reflect.Value) (results []reflect.Value)
) reflect.Value
{
    var funcType reflect.Type
    funcType = reflect.FuncOf(ins, outs, variadic)
    return reflect.MakeFunc(funcType, f)
}
```

获取函数类型的 `Type` 可以这么做:

先定义一个函数签名: `type fn func(int) (int)`

```
var funcVar fn
var funcType reflect.Type
funcType = reflect.TypeOf(funcVar)
```

满足 `fn` 的一个函数:

```golang
func doubler(input int) (int) {
 return input * 2
}
```

上面 `CreateFunc` 第二个参数是 `f func(args []reflect.Value) (results []reflect.Value)`, doubler 并不满足这个泛型函数。

为了满足泛型函数，可以这么做:

```goalng
func doublerGeneric(args []reflect.Value) (result []reflect.Value) {
    if len(args) != 1 {
        panic("invalid input")
    }

    if args[0].Kind() != reflect.Int {
        panic("invalid input args type")
    }

    var intVal int64
    intVal = args[0].Int()

    var doubleIntVal int
    doubleIntVal = doubler(int(intVal))

    var returnValue reflect.Value
    returnValue = reflect.ValueOf(doubleIntVal)

    return []reflect.Value{returnValue}
}
```

这样的话 `doublerGeneric` 即等价于 `doubler`，也满足泛型函数

最终可以这样使用:

```
func main() {
    var funcVar fn
    var funcType reflect.Type
    funcType = reflect.TypeOf(funcVar)
    v, err := CreateFunc(funcType, doublerGeneric)
    if err != nil {
        fmt.Printf("Error creating function %v\n", err)
    }

    input := 42

    ins := []reflect.Value([]reflect.Value{reflect.ValueOf(input)})
    outs := v.Call(ins)
    for i := range outs {
        fmt.Printf("%+v\n", outs[i].Interface())
    }
}

```

```
// Output: 84
```

#### 2.3.4 根据类型创建 Map Object
空的 `Map` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
    return reflect.Zero(t)
}
```

reflect package 提供了 `reflect.Type > func MapOf(key, elem Type) Type` 来创建 Key 和 Value 具体类型的 Map.

```golang
func CreateMap(key , elem reflect.Type) reflect.Value {
    var mapType reflect.Type
    mapType = reflect.MapOf(key, elem)
    return reflect.MakeMap(mapType)
}
```

使用 `Interface()` 导出具体的 Map 仍然是最佳选择

```golang
func extractMap(v reflect.Value) (interface{}, error) {
    if v.Kind() != reflect.Map {
        panic("invalid input")
    }
    var mapVal interface{}
    mapVal = v.Interface()
    return mapVal, nil
}
```

#### 2.3.5 根据类型创建 Ptr Object
空的 `Ptr` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
    return reflect.Zero(t)
}
```

reflect package 提供了 `reflect.Type > func PtrTo(t Type) Type` 来创建 `Ptr` object 指向 具体的 `Type` object

```golang
func CreatePtr(t reflect.Type) reflect.Value {
    var ptrType reflect.Type
    ptrType = reflect.PtrTo(t)
    return reflect.Zero(ptrType)
}
```

导出 `Ptr Object` 指向元素，可以使用以下方法导出 Value 然后处理成 `interface{}`

- `reflect.Value > func Indirect(v Value) Value`
- `reflect.Value > func (v Value) Elem() Value`

```golang
func extractElement(v reflect.Value) (interface{}, error) {
    if v.Kind() != reflect.Ptr {
        return nil, errors.New("invalid input")
    }
    var elem reflect.Value
    elem = v.Elem()

    var elem interface{}
    elem = v.Interface()
    return elem, nil
}
```
#### 2.3.6 根据类型创建 Slice Object
空的 `Slice` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
    return reflect.Zero(t)
}
```

reflect package 提供了 `reflect.Type > func SliceOf(t Type) Type` 来创建 `Slice` 包含具体 `Type` 的元素

```golang
func CreateSlice(t reflect.Type) reflect.Value {
    var sliceType reflect.Type
    sliceType = reflect.SliceOf(length, t)
    return reflect.Zero(sliceType)
}
```

使用 `Interface()` 导出具体的 `Slice` 仍然是最佳选择

```
func extractSlice(v reflect.Value) (interface{}, error) {
    if v.Kind() != reflect.Slice {
        return nil, errors.New("invalid input")
    }
    var slice interface{}
    slice = v.Interface()
    return slice, nil
}
```

#### 2.3.6 根据类型创建 Struct Object
空的 `Slice` 可以通过 `reflect.Zero` 创建:

```golang
func CreateCompositeObject(t reflect.Type) reflect.Value {
    return reflect.Zero(t)
}
```

reflect package 提供了 `reflect.Type >  func StructOf(fields []StructField) Type` 来创建 `Struct` 包含具体 `StructField` 的元素

```golang
func CreateStruct(fields []reflect.StructField) reflect.Value {
    var structType reflect.Type
    structType = reflect.StructOf(fields)
    return reflect.Zero(structType)
}
```

使用 `Interface()` 导出具体的 `Struct` 仍然是最佳选择

```
func extractStruct(v reflect.Value) (interface{}, error) {
    if v.Kind() != reflect.Struct {
        return nil, errors.New("invalid input")
    }
    var st interface{}
    st = v.Interface()
    return st, nil
}
```

## 3. addressable
Go 语言规范中规定了可寻址 (addressable) 对象的定义

```
For an operand x of type T, the address operation &x generates a pointer of type *T to x. The operand must be addressable, that is, either a variable, pointer indirection, or slice indexing operation; or a field selector of an addressable struct operand; or an array indexing operation of an addressable array. As an exception to the addressability requirement, x may also be a (possibly parenthesized) composite literal. If the evaluation of x would cause a run-time panic, then the evaluation of &x does too.
```

上面规范中的这段话规定， `x` 必须是可寻址的， 也就是说，它只能是以下几种方式：

- 一个变量: `&x`
- 指针引用 (pointer indirection): `&*x`
- slice 索引操作 (不管 slice 是否可寻址): `&s[1]`
- 可寻址 struct 的字段: `&point.X`
- 可寻址数组的索引操作: `&a[0]`
- [composite literal](https://golang.org/ref/spec#Composite_literals) 类型: `&struct{ X int }{1}`

下列情况 `x` 是不可以寻址的，你不能使用 `&x` 取得指针：

- 字符串中的字节
- map 对象中的元素
- 接口对象的动态值 (通过 type assertions 获得)
- 常数
- [literal](https://golang.org/ref/spec#Integer_literals) 值 (非 composite literal)
- package 级别的函数
- 方法 method (用作函数值)
- 中间值 (intermediate value):
    - 函数调用
    - 显式类型转换
    - 各种类型的操作 （除了指针引用 (pointer dereference) 操作 `*x`):
        - channel receive operations
        - sub-string operations
        - sub-slice operations
        - 加减乘除等运算符

规范中还有几处提到了 addressable:
- 调用一个 `receiver` 为指针类型的方法时，使用一个 `addressable` 的值将自动获取这个值的指针
- `++`、`--` 语句的操作对象必须是 `addressable` 或者是 `map` 的 index 操作
- 赋值语句 `=` 的左边对象必须是 `addressable`, 或者是 `map` 的 index 操作，或者是 `_`
- 上条同样使用 `for ... range` 语句

### 3.1 reflect.Value CanAddr

在我们使用 `reflect` 执行一些底层的操作的时候， 比如编写序列化库、编解码等业务的时候，经常会使用到 `reflect.Value` 的 `CanSet` 方法，用来动态的给对象赋值。 `CanSet` 比 `CanAddr` 只加了一个限制，就是 `struct` 类型的 `unexported` 的字段不能 `Set`，所以我们这节主要介绍 `CanAddr`

并不是任意的 `reflect.Value` 的 `CanAddr` 方法都返回 `true`, 根据它的 godoc, 我们可以知道：

```golang
CanAddr reports whether the value's address can be obtained with Addr.
Such values are called addressable. A value is addressable if it is an
element of a slice, an element of an addressable array, a field of an
addressable struct, or the result of dereferencing a pointer. If
CanAddr returns false, calling Addr will panic.
```

也就是只有下面的类型 `reflect.Value` 的 `CanAdd` 才是 `true`, 这样的值是 addressable:

- slice 的元素
- 可寻址数组的元素
- 可寻址 struct 的字段
- 指针引用的结果

与规范中规定的 `addressable`, `reflect.Value` 的 `addressable` 范围有所缩小， 比如对于栈上分配的变量， 随着方法的生命周期的结束， 栈上的对象也就被回收掉了，这个时候如果获取它们的地址，就会出现不一致的结果，甚至安全问题

所以如果你想通过 `reflect.Value` 对它的值进行更新，应该确保它的 `CanSet` 方法返回 `true`, 这样才能调用 `SetXXX` 进行设置

使用 `reflect.Value` 的时候有时会对 `func Indirect(v Value) Value` 和 `func (v Value) Elem() Value` 两个方法有些迷惑，有时候他们俩会返回同样的值，有时候又不会。

总结一下：
- 如果 `reflect.Value` 是一个指针， 那么 `v.Elem()` 等价于 `reflect.Indirect(v)`
- 如果不是指针:
    - 如果是 `interface`, 那么 `reflect.Indirect(v)` 返回同样的值，而 `v.Elem()` 返回接口的动态的值
    - 如果是其它值, `v.Elem()` 会 panic, 而 r`eflect.Indirect(v)` 返回原值

下面的代码列出一些 `reflect.Value` 是否可以 `addressable`, 你需要注意数组和 `struct` 字段的情况，也就是 `x7`、`x9`、`x14`、`x15` 的正确的处理方式

```golang
package main
import (
	"fmt"
	"reflect"
	"time"
)
func main() {
	checkCanAddr()
}
type S struct {
	X int
	Y string
	z int
}
func M() int {
	return 100
}
var x0 = 0
func checkCanAddr() {
	// 可寻址的情况
	v := reflect.ValueOf(x0)
	fmt.Printf("x0: %v \tcan be addressable and set: %t, %t\n", x0, v.CanAddr(), v.CanSet()) //false,false
	var x1 = 1
	v = reflect.Indirect(reflect.ValueOf(x1))
	fmt.Printf("x1: %v \tcan be addressable and set: %t, %t\n", x1, v.CanAddr(), v.CanSet()) //false,false
	var x2 = &x1
	v = reflect.Indirect(reflect.ValueOf(x2))
	fmt.Printf("x2: %v \tcan be addressable and set: %t, %t\n", x2, v.CanAddr(), v.CanSet()) //true,true
	var x3 = time.Now()
	v = reflect.Indirect(reflect.ValueOf(x3))
	fmt.Printf("x3: %v \tcan be addressable and set: %t, %t\n", x3, v.CanAddr(), v.CanSet()) //false,false
	var x4 = &x3
	v = reflect.Indirect(reflect.ValueOf(x4))
	fmt.Printf("x4: %v \tcan be addressable and set: %t, %t\n", x4, v.CanAddr(), v.CanSet()) // true,true
	var x5 = []int{1, 2, 3}
	v = reflect.ValueOf(x5)
	fmt.Printf("x5: %v \tcan be addressable and set: %t, %t\n", x5, v.CanAddr(), v.CanSet()) // false,false
	var x6 = []int{1, 2, 3}
	v = reflect.ValueOf(x6[0])
	fmt.Printf("x6: %v \tcan be addressable and set: %t, %t\n", x6[0], v.CanAddr(), v.CanSet()) //false,false
	var x7 = []int{1, 2, 3}
	v = reflect.ValueOf(x7).Index(0)
	fmt.Printf("x7: %v \tcan be addressable and set: %t, %t\n", x7[0], v.CanAddr(), v.CanSet()) //true,true
	v = reflect.ValueOf(&x7[1])
	fmt.Printf("x7.1: %v \tcan be addressable and set: %t, %t\n", x7[1], v.CanAddr(), v.CanSet()) //true,true
	var x8 = [3]int{1, 2, 3}
	v = reflect.ValueOf(x8[0])
	fmt.Printf("x8: %v \tcan be addressable and set: %t, %t\n", x8[0], v.CanAddr(), v.CanSet()) //false,false
	// https://groups.google.com/forum/#!topic/golang-nuts/RF9zsX82MWw
	var x9 = [3]int{1, 2, 3}
	v = reflect.Indirect(reflect.ValueOf(x9).Index(0))
	fmt.Printf("x9: %v \tcan be addressable and set: %t, %t\n", x9[0], v.CanAddr(), v.CanSet()) //false,false
	var x10 = [3]int{1, 2, 3}
	v = reflect.Indirect(reflect.ValueOf(&x10)).Index(0)
	fmt.Printf("x9: %v \tcan be addressable and set: %t, %t\n", x10[0], v.CanAddr(), v.CanSet()) //true,true
	var x11 = S{}
	v = reflect.ValueOf(x11)
	fmt.Printf("x11: %v \tcan be addressable and set: %t, %t\n", x11, v.CanAddr(), v.CanSet()) //false,false
	var x12 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x12))
	fmt.Printf("x12: %v \tcan be addressable and set: %t, %t\n", x12, v.CanAddr(), v.CanSet()) //true,true
	var x13 = S{}
	v = reflect.ValueOf(x13).FieldByName("X")
	fmt.Printf("x13: %v \tcan be addressable and set: %t, %t\n", x13, v.CanAddr(), v.CanSet()) //false,false
	var x14 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x14)).FieldByName("X")
	fmt.Printf("x14: %v \tcan be addressable and set: %t, %t\n", x14, v.CanAddr(), v.CanSet()) //true,true
	var x15 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x15)).FieldByName("z")
	fmt.Printf("x15: %v \tcan be addressable and set: %t, %t\n", x15, v.CanAddr(), v.CanSet()) //true,false
	v = reflect.Indirect(reflect.ValueOf(&S{}))
	fmt.Printf("x15.1: %v \tcan be addressable and set: %t, %t\n", &S{}, v.CanAddr(), v.CanSet()) //true,true
	var x16 = M
	v = reflect.ValueOf(x16)
	fmt.Printf("x16: %p \tcan be addressable and set: %t, %t\n", x16, v.CanAddr(), v.CanSet()) //false,false
	var x17 = M
	v = reflect.Indirect(reflect.ValueOf(&x17))
	fmt.Printf("x17: %p \tcan be addressable and set: %t, %t\n", x17, v.CanAddr(), v.CanSet()) //true,true
	var x18 interface{} = &x11
	v = reflect.ValueOf(x18)
	fmt.Printf("x18: %v \tcan be addressable and set: %t, %t\n", x18, v.CanAddr(), v.CanSet()) //false,false
	var x19 interface{} = &x11
	v = reflect.ValueOf(x19).Elem()
	fmt.Printf("x19: %v \tcan be addressable and set: %t, %t\n", x19, v.CanAddr(), v.CanSet()) //true,true
	var x20 = [...]int{1, 2, 3}
	v = reflect.ValueOf([...]int{1, 2, 3})
	fmt.Printf("x20: %v \tcan be addressable and set: %t, %t\n", x20, v.CanAddr(), v.CanSet()) //false,false
}
```
// Output
```
x0: 0   can be addressable and set: false, false
x1: 1   can be addressable and set: false, false
x2: 0xc0000180d8        can be addressable and set: true, true
x3: 2019-01-06 21:08:05.735797 +0800 CST m=+0.000439737         can be addressable and set: false, false
x4: 2019-01-06 21:08:05.735797 +0800 CST m=+0.000439737         can be addressable and set: true, true
x5: [1 2 3]     can be addressable and set: false, false
x6: 1   can be addressable and set: false, false
x7: 1   can be addressable and set: true, true
x7.1: 2         can be addressable and set: false, false
x8: 1   can be addressable and set: false, false
x9: 1   can be addressable and set: false, false
x9: 1   can be addressable and set: true, true
x11: {0  0}     can be addressable and set: false, false
x12: {0  0}     can be addressable and set: true, true
x13: {0  0}     can be addressable and set: false, false
x14: {0  0}     can be addressable and set: true, true
x15: {0  0}     can be addressable and set: true, false
x15.1: &{0  0}  can be addressable and set: true, true
x16: 0x10935f0  can be addressable and set: false, false
x17: 0x10935f0  can be addressable and set: true, true
x18: &{0  0}    can be addressable and set: false, false
x19: &{0  0}    can be addressable and set: true, true
x20: [1 2 3]    can be addressable and set: false, false
```
