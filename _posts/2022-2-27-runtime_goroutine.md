---
layout:     post
title:      golang runtime goroutine 
category:   golang
tags:       [runtime]
description: go的调度顺序
---

## goroutine执行顺序

概括来说，每个P在每次循环调度可执行的goroutine时，基本都按照这个顺序去查找goroutine的：

1. 先看当前P的 runnext ，不为空，则返回 runnext 上的goroutine；为空则执行2
2. 查看当前P的本地队列 local queue，不为空则依据该队列的指针，返回第一个待执行的goroutine；队列为空，则执行3
3. 查看全局队列 global queue，不为空则获取一个返回执行；如果实在找不到，则执行4
4. 从其他P偷一个待运行状态的 goroutine进行执行


我们简要看下P的结构：


```
type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	pcache      pageCache
	
	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	
	runnext guintptr
	
	...
}
```


其中

- runnext 是只能指向一个goroutine的特殊队列，数据类型和 runq 的具体元素类型一致，它的优先级最高！

- runq 是一个大小为 256 的数组，实际上用 head 和 tail 指针把它当成一个环形数组在使用



## goroutine生成顺序


概括来说，每个goroutine生成后，在进入可运行队列时按这样的顺序：

1. 如果 runnext 为空，那么 goroutine 就会顺利地放入 runnext。接下来，它会以最高优先级得到运行；如果 runnext 不为空时，则执行2
2. 先把 runnext 上的 old goroutine 踢走，再把 new goroutine 放上来。具体踢到哪里呢？接下来步骤3
3. 上面P中的 runq 队列如果不满(256个)，则将 runnext 放入 runq；否则，执行步骤4
4. 将 当前 P 的runq 队列中的一半 goroutine和 old goroutine 一起打包丢到 global queue 里


相关代码：

```runtime/proc.go```


```
// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(_p_ *p, gp *g, next bool)   {
    if randomizeScheduler && next && fastrand() % 2 == 0  {
        next = false
    }
 
    if next  {
        //把gp放在_p_.runnext成员里，
        //runnext成员中的goroutine会被优先调度起来运行
    retryNext:
        oldnext := _p_.runnext
        if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp)))  {
             //有其它线程在操作runnext成员，需要重试
            goto retryNext
        }
        if oldnext == 0  { //原本runnext为nil，所以没任何事情可做了，直接返回
            return
        }
        // Kick the old runnext out to the regular run queue.
        gp = oldnext.ptr() //原本存放在runnext的gp需要放入runq的尾部
    }
 
retry:
    //可能有其它线程正在并发修改runqhead成员，所以需要跟其它线程同步
    h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
    t := _p_.runqtail
    if t - h < uint32(len(_p_.runq))  { //判断队列是否满了
        //队列还没有满，可以放入
        _p_.runq[t % uint32(len(_p_.runq))].set(gp)
        
        // store-release, makes it available for consumption
        //虽然没有其它线程并发修改这个runqtail，但其它线程会并发读取该值以及p的runq成员
        //这里使用StoreRel是为了：
        //1，原子写入runqtail
        //2，防止编译器和CPU乱序，保证上一行代码对runq的修改发生在修改runqtail之前
        //3，可见行屏障，保证当前线程对运行队列的修改对其它线程立马可见
        atomic.StoreRel(&_p_.runqtail, t + 1)
        return
    }
    //p的本地运行队列已满，需要放入全局运行队列
    if runqputslow(_p_, gp, h, t) {
        return
    }
    // the queue is not full, now the put above must succeed
    goto retry
}
```


runqput函数流程很清晰，它首先尝试把gp放入_p_的本地运行队列，如果本地队列满了，则通过runqputslow函数把gp放入全局运行队列。



```
// Put g and a batch of work from local runnable queue on global queue.
// Executed only by the owner P.
func runqputslow(_p_ *p, gp *g, h, t uint32) bool  {
    var batch [len(_p_.runq) / 2 + 1]*g  //gp加上_p_本地队列的一半
 
    // First, grab a batch from local queue.
    n := t - h
    n = n / 2
    if n != uint32(len(_p_.runq) / 2)  {
        throw("runqputslow: queue is not full")
    }
    for i := uint32(0); i < n; i++ { //取出p本地队列的一半
        batch[i] = _p_.runq[(h+i) % uint32(len(_p_.runq))].ptr()
    }
    if !atomic.CasRel(&_p_.runqhead, h, h + n)  { // cas-release, commits consume
        //如果cas操作失败，说明已经有其它工作线程从_p_的本地运行队列偷走了一些goroutine，所以直接返回
        return false
    }
    batch[n] = gp
 
    if randomizeScheduler {
        for i := uint32(1); i <= n; i++ {
            j := fastrandn(i + 1)
            batch[i], batch[j] = batch[j], batch[i]
        }
    }
 
    // Link the goroutines.
    //全局运行队列是一个链表，这里首先把所有需要放入全局运行队列的g链接起来，
    //减少后面对全局链表的锁住时间，从而降低锁冲突
    for i := uint32(0); i < n; i++  {
        batch[i].schedlink.set(batch[i+1])
    }
    var q gQueue
    q.head.set(batch[0])
    q.tail.set(batch[n])
 
    // Now put the batch on global queue.
    lock(&sched.lock)
    globrunqputbatch(&q, int32(n+1))
    unlock(&sched.lock)
    return true
}
```


runqputslow函数首先使用链表把从_p_的本地队列中取出的一半连同gp一起串联起来，然后在加锁成功之后通过globrunqputbatch函数把该链表链入全局运行队列（全局运行队列是使用链表实现的）。runqputslow函数并没有一开始就把全局运行队列锁住，而是等所有的准备工作做完之后才锁住全局运行队列，这是并发编程加锁的基本原则，需要尽量减小锁的粒度，降低锁冲突的概率。



## 例子1


```
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	total := 10
	wg.Add(total)

	for i := 0; i < total; i++ {
		go func(i int) {
			fmt.Println(i)
			wg.Done()
		}(i)
	}

	wg.Wait()
}
```


首先通过 runtime.GOMAXPROCS(1) 设置只有一个 P，接着创建了 10 个 goroutine，并分别打印出 i 值。你可以先想一下输出会是什么

执行结果是：


```
9
0
1
2
3
4
5
6
7
8
```


分析一下原因：

因为一开始就设置了只有一个 P，所以 for 循环里面“生产”出来的 goroutine 都会进入到 P 的 runnext 和本地队列

每次生产出来的 goroutine 都会第一时间塞到 runnext，而 i 从 1 开始，runnext 已经有 goroutine 在了，所以这时会把 old goroutine 移动 P 的本队队列中去，再把 new goroutine 放到 runnext。之后会重复这个过程……

因此这后当一次 i 为 9 时，新 goroutine 被塞到 runnext，其余 goroutine 都在本地队列。

等所有的子goroutine生成后，main goroutine 挂起，运行 P 的 runnext 和本地可运行队列里的 gorotuine。

而我们又知道，runnext 里的 goroutine 的执行优先级是最高的，因此会先打印出 9，接着再执行本地队列中的 goroutine 时，按照先进先出的顺序打印：9, 0, 1, 2, 3, 4, 5, 6, 7, 8



## 例子2


```
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	total := 10
	wg.Add(total)

	for i := 0; i < total; i++ {
		go func(i int) {
			fmt.Println(i)
			wg.Done()
		}(i)
	}

	wg.Wait()
	time.Sleep(time.Second)
}
```


和第一个例子的不同之处是我们在 Wait()之后多执行了一次 Sleep 操作。这一次，会是怎样的执行结果呢？


```
$ go1.13.8 run main.go

0
1
2
3
4
5
6
7
8
```

而当我们用 go1.14 及之后的版本运行时：


```
$ go1.14 run main.go

9
0
1
2
3
4
5
6
7
8
```


可以看到，用 go1.14 及之后的版本运行时，输出顺序和之前的一致。而用 go1.13 运行时，却先输出了 0，这又是什么原因呢？

主要是因为 Go 1.14 修改了 timer 的实现

go 1.13 的 time 包会生产一个名字叫 timerproc 的 goroutine 出来，它专门用于唤醒挂在 timer 上的时间未到期的 goroutine；因此这个 goroutine 会把 runnext 上的 goroutine 挤出去。因此输出顺序就是：0, 1, 2, 3, 4, 5, 6, 7, 8, 9

而 go 1.14 把这个唤醒的 goroutine 干掉了，取而代之的是，在调度循环的各个地方、sysmon 里都是唤醒 timer 的代码，timer 的唤醒更及时了。所以，输出顺序和第一个例子是一致的。



## 例子3


```
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	total := 258
	wg.Add(total)

	for i := 0; i < total; i++ {
		go func(i int) {
			fmt.Println(i)
			wg.Done()
		}(i)
	}

	wg.Wait()
}
```


和例子1不同的是，我们把goroutine的总个数换成了258个。为什么是258个，而不是257、256个呢？

这是因为我们知道P的本地队列 runq 大小是256个，前面也提到过当goroutine个数超过256个时，会将当前 P 的runq 队列中的一半 goroutine和 old goroutine 一起打包丢到 global queue 里。我们主要为了测试这个场景

因为还有个runnext特殊队列，所以goroutine总个数 > 256 + 1 = 257时，会触发这个逻辑，所以我们设置为258

那么执行结果时什么呢？

最终输出为：


```
257
128
129
.
.
.
186
187
0
188
189
.
.
.
246
247
1
248
249
250
251
252
253
254
255
2
3
4
5
6
7
8
9
.
.
.
124
125
126
127
256
```


解释下原因：

在生成第257个goroutine(id:256)时，P的本地队列刚好满却没有溢出。此时runnext是 256， runq 中依次是 0, 1, 2 ... 255

当生成第258个goroutine时，它首先会被放到 runnext 中，而之前runnext中的 old goroutine放到 runq时，由于此时runq中已经有256个goroutine放不下了，就会触发上面的移动逻辑：把 runq 队列中的一半 (0,1,2 ... 127) goroutine 和 old goroutine 256 打包放到全局队列中

所以此时，队列中的情况是 

- runnext : 257
- runq : 128, 129, 130 ... 255
- global queue : 0, 1, 2 ... 127, 256

按照优先级执行顺序，有的朋友可能就会提出疑问，最终结果不应该是： 257, 128, 129 ... 255, 0, 1, 2 ... 256 吗？为什么会有所差异？

其实在调度顺序时，还有个特殊逻辑，每个P会维护一个全局计数器，记录当前执行了多少次goroutine调度，为了避免P的本地队列一直有goroutine未执行完毕导致全局队列中的goroutine得不到调度而被饿死。所以每调度 61 次时，会去全局队列中看下是否有待执行的 goroutine，如果有，则暂停执行P的本地队列，先从全局队列中获取一个goroutine进行执行

所以先执行 runnext 中的 257；然后执行 runq 中的128、129，当执行60次也即是 187时，会去 global queue中拉取第一个 goroutine （0) 插入执行一次，然后再执行 runq 本地队列，后面也是按这个逻辑，所以最终结果会穿插执行0、1







