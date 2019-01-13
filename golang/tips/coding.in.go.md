---
title: 如何写好 golang 代码的一些 tips
date: 2018-07-02T21:12:55+08:00
categories:
    - Golang
tags: 
    - Golang
    - tips
---

# 如何写好 golang 代码的一些 tips

## 1. 首先需要知道的概念

- 不要像其他语言一样写 Go, Go 不是 Java/Python/Ruby/PHP，Go 的程序思维很古典也很超前
- 保持简洁，避免过度工程化
- 学会阅读源码
- interface 很重要，很多问题都可以用 interface 解决
- 保持函数精炼，变量名长度合适 (在不丢失意义的情况下，尽量短小)
- 用更小的单元实现代码功能，然后 ** 组合 **
- 慎重使用全局变量 (尤其是跨 package 使用，变量作用域上容易踩坑，可以参考 [vs6IeAu5U5n](https://play.golang.org/p/vs6IeAu5U5n))

## 2. 函数声明的 tips

- 函数声明首先是函数名字要具有自解释性，需要能够解释清楚大致提供的功能，而不是模糊的 common/util。
- 其次具体的功能解释，这个要说到代码注释了，这里就不赘述了。
- 除了函数声明外，还有函数的形参定义。这里以一个例子来说一下扩展性好的函数的参数应该如何定义 (见 2.1)。

### 2.1 普通函数
假设我们需要一个简单的 server，我们可以像下面这样定义，addr 表示 server 启动在哪个端口上。

```
func NewServer(addr string)
```

第一期的需求很简单，就上面这些足够满足了。项目上线跑了一段时间发现，由于连接没有设置超时，很多连接一直得不到释放（异常情况），严重影响服务器性能。

好，那第二期我们加个 timeout。

```
func NewServer(addr string, timeout time.Duration)
```

这个时候尴尬的情况出现了，调用你代码的所有人都需要改动代码。而且这只是一个改动，之后如果要支持 tls，那么又得改动一次。

### 2.2 不定参数
解决上面的窘境的一种方法是使用不定参数。

比如我要实现一个整数加法。

```
func Add(list ...int) int {
    sum := 0
    for _, x := range list {
        sum += x
    }
    return sum
}

func main() {
    fmt.Println(Add(1,2,3)) //6
}
```

上面是所有的变参都是同一种类型，如果是不同的类型可以使用 interface，使用反射来判断其类型。

```
func Varargs(list ...interface{}) {
    for _, x := range list {
        if reflect.ValueOf(x).Kind() == int {
            //
        }
    }
}
```

但是如果是我们自己定义的函数的话，类型通常是知道的，也就不需要上面那么麻烦地再去判断一次，可以直接进行类型转换。

```
func Varargs(list ...interface{}) {
    // 通过 interface.(type) 将 interface 类型转换成 type 类型
    fmt.Println(list[0].(int))
    fmt.Println(list[1].(string))
}
```

但是这么做比较危险，使用的时候必须严格按照说明进行传参，任何一种类型不正确，程序将 panic。还有一个问题就是不定参数不能为空，或者说传入的实参必须是形参的一个严格前缀。

### 2.3 封装成 struct
相比于上面两种方法更好一点的是把所有参数封装成 struct，这样函数声明看起来很简单。

```
type Param struct {
    x   int
    y   string
    ...
}

func Varargs(p *Param) {}
```

封装成 struct 的方式应该是一种对参数比较好的组织形式，之后函数不管怎么扩张，只需要增加 struct 成员就好，而不需要改变函数声明了。而 struct 的坏处在什么地方呢？比如上面的 Param.x 是 int 型，如果我们不设置 x，也就是下面这样传参。

```
p := &Param{
    y:  "hello",
}

Varargs(p)
```

这个时候 `Varargs` 看到的 `Param.x` 的 0。

你让 `Varargs` 怎么想?

- 用户没有设置 x（忘记设置？想使用默认值？）
- 用户把 x 设置成 0

这真的有点尴尬。但是这个问题还是有解决方案的:

- 1. 避开默认值，int 型不使用 0，string 类型不使用 `""`
- 2. 使用指针，用户没有设置的时候 x==nil，设置的时候对 x 解引用 (*x) 取得值。

这两种方式不管怎么来看，都是十分的反人类，一点也不 simple。

### 2.4 option
option 的方式的最早是由 Rob Pike 提出，Rob Pike 就不做介绍了，感兴趣的可以看他的 wiki 连接。我们把 option 参数封装成一个函数传给我们的目标函数，所有相关的工作由函数来做。举个栗子，我们现在要写个 Server，timeout 和 tls 都是可选项，那么可以像下面这么来写 (所有 error handle 都省去)。

```
func NewServer(addr string, options ...func(*Server)) (*Server, error) {
    srv := &Server{
        Addr:   addr,
    }

    for _, option := range options {
        option(srv)
    }

    return srv
}

func timeout(d time.Duration) func(*Server) {
    return func(srv *Server) {
        srv.timeout = d
    }
}

func tls(c *config) func(*Server) {
    return func(srv *Server) {
        Tls := loadConfig(c)
        srv.tls = Tls
    }
}

// 使用
src, err = NewServer("localhost:8080", timeout(1), tls(path/to/cert))
```

这么写的好处一目了然，横向扩展起来特别方便，而且解决上面的提到的基本所有的问题。

## 3. 函数实现
正常单一功能的函数实现没有什么好说的。如果需要根据不同的条件来执行不同的行为的话，这个应该怎么做的？

举个例子，我现在在公司做一个精细化运营的项目，用户看到的内容时候有一些规则，比如每人兴趣以及职业等。这些规则肯定不会一成不变，也许第一期是 2 个规则，第二期就变成 4 个规则了。正常可能会像下面这么写。

```
func ruleVerify() {
    if cond1 {
        //
    } else if cond2 {
        //
    }
    ...
}
```

或者用 switch-case。虽然很多人说 switch-case 写起来要比 if-else 更好看或者高端一点，其实我并不这么觉得。if-else 和 switch-case 本质上并没有什么区别，扩展的时候如果需要多加一个条件分支，这两种方法改动起来都比较丑。

下面说说我的解决方案

### 3.1 类工厂模式
熟悉设计模式的肯定对工厂模式肯定不会陌生。工厂模式的意思是通过参数来决定生成什么样的对象实例。

我这里并不是说直接使用工厂模式而是使用工厂模式这种思想来编程。

举个典型的例子，webserver 的 router 实现方式：根据不同的路由（/foo，/bar）对应到不同的 handler。光这么说，可能很多人还是不明白这种方式的扩展性好在什么地方。

下面从 0 到 1 来感受一下。

首先根据不同的条件对应不同的 handler，这个最简单的是使用 Map 来实现，没有问题，但是 map 里面存什么呢？如果我要增加一个条件以及对应的处理函数的时候怎么做呢？

```
// 存放 <cond, handler> 对应关系
var mux map[string]func(option *Option) error

// 注册 handler
func register(key string, f func(option *Option) error) {
    if mux == nil {
        mux = make(map[string]func(option *Option) error)
    }
    if _, exist := mux[key]; exist {
        return errors.New("handler exist")
    }
    mux[key] = f
}

//factory
func factory(option *Option) error {
    return mux[option.Key](option)
}
```

代码主要分三个部分：

1. mux 用来存放 cond 和 handler 的对应关系
2. register 用来注册新的 handler
3. 提供给外部的代码入口。

下面到了最核心的问题了，如果某一天我们现在要新加一个用户精细化规则。代码上的改动只需要实现一个新增规则的实现函数，同时调用一下 register 即可

## 4. 组合实现功能
Go 语言不是面向对象语言，实际上只有 struct 而没有类或对象，使用 ** 组合 ** 来实现功能

GoF 对组合模式的定义是，将 ** 对象组合成树形结构以表示 “部分整体” 的层次结构，组合模式使得用户对单个对象和组合对象的使用具有一致性 **

组合模式的例子大家都见得很多了，比如

- 文件系统（文件 / 文件夹
- GUI 窗口（Frame/Control) 菜单（菜单 / 菜单项）

等等，我这里也举个菜单的例子，不过不是操作系统里的菜单，是真正的菜单，KFC 的……

姑且把 KFC 里的食物认为是 ** 菜单项 (MenuItem)**，一份套餐是 ** 菜单 (Menu)**。

菜单和菜单项有一些公有属性：名字、描述、价格、都能被购买等，所以正如 GoF 所说，我们需要一致性地使用它们。它们的层次结构体现在一个菜单里会包含多个菜单项或菜单，其价格是所有子项的和。

这个例子其实不是很恰当，不能很好的体现菜单包含菜单的情况，所以我多定义了一个 **“超值午餐”** 菜单，其中包含若干个套餐。

用代码归纳总结一下，最终我们的调用代码是这样的：

```
func main() {
	menu1 := NewMenu("培根鸡腿燕麦堡套餐", "供应时间：09:15--22:44")
	menu1.Add(NewMenuItem("主食", "培根鸡腿燕麦堡 1 个", 11.5))
	menu1.Add(NewMenuItem("小吃", "玉米沙拉 1 份", 5.0))
	menu1.Add(NewMenuItem("饮料", "九珍果汁饮料 1 杯", 6.5))

	menu2 := NewMenu("奥尔良烤鸡腿饭套餐", "供应时间：09:15--22:44")
	menu2.Add(NewMenuItem("主食", "新奥尔良烤鸡腿饭 1 份", 15.0))
	menu2.Add(NewMenuItem("小吃", "新奥尔良烤翅 2 块", 11.0))
	menu2.Add(NewMenuItem("饮料", "芙蓉荟蔬汤 1 份", 4.5))

	all := NewMenu("超值午餐", "周一至周五有售")
	all.Add(menu1)
	all.Add(menu2)

	all.Print()
}
```

得到的输出如下：

```
超值午餐, 周一至周五有售, ￥53.50
------------------------
培根鸡腿燕麦堡套餐, 供应时间：09:15--22:44, ￥23.00
------------------------
  主食, ￥11.50
    -- 培根鸡腿燕麦堡 1 个
  小吃, ￥5.00
    -- 玉米沙拉 1 份
  饮料, ￥6.50
    -- 九珍果汁饮料 1 杯

奥尔良烤鸡腿饭套餐, 供应时间：09:15--22:44, ￥30.50
------------------------
  主食, ￥15.00
    -- 新奥尔良烤鸡腿饭 1 份
  小吃, ￥11.00
    -- 新奥尔良烤翅 2 块
  饮料, ￥4.50
    -- 芙蓉荟蔬汤 1 份
```

### 4.1 面向对象实现
再次说明一下：虽然 Go 语言不是面向对象语言，但是我就是想用面向对象思维实现一遍

按照惯例，先使用经典的面向对象来分析。首先我们需要定义菜单和菜单项的抽象基类，这样使用者就可以只依赖于接口了，于是实现使用上的一致性。

Go 语言中没有继承，所以我们把抽象基类定义为接口，后面会由菜单和菜单项实现具体功能：

```
type MenuComponent interface {
	Name() string
	Description() string
	Price() float32
	Print()

	Add(MenuComponent)
	Remove(int)
	Child(int) MenuComponent
}
```

菜单项的实现：

```
type MenuItem struct {
	name        string
	description string
	price       float32
}

func NewMenuItem(name, description string, price float32) MenuComponent {
	return &MenuItem{
		name:        name,
		description: description,
		price:       price,
	}
}

func (m *MenuItem) Name() string {
	return m.name
}

func (m *MenuItem) Description() string {
	return m.description
}

func (m *MenuItem) Price() float32 {
	return m.price
}

func (m *MenuItem) Print() {
	fmt.Printf("%s, ￥%.2f\n", m.name, m.price)
	fmt.Printf("-- %s\n", m.description)
}

func (m *MenuItem) Add(MenuComponent) {
	panic("not implement")
}

func (m *MenuItem) Remove(int) {
	panic("not implement")
}

func (m *MenuItem) Child(int) MenuComponent {
	panic("not implement")
}
```

有两点请留意一下。

1. NewMenuItem() 创建的是 MenuItem，但返回的是抽象的接口 MenuComponent。（面向对象中的多态）
2. 因为 MenuItem 是叶节点，无法提供 Add() Remove() Child() 这三个方法的实现，所以若被调用会 panic。

下面是菜单的实现：

```
type Menu struct {
	name        string
	description string
	children    []MenuComponent
}

func NewMenu(name, description string) MenuComponent {
	return &Menu{
		name:        name,
		description: description,
	}
}

func (m *Menu) Name() string {
	return m.name
}

func (m *Menu) Description() string {
	return m.description
}

func (m *Menu) Price() (price float32) {
	for _, v := range m.children {
		price += v.Price()
	}
	return
}

func (m *Menu) Print() {
	fmt.Printf("%s, %s, ￥%.2f\n", m.name, m.description, m.Price())
	fmt.Println("------------------------")
	for _, v := range m.children {
		v.Print()
	}
	fmt.Println()
}

func (m *Menu) Add(c MenuComponent) {
	m.children = append(m.children, c)
}

func (m *Menu) Remove(idx int) {
	m.children = append(m.children[:idx], m.children[idx+1:]...)
}

func (m *Menu) Child(idx int) MenuComponent {
	return m.children[idx]
}
```

其中 Price() 统计所有子项的 Price 后加和，Print() 输出自身的信息后依次输出所有子项的信息。另注意 Remove() 的实现（从 slice 中删除一项）。

好，现在针对这份实现思考下面 3 个问题。

1. MenuItem 和 Menu 中都有 name、description 这两个属性和方法，重复写两遍明显冗余。如果使用其它任何面向对象语言，这两个属性和方法都应该移到基类中实现。可是 Go 没有继承，这可真是坑爹。
2. 这里我们真正实现了 ** 用户一致性访问 ** 了吗？显然没有，当使用者拿到一个 MenuComponent 后，依然要知道其类型后才能正确使用，假如不加判断在 MenuItem 使用 Add() 等未实现的方法就会产生 panic。类似地，我们大可以把文件夹 / 文件都抽象成 “文件系统节点”，可以读取名字，可以计算占用空间，但是一旦我们想往 “文件系统节点” 中添加子节点时，还是必须得判断它到底是不是文件夹
3. 接着第 2 条继续思考：产生某种一致性访问现象的本质原因是什么？一种观点：Menu 和 MenuItem 某种本质上是（is-a）同一个事物（MenuComponent），所以可以对它们一致性访问；另一种观点：Menu 和 MenuItem 是两个不同的事物，只是恰巧有一些相同的属性，所以可以对它们一致性访问

### 4.2 用组合代替继承
前面说到 Go 语言没有继承，本来属于基类的 name 和 description 不能放到基类中实现。其实只要转换一下思路，这个问题是很容易用组合解决的。如果我们认为 Menu 和 MenuItem 本质上是两个不同的事物，只是恰巧有（has-a）一些相同的属性，那么将相同的属性抽离出来，再分别组合进两者，问题就迎刃而解了。

先看抽离出来的属性：

```
type MenuDesc struct {
	name        string
	description string
}

func (m *MenuDesc) Name() string {
	return m.name
}

func (m *MenuDesc) Description() string {
	return m.description
}
```

改写 MenuItem：

```
type MenuItem struct {
	MenuDesc
	price float32
}

func NewMenuItem(name, description string, price float32) MenuComponent {
	return &MenuItem{
		MenuDesc: MenuDesc{
			name:        name,
			description: description,
		},
		price: price,
	}
}

// ... 方法略 ...
```

改写 Menu:

```
type Menu struct {
	MenuDesc
	children []MenuComponent
}

func NewMenu(name, description string) MenuComponent {
	return &Menu{
		MenuDesc: MenuDesc{
			name:        name,
			description: description,
		},
	}
}

// ... 方法略 ...
```

感受一下新的代码，是不是体会到了 **Go 语言中善用组合有助于表达数据结构的意图 **

特别是当一个比较复杂的对象同时处理几方面的事情时，将对象拆成独立的几个部分再组合到一起，会非常清晰优雅。例如上面的 MenuItem 就是描述 + 价格，Menu 就是描述 + 子菜单。

其实对于 Menu，更好的做法是把 children 和 Add() Remove() Child() 也提取封装后再进行组合，这样 Menu 的功能一目了然

```
type MenuGroup struct {
	children []MenuComponent
}

func (m *Menu) Add(c MenuComponent) {
	m.children = append(m.children, c)
}

func (m *Menu) Remove(idx int) {
	m.children = append(m.children[:idx], m.children[idx+1:]...)
}

func (m *Menu) Child(idx int) MenuComponent {
	return m.children[idx]
}

type Menu struct {
	MenuDesc
	MenuGroup
}

func NewMenu(name, description string) MenuComponent {
	return &Menu{
		MenuDesc: MenuDesc{
			name:        name,
			description: description,
		},
	}
}
```

### 4.3 Think in Go
下面让我们用真正 Go 的方式来实现 KFC 菜单。首先请默念三遍：没有继承，没有继承，没有继承；没有基类，没有基类，没有基类；接口只是函数签名的集合，接口只是函数签名的集合，接口只是函数签名的集合；struct 不依赖于接口，struct 不依赖于接口，struct 不依赖于接口。

好了，与之前不同，现在我们不是先定义接口再具体实现，因为 struct 不依赖于接口，所以我们直接实现具体功能。先是 MenuDesc 和 MenuItem，注意现在 NewMenuItem 的返回值类型是 * MenuItem。

```
type MenuDesc struct {
	name        string
	description string
}

func (m *MenuDesc) Name() string {
	return m.name
}

func (m *MenuDesc) Description() string {
	return m.description
}

type MenuItem struct {
	MenuDesc
	price float32
}

func NewMenuItem(name, description string, price float32) *MenuItem {
	return &MenuItem{
		MenuDesc: MenuDesc{
			name:        name,
			description: description,
		},
		price: price,
	}
}

func (m *MenuItem) Price() float32 {
	return m.price
}

func (m *MenuItem) Print() {
	fmt.Printf("%s, ￥%.2f\n", m.name, m.price)
	fmt.Printf("-- %s\n", m.description)
}
```

接下来是 MenuGroup。我们知道 MenuGroup 是菜单 / 菜单项的集合，其 children 的类型是不确定的，于是我们知道这里需要定义一个接口。又因为 MenuGroup 的逻辑是对 children 进行增、删、读操作，对 children 的属性没有任何约束和要求，所以我们这里暂时把接口定义为空接口 interface{}。

```
type MenuComponent interface {
}

type MenuGroup struct {
	children []MenuComponent
}

func (m *Menu) Add(c MenuComponent) {
	m.children = append(m.children, c)
}

func (m *Menu) Remove(idx int) {
	m.children = append(m.children[:idx], m.children[idx+1:]...)
}

func (m *Menu) Child(idx int) MenuComponent {
	return m.children[idx]
}
```

最后是 Menu 的实现：

```
type Menu struct {
	MenuDesc
	MenuGroup
}

func NewMenu(name, description string) *Menu {
	return &Menu{
		MenuDesc: MenuDesc{
			name:        name,
			description: description,
		},
	}
}

func (m *Menu) Price() (price float32) {
	for _, v := range m.children {
		price += v.Price()
	}
	return
}

func (m *Menu) Print() {
	fmt.Printf("%s, %s, ￥%.2f\n", m.name, m.description, m.Price())
	fmt.Println("------------------------")
	for _, v := range m.children {
		v.Print()
	}
	fmt.Println()
}
```

在实现 Menu 的过程中，我们发现 Menu 对其 children 实际上有两个约束：需要有 Price() 方法和 Print() 方法。于是对 MenuComponent 进行修改：

```
type MenuComponent interface {
	Price() float32
	Print()
}
```

最后观察 MenuItem 和 Menu，它们都符合 MenuComponent 的约束，所以二者都可以成为 Menu 的 children，组合模式大功告成！

### 4.4 组合与面向对象实现对比

前后两份代码差异其实很小：

1. 第二份实现的接口简单一些，只有两个函数。
2. New 函数返回值的类型不一样

从思路上看，差异很大却也有些微妙：

1. 第一份实现中接口是模板，是 struct 的蓝图，其属性来源于事先对系统组件的综合分析归纳；第二份实现中接口是一份约束声明，其属性来源于使用者对被使用者的要求
2. 第一份实现认为 children 中的 MenuComponent 是一种具体对象，这个对象具有一系列方法可以调用，只是其方法的功能会由于子类覆盖而表现不同；第二份实现则认为 children 中的 MenuComponent 可以是任意无关的对象，唯一的要求是他们 “恰巧” 实现了接口所指定的约束条件

注意第一份实现中，MenuComponent 中有 Add()、Remove()、Child() 三个方法，但却不一定是可用的，能不能使用由具体对象的类型决定；第二份实现中则不存在这些不安全的方法，因为 New 函数返回的是具体类型，所以可以调用的方法都是安全的。

另外，从 Menu 中取出某个 child，其可用方法只有 Price() 和 Print()，一样可以完全安全的调用。如果想在 MenuComponent 是 Menu 的情况下往其中添加子项呢？很简单

```
if m, ok := all.Child(1).(*Menu); ok {
	m.Add(NewMenuItem("玩具", "Hello Kitty", 5.0))
}
```

清晰明了，如果某 child 是一个 Menu，那么我们可以对其进行 Add() 操作。

更进一步，这里我们对类型的要求其实并没有那么强，并不需要它一定要是 Menu，只是需要其提供组合 MenuComponent 的功能，所以可以提炼出这样一个接口

```
type Group interface {
	Add(c MenuComponent)
	Remove(idx int)
	Child(idx int) MenuComponent
}
```

前面的添加子项的代码改成这样：

```
if m, ok := all.Child(1).(Group); ok {
	m.Add(NewMenuItem("玩具", "Hello Kitty", 5.0))
}
```

再考虑一下 “购买” 这个操作，面向对象的实现中，购买的类型是 MenuComponent，所以购买操作同时可以应用于 Menu 和 MenuItem。如果用 Go 语言的思维方式来考察，可购买对象的唯一要求是有 Price()，所以购买操作的参数是这样的接口

```
type Product interface {
    Price() float32
}
```

于是购买操作不仅可应用于 Menu 和 MenuItem，还可用于任何提供了价格的对象。我们可以任意添加产品，不论是玩具还是会员卡或者优惠券，只要有 Price() 方法就可以被购买

总结:

在组合模式中，一致性访问是个伪需求。一致性访问不是我们在设计时需要去满足的需求，而是当不同实体具有相同属性时自然产生的效果。上面的例子中，我们创建的是 menu 和 MenuItem 两种不同的类型，但由于它们具有相同属性，我们能以相同的方式取价格，取描述，加入 menu 成为子项

Go 语言中的多态不体现在对象创建阶段，而体现在对象使用阶段，合理使用 “小接口” 能显著减少系统耦合度。

上面提到的三份代码链接:

- [面向对象实现](https://play.golang.org/p/KuH2Vu7f9k)
- [使用组合](https://play.golang.org/p/2DzGhVYseY)
- [Go 语言的思维方式](https://play.golang.org/p/TGjI3CDHD4)

## 参考相关文章

- [functional-options-for-friendly-apis](http://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)
- [referential-functions](https://commandcenter.blogspot.com.au/2014/01/self-referential-functions-and-design.html)
- [设计模式实践：组合](https://juejin.im/entry/5ab8a615518825558a069daf)


