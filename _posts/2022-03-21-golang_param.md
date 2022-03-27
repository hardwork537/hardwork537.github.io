---
layout:     post
title:      Go函数参数是值传递还是引用传递？ 
category:   go
tags:       [go]
description: 在 Go 语言中，函数的参数传递只有值传递，而且传递的实参都是原始数据的一份拷贝；不过这里的拷贝是浅拷贝，而不是深拷贝
---

## 结论

在 Go 语言中，函数的参数传递只有值传递，而且传递的实参都是原始数据的一份拷贝；不过这里的拷贝是浅拷贝，而不是深拷贝；如果拷贝的内容是值类型的，那么在函数中就无法修改原始数据；如果拷贝的内容是指针（或者可以理解为引用类型 map、chan 等），那么就可以在函数中修改原始数据。

## 函数传参为数组

```
package main

import "fmt"

const (
   num = 2
)

func main() {
   data := [num]string{"tony", "tom"}
   fmt.Printf("origin addr is %p\n", &data)
   modify(data)
   fmt.Printf("origin data is %v\n", data)

}

func modify(data [num]string) [num]string {
   fmt.Printf("modify addr is %p\n", &data)
   data[1] = "lucy"
   fmt.Printf("modify data is %v\n", data)
   return data
}

```

运行结果

```
➜  go go run test.go
origin addr is 0xc00006c020
modify addr is 0xc00006c040
modify data is [tony lucy]
origin data is [tony tom]

```

从结果可以看到，函数传参外面和函数里面的参数的地址不相同，分别为 0xc00006c020 和 0xc00006c040 ，在函数内修改参数值也不会影响函数外面的原始参数。

所以参数为数组时，传递方式是值传递；传给函数的参数值会被复制，函数在其内部使用的并不是参数值的原值，而是它的副本；

## 函数传参为切片

```
package main

import "fmt"

func main() {
   data := []string{"tony", "tom"}
   fmt.Printf("origin addr is %p\n", data) // 因为data 本身就是指针地址，所以不需要再用 & 取地址，如果再加上 & 则为指向指针的指针
   modify(data)
   fmt.Printf("origin data is %v\n", data)

}

func modify(data []string) []string {
   fmt.Printf("modify addr is %p\n", data)
   data[1] = "lucy"
   fmt.Printf("modify data is %v\n", data)
   return data
}

```

运行结果

```
➜  go go run test.go
origin addr is 0xc00006c020
modify addr is 0xc00006c020
modify data is [tony lucy]
origin data is [tony lucy]

```

从结果可以看到，函数传参外面和函数里面的参数的地址相同，都为 0xc00006c020，在函数内修改参数值会影响到原始参数值。

那切片是不是引用传递呢？

为什么切片和数组看起来差不多，作为参数传递时，结果却不一样呢？

先不急着下结论，我们先看看 slice 的底层结构

```
type slice struct {
    array unsafe.Pointer // 指向一个数组的指针
    len   int
    cap   int
}

```

可以看到，切片的底层结构是由三部分组成，第一个元素指针指向的数组才是真正的数据存储

那其实上面两个问题就可以一块回答了：

先说结论，对于切片来说，传递方式也是值传递。

在传递时，会创建一个 struct 副本，值和其中的三个元素一致；不过这里是浅拷贝，对于第一个指针元素，并不会拷贝其具体指向的数值，拷贝的是指针地址；另外切片和数组的地址其实是第一个元素的地址，所以 main 函数和 modify 函数中，切片的地址是一样的，都是指向第一个元素 unsafe.Pointer

对于引用类型，比如：切片、字典、通道，像上面那样复制它们的值，只会拷贝它们本身而已，并不会拷贝它们引用的底层数据。也就是说，这时只是浅表复制，而不是深层复制。以切片值为例，如此复制的时候，只是拷贝了它指向底层数组中某一个元素的指针，以及它的长度值和容量值，而它的底层数组并不会被拷贝。

## 函数传参为字典 map

```
package main

import "fmt"

func main() {
   data := map[string]int{"tony": 1, "tom": 2}
   fmt.Printf("origin addr is %p\n", data)
   modify(data)
   fmt.Printf("origin data is %v\n", data)

}

func modify(data map[string]int) map[string]int {
   fmt.Printf("modify addr is %p\n", data)
   data["tony"] = 4
   fmt.Printf("modify data is %v\n", data)
   return data
}

```

运行结果

```
➜  go go run test.go
origin addr is 0xc00007c180
modify addr is 0xc00007c180
modify data is map[tom:2 tony:4]
origin data is map[tom:2 tony:4]

```

运行结果基本和上面切片一致，其实底层逻辑也大体一致

在 Go 语言中，任何创建 map 的代码（不管是字面量还是 make 函数）最终调用的都是 runtime.makemap 函数。

用字面量或者 make 函数的方式创建 map，并转换成 makemap 函数的调用，这个转换是 Go 语言编译器自动帮我们做的。

从下面的代码可以看到，makemap 函数返回的是一个 *hmap 类型，也就是说返回的是一个指针，所以我们创建的 map 其实就是一个 *hmap。

src/runtime/map.go

```
// makemap implements Go map creation for make(map[k]v, hint).
func makemap(t *maptype, hint int, h *hmap) *hmap{
  //省略无关代码
}

//map底层结构
type hmap struct {
   // Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
   // Make sure this stays in sync with the compiler's definition.
   count     int // # live cells == size of map.  Must be first (used by len() builtin)
   flags     uint8
   B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
   noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
   hash0     uint32 // hash seed

   buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
   oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
   nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

   extra *mapextra // optional fields
}

```

这也是通过 map 类型的参数可以修改原始数据的原因，因为它本质上就是个指针。

## 函数传参为 channel

channel 也可以理解为引用类型，而它本质上也是个指针。

通过下面的源代码可以看到，所创建的 chan 其实是个 \*hchan，所以它在参数传递中也和 map 一样。

```
func makechan(t *chantype, size int64) *hchan {
    //省略无关代码
}

//channel底层结构
type hchan struct {
   qcount   uint           // total data in the queue
   dataqsiz uint           // size of the circular queue
   buf      unsafe.Pointer // points to an array of dataqsiz elements
   elemsize uint16
   closed   uint32
   elemtype *_type // element type
   sendx    uint   // send index
   recvx    uint   // receive index
   recvq    waitq  // list of recv waiters
   sendq    waitq  // list of send waiters

   // lock protects all fields in hchan, as well as several
   // fields in sudogs blocked on this channel.
   //
   // Do not change another G's status while holding this lock
   // (in particular, do not ready a G), as this can deadlock
   // with stack shrinking.
   lock mutex
}

```

严格来说，Go 语言没有引用类型，但是我们可以把 map、chan 称为引用类型，这样便于理解。除了 map、chan 之外，Go 语言中的函数、接口、slice 切片都可以称为引用类型。指针类型也可以理解为是一种引用类型。

## 函数传参为 struct

```
package main

import "fmt"

type Student struct {
   Name string
   Age  int
}

func main() {
   data := Student{"tony", 18}
   fmt.Printf("origin addr is %p\n", &data)
   modify(data)
   fmt.Printf("origin data is %v\n", data)

}

func modify(data Student) Student {
   fmt.Printf("modify addr is %p\n", &data)
   data.Age = 20
   fmt.Printf("modify data is %v\n", data)
   return data
}

```

运行结果

```
➜  go go run test.go
origin addr is 0xc00011c018
modify addr is 0xc00011c030
modify data is {tony 20}
origin data is {tony 18}

```

从结果可以看到，它们的内存地址不一样，在函数内修改参数值不会会影响到原始参数值。

所以当参数是 struct 时传参是值传递，传递原来数据的一份拷贝，而不是原来的数据本身。

除了 struct 外，还有浮点型、整型、字符串、布尔、数组，这些都是值类型。
