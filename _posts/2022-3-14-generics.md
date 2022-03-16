---
layout:     post
title:      go1.18泛型来了 
category:   golang
tags:       [generics]
description: 在 Go 1.18 中，引入了新的支持使用参数化类型的泛型代码，支持泛型一直是 Go 最常要求的功能，
---

## 安装支持泛型的 golang 版本

2022.3.16 发布的正式版本 1.18 已经默认支持泛型了，可以下载安装官方 golang1.18 版本

```
1. cd ~/Documents/software
2. wget -O ./go1.18.darwin-amd64.tar.gz "https://studygolang.com/dl/golang/go1.18.darwin-amd64.tar.gz"
3. mkdir ./go1.18 && tar -zxvf ./go1.18.darwin-amd64.tar.gz -C ./go1.18 --strip-components 1
4. export GOROOT=~/Documents/software/go1.18
```

## IDE 支持泛型

我用的 IDE 是 goland，之前的版本不支持泛型语法，所以泛型相关的代码地方会飘红

不过最新的 goland 版本已经支持泛型

我升级到 2021.3.3 版本就可以了

另外安装后记得把 goland 的 GOROOT 也设置成 ~/Documents/software/go1.18 ，这样才能真正支持泛型

## 泛型是什么？

就是允许我们基于一个还不确定的类型展开程序设计，我们围绕该类型编写代码逻辑时，它可能还未被定义出来。我们通常需要假设它具有某些属性，或者支持某些操作。所以泛型编程面向的是具有某些共同特性的一组类型

泛型和其他特性一样不是只有好处，为编程语言加入泛型会遇到需要权衡的两难问题。语言的设计者需要在编程效率、编译速度和运行速度三者进行权衡和选择，编程语言要选择牺牲一个而保留另外两个

```
package main

type MyType interface {
	~int16 | ~float64
}

func main() {
	add(int16(1), int16(2)) // 编译器生成 T = int16 的 Add
	add(float64(1), float64(2)) // 编译器生成 T = float64 的 add
}

func add[T MyType](data1, data2 T) T {
	return data1 + data2
}
```

## 为什么需要泛型

我们的代码中经常会用到一些本地缓存组件，有的支持过期时间，有的基于 LRU 算法。这些都是复用性极高的基础组件，经常以 mod 的形式单独提供。

这些组件在使用体验上都跟 map 差不多，都提供了 Set 和 Get 这类方法。为了支持任意类型，这些方法都使用了 interface{}类型的参数。经过简化之后，其核心结构就是如下这种形式：

```
type Cache map[string]interface{}

func (c Cache) Set(k string, v interface{}) {
  c[k] = v
}

func (c Cache) Get(k string) (v interface{}, ok bool) {
    v, ok = c[k]
    return
}
```

内部实际存储数据的就是个值类型为 interface{}的 map，interface{}是个万能容器，可以装载任意类型。使用 Set 方法的时候，一般不会觉得有什么不方便，因为从具体类型或接口类型到 interface{}的赋值不需要进行额外处理。但是 Get 方法使用起来就不那么完美了，我们需要通过类型断言把取出来的数据转换成预期的类型。比如我想从本地缓存 c 里面取出来一个 string，那就需要这样来写代码：

```
if v, ok := c.Get("key"); ok {
  if s, ok := v.(string); ok {
    // use s
  }
}
```

多了一步类型断言操作。如果你能够保证缓存里的值只有string这一种类型的话，也可以不使用comma ok风格的断言，这样更简单一点：



```
s := v.(string)
```

如果仅仅是多这一步操作的话，我们也就忍了，实际上可不是这么简单。在讲接口的时候我们曾经分析过，interface{}本质上是一对指针，用来装载值类型时会发生装箱，造成变量逃逸。比如我们用上面的Cache来缓存int64类型，map的bucket里存的是一个个的interface{}，而实际的int64会在堆上单独分配，interface{}的数据指针指向堆上的int64

而最高效的存储结构是直接把int64存储在map的bucket里

对比之下，基于interface{}的存储方式凭空多出来一次堆分配，并且又多占用了两倍的内存空间。从性能方面来考虑的话，这绝对是个十足的痛点了，我们期待泛型能够解决这个问题。


## 如何使用泛型


```
MyType[T1 constraint1 | constraint2, T2 constraint3...] ... 
```

泛型的语法非常简单, 就类似于上面这样, 其中:

- MyType可以是函数名, 结构体名, 类型名…
- T1, T2…是泛型名, 可以随便取
- constraint的意思是约束, 也是泛型中最重要的概念, 接下来会详解constraint
- 使用 &#124; 可以分隔多个constraint, T满足其中之一即可(如T1可以是constraint1和constraint2中的任何一个)


### Constraint(约束)是什么

约束的意思是限定范围, constraint的作用就是限定范围, 将T限定在某种范围内

而常用的范围, 我们自然会想到的有:

- any(interface{}, 任何类型都能接收, 多方便啊!)
- Interger(所有int, 多方便啊, int64 int32…一网打尽)
- Float(同上)
- comparable(所有可以比较的类型, 我们可以给所有可以比较的类型定制一些方法)
- …

这些约束, 不是被官方定义为内置类型, 就是被涵盖在了constraints包内!!!

下面是builtin.go的部分官方源码:


```
// any is an alias for interface{} and is equivalent to interface{} in all ways.
type any = interface{}

// comparable is an interface that is implemented by all comparable types
// (booleans, numbers, strings, pointers, channels, interfaces,
// arrays of comparable types, structs whose fields are all comparable types).
// The comparable interface may only be used as a type parameter constraint,
// not as the type of a variable.
type comparable comparable
```

下面是constraints.go的部分官方源码:


```
// Integer is a constraint that permits any integer type.
// If future releases of Go add new predeclared integer types,
// this constraint will be modified to include them.
type Integer interface {
	Signed | Unsigned
}

// Float is a constraint that permits any floating-point type.
// If future releases of Go add new predeclared floating-point types,
// this constraint will be modified to include them.
type Float interface {
	~float32 | ~float64
}
//......
```

不过在最新公布的1.18版本中，已经从Go规范库里移除，放到golang.org/x/exp我的项目下。

- golang.org/x下所有package的源码独立于Go源码的骨干分支，也不在Go的二进制安装包里。如果须要应用golang.org/x下的package，能够应用go get来装置。
- golang.org/x/exp下的所有package都属于试验性质或者被废除的package，不倡议应用。

反对泛型的Go 1.18 Beta 1版本公布以来，围绕着constraints包的争议很多。

次要是以下因素，导致Russ Cox决定从Go规范库中移除constraints包。

- constraints名字太长，代码写起来比拟繁琐。
- 大多数泛型的代码只用到了any和comparable这2个类型束缚。constaints包里只有constraints.Ordered应用比拟宽泛，其它很少用。所以齐全能够把Ordered设计成和any以及comparable一样，都作为Go的预申明标识符，不必独自弄一个constraints包。


### 自定义constraint(约束)

下面是constraints包中的官方源码:


```
type Signed interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

igned约束就是这样被写出来的, 其中需要我们get的点有如下几个:

1. 使用interface{}就可以自定义约束
2. 使用 | 就可以在该约束中包含不同的类型, 例如int, int8, int64均满足Signed约束
3. 你可能会有疑问, ~是什么??? int我认识, ~int我可不认识呀??? 没关系, 实际上~非常简单, 它的意思就是模糊匹配, 例如:
- type MyInt int64
- 此时 MyInt并不等同于int64类型(Go语言特性)
- 若我们使用int64来约束MyInt, 则Myint不满足该约束
- 若我们使用~int64来约束MyInt, 则Myint满足该约束(也就是说, ~int64只要求该类型的底层是int64, 也就是模糊匹配了)
- 官方为了鲁棒性, 自然把所有的类型前面都加上了~

我们自定义一个约束


```
type My_64_Bits_Long_Num interface {
	~int64 | ~float64
}
```

## 使用样例

### 自定义函数


```
package main

type MyType interface {
	~int16 | ~float64
}

func main() {
	add(int16(1), int16(2))
	add(float64(1), float64(2))
}

func add[T MyType](data1, data2 T) T {
	return data1 + data2
}
```

### 自定义结构体


```
package main

import (
	"fmt"
	"golang.org/x/exp/constraints"
)

type MyIntegerNode[T constraints.Integer] struct {
	Next *MyIntegerNode[T] //注意这里一定要加类型声明(和C++一样)
	//即使定义T为Integer类型，在使用时，同一个结构实体中，类型也只能为同一种
	//不能存在两个元素，一个为int16，一个为int64
	Data T
}

func main() {
	head := &MyIntegerNode[int64]{Next: nil, Data: 1}
	head.Next = &MyIntegerNode[int64]{Next: nil, Data: 2}

	for p := head; p != nil; p = p.Next {
		fmt.Printf("%d\n", p.Data)
	}
}
```

### 自定义类型


```
package main

import (
	"fmt"
)

type MyMap[V any] map[string]V

func main() {
	data1 := MyMap[any]{"name": "tony", "sex": "male"}
	printMap(data1)
	data2 := MyMap[any]{"1": 99, "2": 88.8}
	printMap(data2)
}

func printMap(data MyMap[any]) {
	for k, v := range data {
		fmt.Printf("key: %v, val: %v\n", k, v)
	}
}
```

### 泛型实现缓存

开头讲的用interface实现缓存，这里我们就可以换成泛型来实现


```
package main

import "fmt"

type cache[T any] map[string]T

func (c cache[T]) Set(k string, v T) {
	c[k] = v
}

func (c cache[T]) Get(k string) (v T) {
	v, _ = c[k]
	return
}

func main() {
	c := make(cache[int64])
	c.Set("nine", 9)
	fmt.Println("nine", c.Get("nine"))

	c1 := make(cache[string])
	c1.Set("name", "tony")
	c1.Set("sex", "male")

	fmt.Println("name:", c1.Get("name"))
	fmt.Println("sex:", c1.Get("sex"))
}
```

## 泛型能替代interface吗？

泛型并不会替代interface{}，其实两者的适用场景根本不同。泛型本质上是编译阶段的代码生成，而interface{}主要用来实现语言的动态特性，这个我们在讲接口的时候已经深入的分析过了。即使不考虑这些，还是针对上面的cache类型来讲，如果你希望在一个缓存对象里存放多种不同类型的值，你还是需要interface{}，只是直接这样写目前编译不通过：


你需要基于interface{}自定义一个类型就可以了：


```
type T interface{}
var c cache[T]
```

这样你就会得到一个cache[T]类型，本质上还是cache[interface{}]。

## 泛型实现原理

Keith H. Randall, MIT的博士，现在在Google/Go team做泛型方面的开发，提出了Go泛型实现的三个方案：

### 1.字典

在编译时生成一组实例化的字典，在实例话一个泛型函数的时候会使用字典进行蜡印(stencile)。

当为泛型函数生成代码的时候，会生成唯一的一块代码，并且会在参数列表中增加一个字典做参数，就像方法会把receiver当成一个参数传入。字典包含为类型参数实例化的类型信息。

字典在编译时生成，存放在只读的data section中。

当然字段可以当成第一个参数，或者最后一个参数，或者放入一个独占的寄存器。

当然这种方案还有依赖问题，比如字典递归的问题，更重要的是，它对性能可能有比较大的影响，比如一个实例化类型int, x=y可能通过寄存器复制就可以了，但是泛型必须通过memmove。


### 2.蜡印

同一个泛型函数，为每一个实例化的类型参数生成一套独立的代码

比如下面一个泛型方法:


```
func f[T1, T2 any](x int, y T1) T2 {
    ...
}
```

如果有两个不同的类型实例化的调用：


```
var a float64 = f[int, float64](7, 8.0)
var b struct{f int} = f[complex128, struct{f int}](3, 1+1i)
```

那么这个方案会生成两套代码：


```
func f1(x int, y int) float64 {
    ... identical bodies ...
}
func f2(x int, y complex128) struct{f int} {
    ... identical bodies ...
}
```

因为编译f时是不知道它的实例化类型的，只有在调用它时才知道它的实例化的类型，所以需要在调用时编译f。对于相同实例化类型的多个调用，同一个package下编译器可以识别出来是一样的，只生成一个代码就可以了，但是不同的package就不简单了，这些函数表标记为DUPOK,所以链接器会丢掉重复的函数实现。

这种策略需要更多的编译时间，因为需要编译泛型函数多次。因为对于同一个泛型函数，每种类型需要单独的一份编译的代码，如果类型非常多，编译的文件可能非常大，而且性能也比较差。

### 3. 混合方案

混合前面的两种方案。

对于实例类型的shape相同的情况，只生成一份代码，对于shape类型相同的类型，使用字典区分类型的不同行为。

这种方案介于前两者之间。

啥叫shape?

类型的shape是它对内存分配器/垃圾回收器呈现的方式，包括它的大小、所需的对齐方式、以及类型哪些部分包含指针。

每一个唯一的shape会产生一份代码，每份代码携带一个字典，包含了实例化类型的信息。

这种方案的问题是到底能带来多大的收益，它会变得有多慢，以及其它的一些问题。

从当前的反编译的代码看，当前Go采用的是第二种方案

比如上面的自定义函数查看对应的汇编代码


```
0x0000 00000 (fanxing.go:7)     TEXT    "".main(SB), ABIInternal, $32-0
        0x0000 00000 (fanxing.go:7)     CMPQ    SP, 16(R14)
        0x0004 00004 (fanxing.go:7)     PCDATA  $0, $-2
        0x0004 00004 (fanxing.go:7)     JLS     80
        0x0006 00006 (fanxing.go:7)     PCDATA  $0, $-1
        0x0006 00006 (fanxing.go:7)     SUBQ    $32, SP
        0x000a 00010 (fanxing.go:7)     MOVQ    BP, 24(SP)
        0x000f 00015 (fanxing.go:7)     LEAQ    24(SP), BP
        0x0014 00020 (fanxing.go:7)     FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (fanxing.go:7)     FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (fanxing.go:8)     LEAQ    ""..dict.add[int16](SB), AX
        0x001b 00027 (fanxing.go:8)     MOVL    $1, BX
        0x0020 00032 (fanxing.go:8)     MOVL    $2, CX
        0x0025 00037 (fanxing.go:8)     PCDATA  $1, $0
        0x0025 00037 (fanxing.go:8)     CALL    "".add[go.shape.int16_0](SB)
        0x002a 00042 (fanxing.go:9)     LEAQ    ""..dict.add[float64](SB), AX
        0x0031 00049 (fanxing.go:9)     MOVSD   $f64.3ff0000000000000(SB), X0
        0x0039 00057 (fanxing.go:9)     MOVSD   $f64.4000000000000000(SB), X1
        0x0041 00065 (fanxing.go:9)     CALL    "".add[go.shape.float64_0](SB)
        0x0046 00070 (fanxing.go:10)    MOVQ    24(SP), BP
        0x004b 00075 (fanxing.go:10)    ADDQ    $32, SP
        0x004f 00079 (fanxing.go:10)    RET
        0x0050 00080 (fanxing.go:10)    NOP
        0x0050 00080 (fanxing.go:7)     PCDATA  $1, $-1
        0x0050 00080 (fanxing.go:7)     PCDATA  $0, $-2
        0x0050 00080 (fanxing.go:7)     CALL    runtime.morestack_noctxt(SB)
        0x0055 00085 (fanxing.go:7)     PCDATA  $0, $-1
        0x0055 00085 (fanxing.go:7)     JMP     0
        0x0000 49 3b 66 10 76 4a 48 83 ec 20 48 89 6c 24 18 48  I;f.vJH.. H.l$.H
        0x0010 8d 6c 24 18 48 8d 05 00 00 00 00 bb 01 00 00 00  .l$.H...........
        0x0020 b9 02 00 00 00 e8 00 00 00 00 48 8d 05 00 00 00  ..........H.....
        0x0030 00 f2 0f 10 05 00 00 00 00 f2 0f 10 0d 00 00 00  ................
        0x0040 00 e8 00 00 00 00 48 8b 6c 24 18 48 83 c4 20 c3  ......H.l$.H.. .
        0x0050 e8 00 00 00 00 eb a9                             .......
        rel 23+4 t=14 ""..dict.add[int16]+0
        rel 38+4 t=7 "".add[go.shape.int16_0]+0
        rel 45+4 t=14 ""..dict.add[float64]+0
        rel 53+4 t=14 $f64.3ff0000000000000+0
        rel 61+4 t=14 $f64.4000000000000000+0
        rel 66+4 t=7 "".add[go.shape.float64_0]+0
        rel 81+4 t=7 runtime.morestack_noctxt+0
"".add[go.shape.int16_0] STEXT dupok nosplit size=65 args=0x10 locals=0x10 funcid=0x0 align=0x0
        0x0000 00000 (fanxing.go:12)    TEXT    "".add[go.shape.int16_0](SB), DUPOK|NOSPLIT|ABIInternal, $16-16
        0x0000 00000 (fanxing.go:12)    SUBQ    $16, SP
        0x0004 00004 (fanxing.go:12)    MOVQ    BP, 8(SP)
        0x0009 00009 (fanxing.go:12)    LEAQ    8(SP), BP
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $5, "".add[go.shape.int16_0].arginfo1(SB)
        0x000e 00014 (fanxing.go:12)    MOVQ    AX, ""..dict+24(SP)
        0x0013 00019 (fanxing.go:12)    MOVW    BX, "".data1+32(SP)
        0x0018 00024 (fanxing.go:12)    MOVW    CX, "".data2+34(SP)
        0x001d 00029 (fanxing.go:12)    MOVW    $0, "".~r0+6(SP)
        0x0024 00036 (fanxing.go:13)    MOVWLZX "".data2+34(SP), CX
        0x0029 00041 (fanxing.go:13)    MOVWLZX "".data1+32(SP), DX
        0x002e 00046 (fanxing.go:13)    LEAL    (DX)(CX*1), AX
        0x0031 00049 (fanxing.go:13)    MOVW    AX, "".~r0+6(SP)
        0x0036 00054 (fanxing.go:13)    MOVQ    8(SP), BP
        0x003b 00059 (fanxing.go:13)    ADDQ    $16, SP
        0x003f 00063 (fanxing.go:13)    NOP
        0x0040 00064 (fanxing.go:13)    RET
        0x0000 48 83 ec 10 48 89 6c 24 08 48 8d 6c 24 08 48 89  H...H.l$.H.l$.H.
        0x0010 44 24 18 66 89 5c 24 20 66 89 4c 24 22 66 c7 44  D$.f.\$ f.L$"f.D
        0x0020 24 06 00 00 0f b7 4c 24 22 0f b7 54 24 20 8d 04  $.....L$"..T$ ..
        0x0030 0a 66 89 44 24 06 48 8b 6c 24 08 48 83 c4 10 90  .f.D$.H.l$.H....
        0x0040 c3                                               .
"".add[go.shape.float64_0] STEXT dupok nosplit size=66 args=0x18 locals=0x10 funcid=0x0 align=0x0
        0x0000 00000 (fanxing.go:12)    TEXT    "".add[go.shape.float64_0](SB), DUPOK|NOSPLIT|ABIInternal, $16-24
        0x0000 00000 (fanxing.go:12)    SUBQ    $16, SP
        0x0004 00004 (fanxing.go:12)    MOVQ    BP, 8(SP)
        0x0009 00009 (fanxing.go:12)    LEAQ    8(SP), BP
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x000e 00014 (fanxing.go:12)    FUNCDATA        $5, "".add[go.shape.float64_0].arginfo1(SB)
        0x000e 00014 (fanxing.go:12)    MOVQ    AX, ""..dict+24(SP)
        0x0013 00019 (fanxing.go:12)    MOVSD   X0, "".data1+32(SP)
        0x0019 00025 (fanxing.go:12)    MOVSD   X1, "".data2+40(SP)
        0x001f 00031 (fanxing.go:12)    XORPS   X1, X1
        0x0022 00034 (fanxing.go:12)    MOVSD   X1, "".~r0(SP)
        0x0027 00039 (fanxing.go:13)    MOVSD   "".data1+32(SP), X0
        0x002d 00045 (fanxing.go:13)    ADDSD   "".data2+40(SP), X0
        0x0033 00051 (fanxing.go:13)    MOVSD   X0, "".~r0(SP)
        0x0038 00056 (fanxing.go:13)    MOVQ    8(SP), BP
        0x003d 00061 (fanxing.go:13)    ADDQ    $16, SP
        0x0041 00065 (fanxing.go:13)    RET
        0x0000 48 83 ec 10 48 89 6c 24 08 48 8d 6c 24 08 48 89  H...H.l$.H.l$.H.
        0x0010 44 24 18 f2 0f 11 44 24 20 f2 0f 11 4c 24 28 0f  D$....D$ ...L$(.
        0x0020 57 c9 f2 0f 11 0c 24 f2 0f 10 44 24 20 f2 0f 58  W.....$...D$ ..X
        0x0030 44 24 28 f2 0f 11 04 24 48 8b 6c 24 08 48 83 c4  D$(....$H.l$.H..
        0x0040 10 c3                                            ..
""..dict.add[int16] SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 type.int16+0
        rel 0+0 t=23 type.int16+0
""..dict.add[float64] SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 type.float64+0
        rel 0+0 t=23 type.float64+0
go.cuinfo.packagename. SDWARFCUINFO dupok size=0
        0x0000 6d 61 69 6e                                      main
""..inittask SNOPTRDATA size=24
        0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0010 00 00 00 00 00 00 00 00                          ........
runtime.nilinterequal·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 runtime.nilinterequal+0
runtime.memequal64·f SRODATA dupok size=8
        0x0000 00 00 00 00 00 00 00 00                          ........
        rel 0+8 t=1 runtime.memequal64+0
runtime.gcbits.01 SRODATA dupok size=1
        0x0000 01                                               .
type..namedata.*main.MyType. SRODATA dupok size=14
        0x0000 01 0c 2a 6d 61 69 6e 2e 4d 79 54 79 70 65        ..*main.MyType
type.*"".MyType SRODATA dupok size=56
        0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
        0x0010 a3 81 c9 8d 08 08 08 36 00 00 00 00 00 00 00 00  .......6........
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00                          ........
        rel 24+8 t=1 runtime.memequal64·f+0
        rel 32+8 t=1 runtime.gcbits.01+0
        rel 40+4 t=5 type..namedata.*main.MyType.+0
        rel 48+8 t=1 type."".MyType+0
runtime.gcbits.02 SRODATA dupok size=1
        0x0000 02                                               .
type."".MyType SRODATA dupok size=96
        0x0000 10 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
        0x0010 1f a5 c9 45 07 08 08 14 00 00 00 00 00 00 00 00  ...E............
        0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0050 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
        rel 24+8 t=1 runtime.nilinterequal·f+0
        rel 32+8 t=1 runtime.gcbits.02+0
        rel 40+4 t=5 type..namedata.*main.MyType.+0
        rel 44+4 t=5 type.*"".MyType+0
        rel 48+8 t=1 type..importpath."".+0
        rel 56+8 t=1 type."".MyType+96
        rel 80+4 t=5 type..importpath."".+0
gclocals·33cdeccccebe80329f1fdbee7f5874cb SRODATA dupok size=8
        0x0000 01 00 00 00 00 00 00 00                          ........
"".add[go.shape.int16_0].arginfo1 SRODATA static dupok size=5
        0x0000 08 02 0a 02 ff                                   .....
"".add[go.shape.float64_0].arginfo1 SRODATA static dupok size=5
        0x0000 08 08 10 08 ff                                   .....
$f64.3ff0000000000000 SRODATA size=8
        0x0000 00 00 00 00 00 00 f0 3f                          .......?
$f64.4000000000000000 SRODATA size=8
        0x0000 00 00 00 00 00 00 00 40                          .......@

```


## 参考

[【Golang】泛型要来了吗？](https://zhuanlan.zhihu.com/p/411595941)

[Go1.18版本泛型详解](https://blog.csdn.net/qq_52582768/article/details/121984157)

















