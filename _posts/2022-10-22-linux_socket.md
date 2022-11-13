---
layout: post
title: Linux内核中网络数据包的接收
category: [linux]
tags: [package]
description: 
---

与网络数据包的发送不同，网络收包是异步的的，因为你不确定谁会在什么时候突然发一个网络包给你，因此这个网络收包逻辑其实包含两件事：

1. 数据包到来后的通知
2. 收到通知并从数据包中获取数据

这两件事发生在协议栈的两端，即网卡/协议栈边界以及协议栈/应用边界：

- **网卡/协议栈边界**：网卡通知数据包到来，中断协议栈收包；
- **协议栈栈/应用边界**：协议栈将数据包填充 socket 队列，通知应用程序有数据可读，应用程序负责接收数据。

本文就来介绍一下关于这两个边界的这两件事是怎么一个细节，关乎网卡中断，NAPI，网卡 poll，select/poll/epoll 等细节，并假设你已经大约懂了这些。

## 网卡/协议栈边界的事件

网卡在数据包到来的时候会触发中断，然后协议栈就知道了数据包到来事件，接下来怎么收包完全取决于协议栈本身，这就是网卡中断处理程序的任务，当然，也可以不采用中断的方式，而是采用一个单独的线程不断轮询网卡是否有数据包的到来，但是这种方式过于消耗 CPU，做过多的无用功，因此基本被弃用，像这种异步事件，基本都是采用中断通知的方案。综合整个收包逻辑，大致可以分为以下两种方式

a. 每个数据包到来即中断 CPU，由 CPU 调度中断处理程序进行收包处理，收包逻辑又分为上半部和下半部，核心的协议栈处理逻辑在下半部完成。

b.数据包到来，中断 CPU，CPU 调度中断处理程序并且关闭中断响应，调度下半部不断轮询网卡，收包完毕或者达到一个阀值后，重新开启中断。

其中的方式 a 在数据包持续快速到达的时候会造成很大的性能损害，因此这种情况下一般采用方式 b，这也是 Linux NAPI 采用的方式。

关于网卡/协议栈边界所发生的事件，不想再说更多了，因为这会涉及到很多硬件的细节，比如你在 NAPI 方式下关了中断后，网卡内部是怎么缓存数据包的，另外考虑到多核处理器的情形，是不是可以将一个网卡收到的数据包中断到不同的 CPU 核心呢？那么这就涉及到了多队列网卡的问题，而这些都不是一个普通的内核程序员所能驾驭的，你要了解更多的与厂商相关的东西，比如 Intel 的各种规范，各种让人看到晕的手册...

## 协议栈/socket 边界事件

因此，为了更容易理解，我决定在另一个边界，即协议栈栈/应用边界来描述同样的事情，而这些基本都是内核程序员甚至应用程序员所感兴趣的领域了，为了使后面的讨论更容易进行，我将这个协议栈栈/应用边界重命名为协议栈/socket 边界，socket 隔离了协议栈和应用程序，它就是一个接口，对于协议栈，它可以代表应用程序，对于应用程序，它可以代表协议栈，当数据包到来的时候，会发生如下的事情：

1. 协议栈将数据包放入 socket 的接收缓冲区队列，并通知持有该 socket 的应用程序；
2. CPU 调度持有该 socket 的应用程序，将数据包从接收缓冲区队列中取出，收包完毕。

总体的示意图如下：

![1](/images/linux_socket/1.png)

## socket 要素

如上图所示，每一个 socket 的收包逻辑都包含以下两个要素

- 接收队列

  - 协议栈处理完毕的数据包要排入到的队列，应用程序被唤醒后要从该队列中读取数据。

- 睡眠队列
  - 与该 socket 相关的应用程序如果没有数据可读，可以在这个队列上睡眠，一旦协议栈将数据包排入 socket 的接收队列，将唤醒该睡眠队列上的进程或者线程
- 一把 socket 锁
  - 在有执行流操作 socket 的元数据的时候，必须锁定 socket，注意，接收队列和睡眠队列并不需要该锁来保护，该锁所保护的是类似 socket 缓冲区大小修改，TCP 按序接收之类的事情

这个模型非常简单且直接，和网卡中断 CPU 通知网络上有数据包到来需要处理一样，协议栈通过这种方式通知应用程序有数据可读，在继续讨论细节以及 select/poll/epoll 之前，先说两个无关的事，后面就不再说了，只是因为它们相关，所以只是提一下而已，不会占用大量篇幅。

1. 惊群与排他唤醒

类似 TCP accpet 逻辑这样，对于大型 web 服务器而言，基本上都是有多个进程或者线程同时在一个 Listen socket 上进行 accept，如果协议栈将一个客户端 socket 排入了 accept 队列，是将这些线程全部唤醒还是只唤醒一个呢？如果是全部唤醒，很显然，只有一个线程会抢到这个 socket，其它的线程抢夺失败后继续睡眠，可以说是被白白唤醒了，这就是经典的 TCP 惊群，因此产生了一种排他式的唤醒，也就是说只唤醒睡眠队列上的第一个线程，然后退出 wakeup 逻辑，不再唤醒后面的线程，这就避免了惊群。

这个话题在网上的讨论早就已经汗牛充栋，但是仔细想一下就会发现排他唤醒依然有问题，它会大大降低效率。

为什么这么说呢？因为协议栈的唤醒操作和应用程序的实际 Accept 操作之间是完全异步的，除非在协议栈唤醒应用程序的时候，应用程序恰好阻塞在 Accept 上，任何人都不能保证当时应用程序在干什么。举一个简单的例子，在多核系统上，协议栈方面同时来了多个请求，而且也恰恰有多个线程等待在睡眠队列上，如果能让这多个协议栈执行流同时唤醒这多个线程该有多好，但是由于一个 socket 只有一个 Accept 队列，因此对于该队列的排他唤醒机制基本上将这个畅想给打回去了，唯一一个 accpet 队列的排入/取出的带锁操作让整个流程串行化了，完全丧失了多核并行的优势，因此 REUSEPORT 以及基于此的 FastTCP 就出现了。(今天周末，仔细研究了 Linux kernel 4.4 版本带来的更新，真的让人眼前一亮啊)

2. REUSEPORT 与多队列

起初在了解到 google 的 reuseport 之前，我个人也做过一个类似的 patch，当时的想法正是来自于与多队列网卡的类比，既然一块网卡可以中断多个 CPU，一个 socket 上的数据可读事件为什么不能中断多个应用程序呢？然而 socket API 早就已经固定死了，这对我的想法造成了打击，因为一个 socket 就是一个文件描述符，表示一个五元组(非 connect UDP 的 socket 以及 Listen tcp 除外！)，协议栈的事件恰恰只跟一个五元组相关...因此为了让想法可行，只能在 socket API 之外做文章，那就是允许多个 socket 绑定同样的 IP 地址/源端口对，然后按照源 IP 地址/端口对的 HASH 值来区分流量的路由，这个想法本人也实现了，其实跟多队列网卡是一个思想，完全一致的。多队列网卡不也是按照不同五元组(或者 N 元组？咱不较真儿)的 HASH 值来中断不同的 CPU 核心的吗？仔细想想当时的这个移植太 TMD 帅了，然而看到 google 的 reuseport patch 就觉得自己做了无用功，重新造了轮子...于是就想解决 Accept 单队列的问题，既然已经多核时代了，为什么不在每个 CPU 核心上保持一个 accept 队列呢？应用程序的调度让 schedule 子系统去考虑吧...这次没有犯傻，于是看到了新浪的 FastTCP 方案。

当然，如果 REUSEPORT 的基于源 IP/源端口对的 hash 计算，直接避免了将同一个流“中断”到不同的 socket 的接收队列上

SO_REUSEPORT 支持多个进程或线程绑定到同一端口，提高服务器程序的性能，解决问题：

- 允许多个套接字 bind()/listen() 同一个 TCP/UDP 端口
  - 每一个线程拥有自己的服务器套接字
  - 在服务器套接字上没有了锁的竞争
- 内核层面实现负载均衡
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面

其核心的实现主要有三点

- 扩展 socket option，增加 SO_REUSEPORT 选项，用来设置 reuseport
- 修改 bind 系统调用实现，以便支持可以绑定到相同的 IP 和端口
- 修改处理新建连接的实现，查找 listener 的时候，能够支持在监听相同 IP 和端口的多个 sock 之间均衡选择

Q1：什么是 reuseport？

A1：reuseport 是一种套接字复用机制，它允许你将多个套接字 bind 在同一个 IP 地址/端口对上，这样一来，就可以建立多个服务来接受到同一个端口的连接。

Q2：当来了一个连接时，系统怎么决定到底是哪个套接字来处理它？

A2：对于不同的内核，处理机制是不一样的，总的说来，reuseport 分为两种模式，即热备份模式和负载均衡模式，在早期的内核版本中，即便是加入对 reuseport 选项的支持，也仅仅为热备份模式，而在 3.9 内核之后，则全部改为了负载均衡模式，两种模式没有共存

Q3：什么是热备份模式和负载均衡模式呢？

A3： 热备份模式：即你创建了 N 个 reuseport 的套接字，然而工作的只有一个，其它的作为备份，只有当前一个套接字不再可用的时候，才会由后一个来取代，其投入工作的顺序取决于实现。

负载均衡模式：即你创建的所有 N 个 reuseport 的套接字均可以同时工作，当连接到来的时候，系统会取一个套接字来处理它。这样就可以达到负载均衡的目的，降低某一个服务的压力。

Q4：到底怎么取套接字呢？

A4：这个对于热备份模式和负载均衡模式是不同的。
热备份模式：一般而言，会将所有的 reuseport 同一个 IP 地址/端口的套接字挂在一个链表上，取第一个即可，如果该套接字挂了，它会被从链表删除，然后第二个便会成为第一个。

负载均衡模式：和热备份模式一样，所有 reuseport 同一个 IP 地址/端口的套接字会挂在一个链表上，你也可以认为是一个数组，这样会更加方便，当有连接到来时，用数据包的源 IP/源端口作为一个 HASH 函数的输入，将结果对 reuseport 套接字数量取模，得到一个索引，该索引指示的数组位置对应的套接字便是工作套接字

## 接收队列的管理

接收队列的管理其实非常简单，就是一个 skb 链表，协议栈将 skb 插入到链表的时候先 lock 住队列本身，然后插入 skb，然后唤醒 socket 睡眠队列上的线程，接着线程加锁获取 socket 接收队列上 skb 的数据，就是这么简单。

起码在 2.6.8 的内核上就是这么搞的。后来的版本都是这个基础版本的优化版本，先后经历了两次的优化

### 接收路径优化 1：引入 backlog 队列

考虑到复杂的细节，比如根据收到的数据修改 socket 缓冲区大小时，应用程序在调用 recv 例程时需要对整个 socket 进行锁定，在复杂的多核 CPU 环境中，有多个应用程序可能会操作同一个 socket，有多个协议栈执行流也可能会往同一个 socket 接收缓冲区排入 skb[详情请参考《 多核心 Linux 内核路径优化的不二法门之-多核心平台 TCP 优化》]，因此锁的粒度范围自然就变成了 socket 本身。在应用程序持有 socket 的时候，协议栈由于可能会在软中断上下文运行，是不可睡眠等待的，为了使得协议栈执行流不至于因此而自旋阻塞，引入了一个 backlog 队列，协议栈在应用程序持有 socket 的时候，只需要将 skb 排入 backlog 队列就可以返回了，那么这个 backlog 队列最终由谁来处理呢？

谁找的事谁来处理！当初就是因为应用程序 lock 住了 socket 而使得协议栈不得不将 skb 排入 backlog，那么在应用程序 release socket 的时候，就会将 backlog 队列里面的 skb 放入接收队列中去，模拟协议栈将 skb 入队并唤醒操作。

引入 backlog 队列后，单一的接收队列变成了一个两阶段的接力队列，类似流水线作业那样。这样无论如何协议栈都不用阻塞等待，协议栈如果不能马上将 skb 排入接收队列，那么这件事就由 socket 锁定者自己来完成，等到它放弃锁定的时候这件事即可进行。操作例程如下：

```
协议栈排队skb---
       获取socket自旋锁
       应用程序占有socket的时候：将skb排入backlog队列
       应用程序未占有socket的时候：将skb排入接收队列，唤醒接收队列
       释放socket自旋锁
应用程序接收数据---
      获取socket自旋锁
             阻塞占用socket
      释放socket自旋锁
      读取数据：由于已经独占了socket，可以放心地将接收队列skb的内容拷贝到用户态
      获取socket自旋锁
            将backlog队列的skb排入接收队列(这其实本该由协议栈完成的，但是由于应用程序占有socket而被延后到了此刻)，唤醒睡眠队列
      释放socket自旋锁

```

可以看到，所谓的 socket 锁，并不是一把简单的自旋锁，而是在不同的路径有不同的锁定方式，总之，只要能保证 socket 的元数据受到保护，方案都是合理的，于是我们看到这是一个两层锁的模型。

**两层锁定的 lock 框架**

啰嗦了这么多，其实我们可以把上面最后的那个序列总结成一个更为抽象通用的模式，在某些场景下可以套用。现在就描述一下这个模式。

参与者类别：NON-Sleep-不可睡眠类(内核 task)，Sleep-可睡眠类(用户 task)

参与者数量：NON-Sleep 多个，Sleep 类多个

竞争者：NON-Sleep 类之间，Sleep 类之间，NON-Sleep 类和 Sleep 类之间

数据结构：

- X-被锁定实体
- X.LOCK-自旋锁，用于锁定不可睡眠路径以及保护标记锁
- X.FLAG-标记锁，用来锁定可睡眠路径
- X.sleeplist-等待获得标记锁的 task 队列

**NON-Sleep 类的锁定/解锁逻辑：**

```
spin_lock(X.LOCK);

if(X.FLAG == 1) {
    //add something todo to backlog
    delay_func(...);
} else {
    //do it directly
    direct_func(...);
}

spin_unlock(X.LOCK);

```

**Sleep 类的锁定/解锁逻辑：**

```
spin_lock(X.LOCK);
do {
    if (X.FLAG == 0) {
        break;
    }
    for (;;) {
        ready_to_wait(X.sleeplist);
        spin_unlock(X.lock);
        wait();
        spin_lock(X.lock);
        if (X.FLAG == 0) {
            break;
        }
    }
} while(0);
X.FLAG = 1;
spin_unlock(X.LOCK);

do_something(...);

spin_lock(X.LOCK)
if (have_delayed_work) {
    do {
        fetch_delayed_work(...);
        direct_func(...);
    } while(have_delayed_work);
}
X.FLAG = 0;
wakeup(X.sleeplist);
spin_unlock(X.LOCK);

```

对于 socket 收包逻辑，其实就是将 skb 插入接收队列并唤醒 socket 的睡眠队列填充到上述的 direct_func 中即可，同时 delay_func 的任务就是将 skb 插入到 backlog 队列。

该抽象出来的模型基本就是一个两层锁逻辑，自旋锁在可睡眠路径仅仅用来保护标记位，可睡眠路径使用标记位来锁定而不是使用自旋锁本身，标记位的修改被自旋锁保护，这个非常快的修改操作代替了慢速的业务逻辑处理路径(比如 socket 收包...)的完全锁定，这样就大大减少了竞态带来的 CPU 时间的自旋开销。近期我在实际的一个场景中就采用了这个模型，非常不错，效果也真的还好，因此特意抽象出了上述的代码。

引入这个两层锁解放了不可睡眠路径的操作，使其在可睡眠路径的 task 占有一个 socket 期间仍然可以将数据包排入到 backlog 队列而不是等待可睡眠路径 task 解锁，然而有的时候可睡眠路径上的逻辑也不是那么慢，如果它不慢，甚至很快，锁定时间很短，那么是不是就可以直接跟不可睡眠路径去争抢自旋锁了呢？这正是引入可睡眠路径 fast lock 的契机。

### 接收路径优化 2：引入 fast lock

进程/线程上下文中的 socket 处理逻辑在满足下列情况的前提下可以直接与内核协议栈竞争该 socket 的自旋锁：

a. 处理临界区非常小

b. 当前没有其它进程/线程上下文中的 socket 处理逻辑正在处理这个 socket。

满足以上条件的，说明这是一个单纯的环境，竞争者地位对等。那么很显然的一个问题就是谁来处理 backlog 队列的问题，这个问题其实不是问题，因为这种情况下 backlog 就用不到了，操作 backlog 必须持有自旋锁，socket 在 fast lock 期间也是持有自旋锁的，两个路径完全互斥！因此上述条件 a 就极其重要，如果在临界区内出现了大的延迟，会造成协议栈路径过度自旋！新的 fast lock 框架如下

**Sleep 类的 fast 锁定/解锁逻辑：**

```
fast = 0;
spin_lock(X.LOCK)
do {
    if (X.FLAG == 0) {
        fast = 0;
        break;
    }
    for (;;) {
        ready_to_wait(X.sleeplist);
        spin_unlock(X.LOCK);
        wait();
        spin_lock(X.LOCK);
        if (X.FLAG == 0) {
            break;
        }
    }
    X.FLAG = 1;
    spin_unlock(X.LOCK);
} while(0);

do_something_very_small(...);

do {
    if (fast == 1) {
        break;
    }
    spin_lock(X.LOCK);
    if (have_delayed_work) {
        do {
            fetch_delayed_work(...);
            direct_func(...);
        } while(have_delayed_work);
    }
    X.FLAG = 0;
    wakeup(X.sleeplist);
} while(0);
spin_unlock(X.LOCK);
```

之所以上述代码那么复杂而不是仅仅的 spin_lock/spin_unlock，是因为如果 X.FLAG 为 1，说明该 socket 已经在处理了，比如阻塞等待。

以上就是在协议栈/socket 边界上的异步流程的队列和锁的总体架构，总结一下，包含 5 个要素：

- a=socket 的接收队列
- b=socket 的睡眠队列
- c=socket 的 backlog 队列
- d=socket 的自旋锁
- e=socket 的占有标记

这 5 者之间执行以下的流程

![2](/images/linux_socket/2.png)

有了这个框架，协议栈和 socket 之间就可以安全异步地进行网络数据的交接了，如果你仔细看，并且对 Linux 2.6 内核的 wakeup 机制有足够的了解，且有一定的解耦合的思想，我想应该可以知道 select/poll/epoll 是怎样一种工作机制了。关于这个我想在本文的第二部分描述，我觉得，只要对基础概念有足够的理解且可以融会贯通，很多东西都是可以仅仅靠想而推导出来的。

## 接力传递的 sk

在 Linux 的协议栈实现中，skb 表示一个数据包，一个 skb 可以属于一个 socket 或者协议栈，但不能同时属于两者，一个 skb 属于协议栈指的是它不和任何一个 socket 相关联，它仅对协议栈本身负责，如果一个 skb 属于一个 socket，就意味着它已经和一个 socket 进行了绑定，所有的关于它的操作，都要由该 socket 负责。

Linux 为 skb 提供了一个 destructor 析构回调函数，每当 skb 被赋予新的属主的时候会调用前一个属主的析构函数，并被指定一个新的析构函数，我们比较关注的是 skb 从协议栈到 socket 的这最后一棒，在将 skb 排入到 socket 接收队列之前，会调用下面的函数：

```
static inline void skb_set_owner_r(struct sk_buff *skb, struct sock *sk)
{
    skb_orphan(skb);
    skb->sk = sk;
    skb->destructor = sock_rfree;
    atomic_add(skb->truesize, &sk->sk_rmem_alloc);
    sk_mem_charge(sk, skb->truesize);
}

```

其中 skb_orphan 主要是回调了前一个属主赋予该 skb 的析构函数，然后为其指定一个新的析构回调函数 sock_rfree。在 skb_set_owner_r 调用完成后，该 skb 就正式进入 socket 的接收队列了：

```
skb_set_owner_r(skb, sk);

/* Cache the SKB length before we tack it onto the receive
 * queue.  Once it is added it no longer belongs to us and
 * may be freed by other threads of control pulling packets
 * from the queue.
 */
skb_len = skb->len;

skb_queue_tail(&sk->sk_receive_queue, skb);

if (!sock_flag(sk, SOCK_DEAD))
    sk->sk_data_ready(sk, skb_len);


```

最后通过调用 sk_data_ready 来通知睡眠在 socket 睡眠队列上的 task 数据已经被排入接收队列，其实就是一个 wakeup 操作，然后协议栈就返回。很显然，接下来关于该 skb 的所有处理均在进程/线程上下文中进行了，等到 skb 的数据被取出后，这个 skb 不会返回给协议栈，而是由进程/线程自行释放，因此在其 destructor 回调函数 sock_rfree 中，主要做的就是把缓冲区空间还给系统，主要做两件事：

1. 该 socket 已分配的内存减去该 skb 占据的空间
   1. sk->sk_rmem_alloc = sk->sk_rmem_alloc - skb->truesize;
2. 该 socket 预分配的空间加上该 skb 占据的空间
   1. sk->sk_forward_alloc = sk->sk_forward_alloc + skb->truesize;

## 协议数据包内存用量的统计和限制

内核协议栈仅仅是内核的一个子系统，且其数据来自本机之外，数据来源并不受控，很容易受到 DDos 攻击，因此有必要限制一个协议的总体内存用量，比如所有的 TCP 连接只能用 10M 的内存这类，Linux 内核起初仅仅针对 TCP 做统计，后来也加入了针对 UDP 的统计限制，在配置上体现为几个 sysctl 参数：

- net.ipv4.tcp_mem = 18978 25306 37956
- net.ipv4.tcp_rmem = 4096 87380 6291456
- net.ipv4.tcp_wmem = 4096 16384 4194304
- net.ipv4.udp_mem = 18978 25306 37956
- ....

以上的每一项三个值中，含义如下：

- 第一个值 mem[0]：表示正常值，凡是内存用量低于这个值时，都正常；
- 第二个值 mem[1]：警告值，凡是高于这个值，就要着手紧缩方案了；
- 第三个值 mem[2]：不可逾越的界限，高于这个值，说明内存使用已经超限了，数据要丢弃了。

注意，这些配置值是针对单独协议的，而 sockopt 中配置的 recvbuff 配置的是针对单独一条连接的缓冲区大小限制，两者是不同的。内核在处理这个协议限额的时候，为了避免频繁检测，采用了预分配机制，第一次即便只是来了一个 1byte 的包，也会为其透支一个页面的内存限额，这里并没有实际进行内存分配，因为实际的内存分配在 skb 生成以及 IP 分片重组的时候就已经确定了，这里只是将这些值累加起来，检测一下是否超过限额而已，因此这里的逻辑仅仅是一个加减乘除的过程，除了计算过程消耗的 CPU 之外，并没有消耗其它的机器资源

计算方法如下

- proto.memory_allocated：每一个协议一个，表示当前该协议在内核 socket 缓冲区里面一共已经使用了多少内存存放 skb；
- sk.sk_forward_alloc：每一个 socket 一个，表示当前预分配给该 socket 的内存剩余用量，可以用来存放 skb；
- skb.truesize：该 skb 结构体本身的大小以及其数据大小的总和；

skb 即将进入 socket 的接收队列前夕的累加例程：

```
ok = 0;
if (skb.truesize < sk.sk_forward_alloc) {
    ok = 1;
    goto addload;
}
pages = how_many_pages(skb.truesize);

tmp = atomic_add(proto.memory_allocated, pages*page_size);
if (tmp < mem[0]) {
    ok = 1;
    正常;
}

if (tmp > mem[1]) {
    ok = 2;
    吃紧;
}

if (tmp > mem[2]) {
    超限;
}

if (ok == 2) {
    if (do_something(proto)) {
        ok = 1;
    }
}

addload:
if (ok == 1) {
    sk.sk_forward_alloc = sk.sk_forward_alloc - skb.truesize;
    proto.memory_allocated = tmp;
} else {
    drop skb;
}

```

skb 被 socket 释放时调用析构函数时期的 sk.sk_forward_alloc 延展：

```
sk.sk_forward_alloc = sk.sk_forward_alloc + skb.truesize;

```

协议缓冲区回收时期(会在释放 skb 或者过期删除 skb 时调用)：

```
if (sk.sk_forward_alloc > page_size) {
    pages = sk.sk_forward_alloc调整到整页面数;
    prot.memory_allocated = prot.memory_allocated - pages*page_size;
}

```

这个逻辑可以在 sk_rmem_schedule 等 sk_mem_XXX 函数中看个究竟。

## Linux 2.6+内核的 wakeup callback 机制

Linux 内核通过睡眠队列来组织所有等待某个事件的 task，而 wakeup 机制则可以异步唤醒整个睡眠队列上的 task，每一个睡眠队列上的节点都拥有一个 callback，wakeup 逻辑在唤醒睡眠队列时，会遍历该队列链表上的每一个节点，调用每一个节点的 callback，如果遍历过程中遇到某个节点是排他节点，则终止遍历，不再继续遍历后面的节点。总体上的逻辑可以用下面的伪代码表示：

### 睡眠等待

```
define sleep_list;
define wait_entry;
wait_entry.task= current_task;
wait_entry.callback = func1;
if (something_not_ready); then
    # 进入阻塞路径
    add_entry_to_list(wait_entry, sleep_list);
go on:
    schedule();
    if (something_not_ready); then
        goto go_on;
    endif
    del_entry_from_list(wait_entry, sleep_list);
endif
...

```

### 唤醒机制

```
something_ready;
for_each(sleep_list) as wait_entry; do
    wait_entry.callback(...);
    if(wait_entry.exclusion); then
        break;
    endif
done

```

我们只需要狠狠地关注这个 callback 机制，它能做的事真的不止 select/poll/epoll，Linux 的 AIO 也是它来做的，注册了 callback，你几乎可以让一个阻塞路径在被唤醒的时候做任何事情。一般而言，一个 callback 里面都是以下的逻辑：

```
common_callback_func(...)
{
    do_something_private;
    wakeup_common;
}

```

其中，do_something_private 是 wait_entry 自己的自定义逻辑，而 wakeup_common 则是公共逻辑，旨在将该 wait_entry 的 task 加入到 CPU 的就绪 task 队列，然后让 CPU 去调度它。

Linux 通过 socket 睡眠队列来管理所有等待 socket 的某个事件的 process，同时通过 wakeup 机制来异步唤醒整个睡眠队列上等待事件的 process，通知 process 相关事件发生。通常情况，socket 的事件发生的时候，其会顺序遍历 socket 睡眠队列上的每个 process 节点，调用每个 process 节点挂载的 callback 函数。在遍历的过程中，如果遇到某个节点是排他的，那么就终止遍历，总体上会涉及两大逻辑：**(1). 睡眠等待逻辑；(2). 唤醒逻辑**。

1. 睡眠等待逻辑：涉及 select、poll、epoll_wait 的阻塞等待逻辑
   1. select、poll、epoll_wait 陷入内核，判断监控的 socket 是否有关心的事件发生了，如果没，则为当前 process 构建一个 wait_entry 节点，然后插入到监控 socket 的 sleep_list
   2. 进入循环的 schedule 直到关心的事件发生了
   3. 关心的事件发生后，将当前 process 的 wait_entry 节点从 socket 的 sleep_list 中删除。
2. 唤醒逻辑：
   1. socket 的事件发生了，然后 socket 顺序遍历其睡眠队列，依次调用每个 wait_entry 节点的 callback 函数
   2. 直到完成队列的遍历或遇到某个 wait_entry 节点是排他的才停止。
   3. 一般情况下 callback 包含两个逻辑：
      1. wait_entry 自定义的私有逻辑；
      2. 唤醒的公共逻辑，主要用于将该 wait_entry 的 process 放入 CPU 的就绪队列，让 CPU 随后可以调度其执行

## select/poll 的逻辑

要知道，在大多数情况下，要高效处理网络数据，一个 task 一般会批量处理多个 socket，哪个来了数据就去读那个，这就意味着要公平对待所有这些 socket，你不可能阻塞在任何 socket 的“数据读”上，也就是说你不能在阻塞模式下针对任何 socket 调用 recv/recvfrom，这就是多路复用 socket 的实质性需求。

假设有 N 个 socket 被同一个 task 处理，怎么完成多路复用逻辑呢？很显然，我们要等待“数据可读”这个事件，而不是去等待“实际的数据”！！我们要阻塞在事件上，该事件就是“N 个 socket 中有一个或多个 socket 上有数据可读”，也就是说，只要这个阻塞解除，就意味着一定有数据可读，意味着接下来调用 recv/recvform 一定不会阻塞！另一方面，这个 task 要同时排入所有这些 socket 的 sleep_list 上，期待任意一个 socket 只要有数据可读，都可以唤醒该 task。

那么，select/poll 这类多路复用模型的设计就显而易见了。

select/poll 的设计非常简单，为每一个 socket 引入一个 poll 逻辑，该逻辑用于收集 socket 发生的事件，对于“数据可读”的判断如下：

```
poll()
{
    ...
    if (接收队列不为空) {
        ev |= POLL_IN;
    }
    ...
}

```

当 task 调用 select/poll 的时候，如果没有数据可读，task 会阻塞，此时它已经排入了所有 N 个 socket 的 sleep_list，只要有一个 socket 来了数据，这个 task 就会被唤醒，接下来的事情就是

```
for_each_N_socket as sk; do
    event.evt = sk.poll(...);
    event.sk = sk;
    put_event_to_user;
done;

```

当用户 process 调用 select 的时候，select 会将需要监控的 readfds 集合拷贝到内核空间（假设监控的仅仅是 socket 可读），然后遍历自己监控的 socket sk，挨个调用 sk 的 poll 逻辑以便检查该 sk 是否有可读事件，遍历完所有的 sk 后，如果没有任何一个 sk 可读，那么 select 会调用 schedule_timeout 进入 schedule 循环，使得 process 进入睡眠。如果在 timeout 时间内某个 sk 上有数据可读了，或者等待 timeout 了，则调用 select 的 process 会被唤醒，接下来 select 就是遍历监控的 sk 集合，挨个收集可读事件并返回给用户了

可见，select/poll 非常原始，如果有 100000 个 socket(夸张吗？)，有一个 socket 可读，那么系统不得不遍历一遍...因此 select 只限制了最多可以复用 1024 个 socket，并且在 Linux 上这是宏控制的。select/poll 只是朴素地实现了 socket 的多路复用，根本不适合大容量网络服务器的处理场景。其瓶颈在于，不能随着 socket 的增多而战时扩展性。

## epoll 对 wait_entry callback 的利用

既然一个 wait_entry 的 callback 可以做任意事，那么能否让其做的比 select/poll 场景下的 wakeup_common 更多呢？

为此，epoll 准备了一个链表，叫做 ready_list，所有处于 ready_list 中的 socket，都是有事件的，对于数据读而言，都是确实有数据可读的。epoll 的 wait_entry 的 callback 要做的就是，将自己自行加入到这个 ready_list 中去，等待 epoll_wait 返回的时候，只需要遍历 ready_list 即可。epoll_wait 睡眠在一个单独的队列(single_epoll_waitlist)上，而不是 socket 的睡眠队列上。

和 select/poll 不同的是，使用 epoll 的 task 不需要同时排入所有多路复用 socket 的睡眠队列，这些 socket 都拥有自己的队列，task 只需要睡眠在自己的单独队列中等待事件即可，每一个 socket 的 wait_entry 的 callback 逻辑为：

```
epoll_wakecallback(...)
{
    add_this_socket_to_ready_list;
    wakeup_single_epoll_waitlist;
}

```

为此，epoll 需要一个额外的调用，那就是 epoll_ctrl ADD，将一个 socket 加入到 epoll table 中，它主要提供一个 wakeup callback，将这个 socket 指定给一个 epoll entry，同时会初始化该 wait_entry 的 callback 为 epoll_wakecallback。整个 epoll_wait 以及协议栈的 wakeup 逻辑如下所示：

协议栈唤醒 socket 的睡眠队列

1. 数据包排入了 socket 的接收队列;；
2. 唤醒 socket 的睡眠队列，即调用各个 wait_entry 的 callback；
3. callback 将自己这个 socket 加入 ready_list；
4. 唤醒 epoll_wait 睡眠在的单独队列。
5. 遍历 epoll 的 ready_list，挨个调用每个 sk 的 poll 逻辑收集发生的事件
6. 将每个 sk 收集到的事件，通过 epoll_wait 传入的 events 数组回传并唤醒相应的 process

自此，epoll_wait 继续前行，遍历调用 ready_list 里面每一个 socket 的 poll 历程，搜集事件。这个过程是例行的，因为这是必不可少的，ready_list 里面每一个 socket 都有数据可读，做不了无用功，这是和 select/poll 的本质区别(select/poll 中，即便没有数据可读，也要全部遍历一遍)。

总结一下，epoll 逻辑要做以下的例程

### epoll add 逻辑

```
define wait_entry
wait_entry.socket = this_socket;
wait_entry.callback = epoll_wakecallback;
add_entry_to_list(wait_entry, this_socket.sleep_list);

```

###

```
define single_wait_list
define single_wait_entry
single_wait_entry.callback = wakeup_common;
single_wait_entry.task = current_task;
if (ready_list_is_empty); then
    # 进入阻塞路径
    add_entry_to_list(single_wait_entry, single_wait_list);
go on:
    schedule();
    if (sready_list_is_empty); then
        goto go_on;
    endif
    del_entry_from_list(single_wait_entry, single_wait_list);
endif
for_each_ready_list as sk; do
    event.evt = sk.poll(...);
    event.sk = sk;
    put_event_to_user;
done;

```

### epoll 唤醒的逻辑

```
add_this_socket_to_ready_list;
wakeup_single_wait_list;

```

综合以上，可以给出下面的关于 epoll 的流程图，可以对比本文第一部分的流程图做比较

![3](/images/linux_socket/3.png)

epoll 巧妙的引入一个中间层解决了大量监控 socket 的无效遍历问题。epoll 在中间层上为每个监控的 socket 准备了一个单独的回调函数 epoll_callback_sk，而对于 select/poll，所有的 socket 都公用一个相同的回调函数。正是这个单独的回调 epoll_callback_sk 使得每个 socket 都能单独处理自身，当自己就绪的时候将自身 socket 挂入 epoll 的 ready_list。同时，epoll 引入了一个睡眠队列 single_epoll_wait_list，分割了两类睡眠等待。process 不再睡眠在所有的 socket 的睡眠队列上，而是睡眠在 epoll 的睡眠队列上，在等待”任意一个 socket 可读就绪”事件。而中间 wait_entry_sk 则代替 process 睡眠在具体的 socket 上，当 socket 就绪的时候，它就可以处理自身了。

## epoll 的 ET 和 LT

是时候提到 ET 和 LT 了，最大的争议在于哪个性能高，而不是到底怎么用。各种文档上都说 ET 高效，但事实上，根本不是这样，对于实际而言，LT 高效的同时，更安全。两者到底什么区别呢？

### 概念上的区别

- ET：只有状态发生变化的时候，才会通知，比如数据缓冲区从无到有的时候(不可读-可读)，如果缓冲区里面有数据，便不会一直通知；
- LT：只要缓冲区里面有数据，就会一直通知

其实，sk 从 ready_list 移除的时机正是区分两种事件模式的本质。因为，通过上面的介绍，我们知道 ready_list 是否为空是 epoll_wait 是否返回的条件。于是，在两种事件模式下，步骤 5 如下：

- 对于 Edge Triggered (ET) 边沿触发：
  - 遍历 epoll 的 ready_list，将 sk 从 ready_list 中移除，然后调用该 sk 的 poll 逻辑收集发生的事件
- 对于 Level Triggered (LT) 水平触发
  - 遍历 epoll 的 ready_list，将 sk 从 ready_list 中移除，然后调用该 sk 的 poll 逻辑收集发生的事件
  - 如果该 sk 的 poll 函数返回了关心的事件(对于可读事件来说，就是 POLL_IN 事件)，那么该 sk 被重新加入到 epoll 的 ready_list 中。

对于可读事件而言，在 ET 模式下，如果某个 socket 有新的数据到达，那么该 sk 就会被排入 epoll 的 ready_list，从而 epoll_wait 就一定能收到可读事件的通知(调用 sk 的 poll 逻辑一定能收集到可读事件)。于是，我们通常理解的缓冲区状态变化(从无到有)的理解是不准确的，准确的理解应该是是否有新的数据达到缓冲区。

而在 LT 模式下，某个 sk 被探测到有数据可读，那么该 sk 会被重新加入到 read_list，那么在该 sk 的数据被全部取走前，下次调用 epoll_wait 就一定能够收到该 sk 的可读事件(调用 sk 的 poll 逻辑一定能收集到可读事件)，从而 epoll_wait 就能返回。

### 实现上的区别

在代码实现的逻辑上，ET 和 LT 实现的区别在于 LT 一旦有事件则会一直加进 ready_list，直到下一次的 poll 将其移出，然后在探测到感兴趣事件后再将其加进 ready_list。由 poll 例程来判断是否有事件，而不是完全依赖 wakeup callback，这是真正意义的 poll，即不断轮询！也就是说，LT 模式是完全轮询的，每次都会去 poll 一次，直到 poll 不到感兴趣的事件，才会歇息，此时就只有数据包的到来可以重新依赖 wakeup callback 将其加入 ready_list 了。在实现上，从下面的代码可以看出二者的差异。

```
epoll_wait
for_each_ready_list_item as entry; do
    remove_from_ready_list(entry);
    event = entry.poll(...);
    if (event) then
        put_user;
        if (LT) then
            # 以下一次poll的结论为结果
            add_entry_to_ready_list(entry);
        endif
    endif
done

```

### 性能上的区别

性能的区别主要体现在数据结构的组织以及算法上，对于 epoll 而言，主要就是链表操作和 wakeup callback 操作，对于 ET 而言，是 wakeup callback 将 socket 加入到 ready_list，而对于 LT 而言，则除了 wakeup callback 可以将 socket 加入到 ready_list 之外，epoll_wait 也可以将其为了下一次的 poll 加入到 ready_list，wakeup callback 中反而有更少工作量，但这并不是性能差异的根本，性能差异的根本在于链表的遍历，如果有海量的 socket 采用 LT 模式，由于每次发生事件后都会再次将其加入 ready_list，那么即便是该 socket 已经没有事件了，还是会用一次 poll 来确认，这额外的一次对于无事件 socket 没有意义的遍历在 ET 上是没有的。但是注意，遍历链表的性能消耗只有在链表超长时才会体现，你觉得千儿八百的 socket 就会体现 LT 的劣势吗？诚然，ET 确实会减少数据可读的通知次数，但这事实上并没有带来压倒性的优势。

先不说第一次 epoll_wait 返回的时候，用户进程能否都将有数据返回的 socket 处理掉。在用户处理的过程中，如果该 socket 有新的数据上来，那么协议栈发现 sk 已经在 ready_list 中了，那么就不需要再次放入 ready_list，也就是在 LT 模式下，对该 sk 的再次遍历不是多余的，是有效的。同时，我们回归 epoll 高效的场景在于，服务器有海量 socket，但是活跃 socket 较少的情况下才会体现出 epoll 的高效、高性能。因此，在实际的应用场合，绝大多数情况下，ET 模式在性能上并不会比 LT 模式具有压倒性的优势，至少，目前还没有实际应用场合的测试表面 ET 比 LT 性能更好。

### 编程上的区别

我们知道，对于可读事件而言，在阻塞模式下，是无法识别队列空的事件的，并且，事件通知机制，仅仅是通知有数据，并不会通知有多少数据。于是，在阻塞模式下，在 epoll_wait 返回的时候，我们对某个 socket_fd 调用 recv 或 read 读取并返回了一些数据的时候，我们不能再次直接调用 recv 或 read，因为，如果 socket_fd 已经无数据可读的时候，进程就会阻塞在该 socket_fd 的 recv 或 read 调用上，这样就影响了 IO 多路复用的逻辑(我们希望是阻塞在所有被监控 socket 的 epoll_wait 调用上，而不是单独某个 socket_fd 上)，造成其他 socket 饿死，即使有数据来了，也无法处理。

接下来，我们只能再次调用 epoll_wait 来探测一些 socket_fd，看是否还有数据可读。在 LT 模式下，如果 socket_fd 还有数据可读，那么 epoll_wait 就一定能够返回，接着，我们就可以对该 socket_fd 调用 recv 或 read 读取数据。然而，在 ET 模式下，尽管 socket_fd 还是数据可读，但是如果没有新的数据上来，那么 epoll_wait 是不会通知可读事件的。这个时候，epoll_wait 阻塞住了，这下子坑爹了，明明有数据你不处理，非要等新的数据来了在处理，那么我们就死扛咯，看谁先忍不住。

等等，在阻塞模式下，不是不能用 ET 的么？是的，正是因为有这样的缺点，ET 强制需要在非阻塞模式下使用。在 ET 模式下，epoll_wait 返回 socket_fd 有数据可读，我们必须要读完所有数据才能离开。因为，如果不读完，epoll 不会在通知你了，虽然有新的数据到来的时候，会再次通知，但是我们并不知道新数据会不会来，以及什么时候会来。由于在阻塞模式下，我们是无法通过 recv/read 来探测空数据事件，于是，我们必须采用非阻塞模式，一直 read 直到 EAGAIN。因此，ET 要求 socket_fd 非阻塞也就不难理解了。

另外，epoll_wait 原本的语意是：监控并探测 socket 是否有数据可读(对于读事件而言)。LT 模式保留了其原本的语意，只要 socket 还有数据可读，它就能不断反馈，于是，我们想什么时候读取处理都可以，我们永远有再次 poll 的机会去探测是否有数据可以处理，这样带来了编程上的很大方便，不容易死锁造成某些 socket 饿死。相反，ET 模式修改了 epoll_wait 原本的语意，变成了：监控并探测 socket 是否有新的数据可读。

于是，在 epoll_wait 返回 socket_fd 可读的时候，我们需要小心处理，要不然会造成死锁和 socket 饿死现象。典型如 listen_fd 返回可读的时候，我们需要不断的 accept 直到 EAGAIN。假设同时有三个请求到达，epoll_wait 返回 listen_fd 可读，这个时候，如果仅仅 accept 一次拿走一个请求去处理，那么就会留下两个请求，如果这个时候一直没有新的请求到达，那么再次调用 epoll_wait 是不会通知 listen_fd 可读的，于是 epoll_wait 只能睡眠到超时才返回，遗留下来的两个请求一直得不到处理，处于饿死状态。

在绝大多少情况下，ET 模式并不会比 LT 模式更为高效，同时，ET 模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑和坑点。因此，建议还是采用 LT 模式来编程更为舒爽

## Accept"惊群"现象

惊群效应（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

对于高性能的服务器而言，为了利用多 CPU 核的优势，大多采用多个进程(线程)同时在一个 listen socket 上进行 accept 请求。多个进程阻塞在 Accept 调用上，那么在协议栈将 Client 的请求 socket 放入 listen socket 的 accept 队列的时候，是要唤醒一个进程还是全部进程来处理呢？

linux 内核通过睡眠队列来组织所有等待某个事件的 task，而 wakeup 机制则可以异步唤醒整个睡眠队列上的 task，wakeup 逻辑在唤醒睡眠队列时，会遍历该队列链表上的每一个节点，调用每一个节点的 callback，从而唤醒睡眠队列上的每个 task。这样，在一个 connect 到达这个 lisent socket 的时候，内核会唤醒所有睡眠在 accept 队列上的 task。N 个 task 进程(线程)同时从 accept 返回，但是，只有一个 task 返回这个 connect 的 fd，其他 task 都返回-1(EAGAIN)。这是典型的 accept"惊群"现象。

这个是 linux 上困扰了大家很长时间的一个经典问题，在 linux 2.6 以后的内核，用户进程 task 对 listen socket 进行 accept 操作，如果这个时候如果没有新的 connect 请求过来，用户进程 task 会阻塞睡眠在 listent fd 的睡眠队列上。这个时候，用户进程 Task 会被设置 WQ_FLAG_EXCLUSIVE 标志位，并加入到 listen socket 的睡眠队列尾部(这里要确保所有不带 WQ_FLAG_EXCLUSIVE 标志位的 non-exclusive waiters 排在带 WQ_FLAG_EXCLUSIVE 标志位的 exclusive waiters 前面)。根据前面的唤醒逻辑，一个新的 connect 到来，内核只会唤醒一个用户进程 task 就会退出唤醒过程，从而不存在了"惊群"现象。

## select/poll/Epoll "惊群"现象

尽管 accept 系统调用已经不再存在"惊群"现象，但是我们的"惊群"场景还没结束。通常一个 server 有很多其他网络 IO 事件要处理，我们并不希望 server 阻塞在 accept 调用上，为提高服务器的并发处理能力，我们一般会使用 select/poll/epoll I/O 多路复用技术，同时为了充分利用多核 CPU，服务器上会起多个进程(线程)同时提供服务。于是，在某一时刻多个进程(线程)阻塞在 select/poll/epoll_wait 系统调用上，当一个请求上来的时候，多个进程都会被 select/poll/epoll_wait 唤醒去 accept，然而只有一个进程(线程 accept 成功，其他进程(线程 accept 失败，然后重新阻塞在 select/poll/epoll_wait 系统调用上。可见，尽管 accept 不存在"惊群"，但是我们还是没能摆脱"惊群"的命运。难道真的没办法了么？我只让一个进程去监听 listen socket 的可读事件，这样不就可以避免"惊群"了么？

没错，就是这个思路，我们来看看 Nginx 是怎么避免由于 listen fd 可读造成的 epoll_wait"惊群"。这里简单说下具体流程，不进行具体的源码分析。

Nginx 中有个标志 ngx_use_accept_mutex，当 ngx_use_accept_mutex 为 1 的时候（当 nginx worker 进程数>1 时且配置文件中打开 accept_mutex 时，这个标志置为 1），表示要进行 listen fdt"惊群"避免。

Nginx 的 worker 进程在进行 event 模块的初始化的时候，在 core event 模块的 process_init 函数中(ngx_event_process_init)将 listen fd 加入到 epoll 中并监听其 READ 事件。Nginx 在进行相关初始化完成后，进入事件循环(ngx_process_events_and_timers 函数)，在 ngx_process_events_and_timers 中判断，如果 ngx_use_accept_mutex 为 0，那就直接进入 ngx_process_events(ngx_epoll_process_events)，在 ngx_epoll_process_events 将调用 epoll_wait 等待相关事件到来或超时，epoll_wait 返回的时候该干嘛就干嘛。这里不讲 ngx_use_accept_mutex 为 0 的流程，下面讲下 ngx_use_accept_mutex 为 1 的流程。

- [1] 进入 ngx_trylock_accept_mutex，加锁抢夺 accept 权限（ngx_shmtx_trylock(&ngx_accept_mutex)），加锁成功，则调用 ngx_enable_accept_events(cycle) 来将一个或多个 listen fd 加入 epoll 监听 READ 事件(设置事件的回调函数 ngx_event_accept)，并设置 ngx_accept_mutex_held = 1;标识自己持有锁。
- [2] 如果 ngx_shmtx_trylock(&ngx_accept_mutex)失败，则调用 ngx_disable_accept_events(cycle, 0)来将 listen fd 从 epoll 中 delete 掉。
- [3] 如果 ngx_accept_mutex_held = 1(也就是抢到 accept 权)，则设置延迟处理事件标志位 flags |= NGX_POST_EVENTS; 如果 ngx_accept_mutex_held = 0(没抢到 accept 权)，则调整一下自己的 epoll_wait 超时，让自己下次能早点去抢夺 accept 权。
- [4] 进入 ngx_process_events(ngx_epoll_process_events)，在 ngx_epoll_process_events 将调用 epoll_wait 等待相关事件到来或超时。
- [5] epoll_wait 返回，循环遍历返回的事件，如果标志位 flags 被设置了 NGX_POST_EVENTS，则将事件挂载到相应的队列中(Nginx 有两个延迟处理队列，(1)ngx_posted_accept_events：listen fd 返回的事件被挂载到的队列。(2)ngx_posted_events：其他 socket fd 返回的事件挂载到的队列)，延迟处理事件，否则直接调用事件的回调函数。
- [6] ngx_epoll_process_events 返回后，则开始处理 ngx_posted_accept_events 队列上的事件，于是进入的回调函数是 ngx_event_accept，在 ngx_event_accept 中 accept 客户端的请求，进行一些初始化工作，将 accept 到的 socket fd 放入 epoll 中。
- [7] ngx_epoll_process_events 处理完成后，如果本进程持有 accept 锁 ngx_accept_mutex_held = 1，那么就将锁释放。
- [8] 接着开始处理 ngx_posted_events 队列上的事件

Nginx 通过一次仅允许一个进程将 listen fd 放入自己的 epoll 来监听其 READ 事件的方式来达到 listen fd"惊群"避免。然而做好这一点并不容易，作为一个高性能 web 服务器，需要尽量避免阻塞，并且要很好平衡各个工作 worker 的请求，避免饿死情况，下面有几个点需要大家留意：

- [1] 避免新请求不能及时得到处理的饿死现象 工作 worker 在抢夺到 accept 权限，加锁成功的时候，要将事件的处理 delay 到释放锁后在处理(为什么 ngx_posted_accept_events 队列上的事件处理不需要延迟呢？ 因为 ngx_posted_accept_events 上的事件就是 listen fd 的可读事件，本来就是我抢到的 accept 权限，我还没 accept 就释放锁，这个时候被别人抢走了怎么办呢？)。否则，获得锁的工作 worker 由于在处理一个耗时事件，这个时候大量请求过来，其他工作 worker 空闲，然而没有处理权限在干着急。
- [2] 避免总是某个 worker 进程抢到锁，大量请求被同一个进程抢到，而其他 worker 进程却很清闲。 Nginx 有个简单的负载均衡，ngx_accept_disabled 表示此时满负荷程度，没必要再处理新连接了，我们在 nginx.conf 曾经配置了每一个 nginx worker 进程能够处理的最大连接数，当达到最大数的 7/8 时，ngx_accept_disabled 为正，说明本 nginx worker 进程非常繁忙，将不再去处理新连接。每次要进行抢夺 accept 权限的时候，如果 ngx_accept_disabled 大于 0，则递减 1，不进行抢夺逻辑。

Nginx 采用在同一时刻仅允许一个 worker 进程监听 listen fd 的可读事件的方式，来避免 listen fd 的"惊群"现象。然而这种方式编程实现起来比较难，难道不能像 accept 一样解决 epoll 的"惊群"问题么？答案是可以的。要说明 epoll 的"惊群"问题以及解决方案，不能不从 epoll 的两种触发模式说起。

## Epoll"惊群"之 LT(水平触发模式)、ET(边沿触发模式)

epoll 作为中间层，为多个进程 task，监听多个 fd 的多个事件提供了一个便利的高效机制，我们来看下 epoll 的机制图：

![4](/images/linux_socket/4.png)

要说代码实现上，其实也比较简单，大致有以下的几个逻辑：

1. 创建 epoll 句柄，初始化相关数据结构
2. 为 epoll 句柄添加文件句柄，注册睡眠 entry 的回调
3. 事件发生，唤醒相关文件句柄睡眠队列的 entry，调用其回调
4. 唤醒 epoll 睡眠队列的 task，搜集并上报数据

图中的一个 epoll 的 sleep/wakeup 流程如下：

无事件的时候，多个进程 task 调用 epoll_wait 睡眠在 epoll 的 wq 睡眠队列上。

- [1] 这个时候一个请求 RQ_1 上来，listen fd 这个时候 ready 了，开始唤醒其睡眠队列上的 epoll entry，并执行之前 epoll 注册的回调函数 ep_poll_callback。
- [2] ep_poll_callback 主要做两件事情，(1)发生的 event 事件是 epoll entry 关心的，则将 epi 挂载到 epoll 的就绪队列 ready list 并进入(2)，否则结束。(2)如果当前 wq 不为空，则唤醒睡眠在 epoll 等待队列上睡眠的 task(这里唤醒一个还是多个，是区分 epoll 的 ET 模式还是 LT 模式，下面在细讲)。
- [3] epoll_wait 被唤醒继续前行，在 ep_poll 中调用 ep_send_events 将 fd 相关的 event 事件和数据 copy 到用户空间，这个时候就需要遍历 epoll 的 ready list 以便收集 task 需要监控的多个 fd 的 event 事件和数据上报给用户进程 task，这个在 ep_scan_ready_list 中完成，这里会将 ready list 清空。

通过上图的 epoll 事件通知机制，epoll 的 LT 模式、ET 模式在事件通知行为上的差别，也只能是在[2]上 task 唤醒逻辑上的差别了。我们先来看下，在 epoll_wait 中调用的导致用户进程 task 睡眠的 ep_poll 函数的核心逻辑：

```
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
{
int res = 0, eavail, timed_out = 0;
bool waiter = false;
...
eavail = ep_events_available(ep);//是否有fd就绪
if (eavail)
goto send_events;//有fd就绪，则直接跳过去上报事件给用户

if (!waiter) {
          waiter = true;
          init_waitqueue_entry(&wait, current);//为当前进程task构造一个睡眠entry

          spin_lock_irq(&ep->wq.lock);
          //插入到epoll的wq后面，注意这里是排他插入的，就是带WQ_FLAG_EXCLUSIVE flag
          __add_wait_queue_exclusive(&ep->wq, &wait);
          spin_unlock_irq(&ep->wq.lock);
    }

for (;;) {
  //将当前进程设置位睡眠, 但是可以被信号唤醒的状态, 注意这个设置是"将来时", 我们此刻还没睡
  set_current_state(TASK_INTERRUPTIBLE);
  // 检查是否真的要睡了
  if (fatal_signal_pending(current)) {
              res = -EINTR;
              break;
          }

          eavail = ep_events_available(ep);
          if (eavail)
              break;
          if (signal_pending(current)) {
              res = -EINTR;
              break;
          }
          // 检查是否真的要睡了 end

  //使得当前进程休眠指定的时间范围，
  if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS)) {
  timed_out = 1;
  break;
  }
}

__set_current_state(TASK_RUNNING);

send_events:
      /*
       * Try to transfer events to user space. In case we get 0 events and
       * there's still timeout left over, we go trying again in search of
       * more luck.
       */
      // ep_send_events往用户态上报事件，即那些epoll_wait返回后能获取的事件
      if (!res && eavail &&
          !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
          goto fetch_events;

      if (waiter) {
          spin_lock_irq(&ep->wq.lock);
          __remove_wait_queue(&ep->wq, &wait);
          spin_unlock_irq(&ep->wq.lock);
      }

   return res;
}

```

接着，我们看下监控的 fd 有事件发生的回调函数 ep_poll_callback 的核心逻辑：

```
#define wake_up(x)__wake_up(x, TASK_NORMAL, 1, NULL)

static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
      int pwake = 0;
      struct epitem *epi = ep_item_from_wait(wait);
      struct eventpoll *ep = epi->ep;
      __poll_t pollflags = key_to_poll(key);
      unsigned long flags;
      int ewake = 0;
  ....

  //判断是否有我们关心的event
      if (pollflags && !(pollflags & epi->event.events))
          goto out_unlock;
  //将当前的epitem放入epoll的ready list
      if (!ep_is_linked(epi) &&
          list_add_tail_lockless(&epi->rdllink, &ep->rdllist)) {
          ep_pm_stay_awake_rcu(epi);
      }
      //如果有task睡眠在epoll的等待队列，唤醒它
  if (waitqueue_active(&ep->wq)) {
  ....
  wake_up(&ep->wq);//
  }
  ....
}

```

wake_up 函数最终会调用到 wake_up_common，通过前面的 wake_up_common 我们知道，唤醒过程在唤醒一个带 WQ_FLAG_EXCLUSIVE 标记的 task 后，即退出唤醒过程。通过上面的 ep_poll，task 是排他(带 WQ_FLAG_EXCLUSIVE 标记)加入到 epoll 的等待队列 wq 的。也就是，在 ep_poll_callback 回调中，只会唤醒一个 task。这就有问题，根据 LT 的语义：只要仍然有未处理的事件，epoll 就会通知你。例如有两个进程 A、B 睡眠在 epoll 的睡眠队列，fd 的可读事件到来唤醒进程 A，但是 A 可能很久才会去处理 fd 的事件，或者它根本就不去处理。根据 LT 的语义，应该要唤醒进程 B 的。

我们来看下 epoll 怎么在 ep_send_events 中实现满足 LT 语义的：

```
// 将就绪的events传递到用户空间
static int ep_send_events(struct eventpoll *ep, struct epoll_event __user *events, int maxevents)
  {
      struct ep_send_events_data esed;

      esed.maxevents = maxevents;
      esed.events = events;

      ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
      return esed.res;
  }
// 扫描就绪链表
  static __poll_t ep_scan_ready_list(struct eventpoll *ep, __poll_t (*sproc)(struct eventpoll *, struct list_head *, void *), void *priv, int depth, bool ep_locked)
  {
  ...
  // 所有的epitem都转移到了txlist上, 而rdllist被清空了
      list_splice_init(&ep->rdllist, &txlist);
      ...
      //sproc 就是 ep_send_events_proc
      res = (*sproc)(ep, &txlist, priv);
      ...
      //没有处理完的epitem, 重新插入到ready list
      list_splice(&txlist, &ep->rdllist);

      /*
     * 注意到epoll_wait()中，将wait_queue_t的等待状态设置为互斥等待，因此
     * 每次被唤醒的只有一个节点。现在我们已经将eventpoll中就绪队列里的事件
     * 尽量向用户交付了，但是在交付时，可能没有交付完全(1.交付过程中出现了
     * 错误 2.使用了LT模式），也有可能在过程中又发生了新的事件。也就是这次
     * epoll_wait()调用后，还剩下一些就绪资源，那么我们再次唤醒一个等待节点
     * 让别的用户也享用一下资源
     *
     * 从这里已经可以看出内核对于epoll惊群的解决方案：ET模式：
     * 1. 每次只唤醒一个节点
     * 2. 事件交付后不再将事件重新挂载到就绪队列
     */
     if (!list_empty(&ep->rdllist)) {
          if (waitqueue_active(&ep->wq))
              wake_up(&ep->wq);
  ...
      }
  }

    static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head, void *priv)
  {
    ...
    //遍历就绪fd列表
        list_for_each_entry_safe(epi, tmp, head, rdllink) {
        ...
        //然后从链表里面移除当前就绪的epi
        list_del_init(&epi->rdllink);
        //读取当前epi的事件
            revents = ep_item_poll(epi, &pt, 1);
            if (!revents)
              continue;
        //将当前的事件和用户传入的数据都copy给用户空间
            if (__put_user(revents, &uevent->events) ||
              __put_user(epi->event.data, &uevent->data)) {
              //如果发生错误了， 则终止遍历过程，将当前epi重新返回就绪队列，剩下的也会在ep_scan_ready_list中重新放回就绪队列
              list_add(&epi->rdllink, head);
              ep_pm_stay_awake(epi);
              if (!esed->res)
                  esed->res = -EFAULT;
              return 0;
           }
        }
          if (epi->event.events & EPOLLONESHOT)
              epi->event.events &= EP_PRIVATE_BITS;
          else if (!(epi->event.events & EPOLLET)) { // 保证(1)
          //如果是非ET模式(即LT模式)，当前epi会被重新放到epoll的ready list。
              list_add_tail(&epi->rdllink, &ep->rdllist);
              ep_pm_stay_awake(epi);
          }
  }

```

处理逻辑的核心流程就 2 点：

- [1] 遍历并清空 epoll 的 ready list，遍历过程中，对于每个 epi 收集其返回的 events，如果没收集到 event，则 continue 去处理其他 epi，否则将当前 epi 的事件和用户传入的数据都 copy 给用户空间，并判断，如果是在 LT 模式下，则将当前 epi 重新放回 epoll 的 ready list
- [2] 遍历 epoll 的 ready list 完成后，如果 ready list 不为空，则继续唤醒 epoll 睡眠队列 wq 上的其他 task B。task B 从 epoll_wait 醒来继续前行，重复上面的流程，继续唤醒 wq 上的其他 task C，这样链式唤醒下去

通过上面的流程，在一个 epoll 上睡眠的多个 task，如果在一个 LT 模式下的 fd 的事件上来，会唤醒 epoll 睡眠队列上的所有 task，而 ET 模式下，仅仅唤醒一个 task，这是 epoll"惊群"的根源。等等，这样在 LT 模式下就必然"惊群"，epoll 在 LT 模式下的"惊群"没办法解决么？

## epoll_create& fork

相信大家在多进程服务中使用 epoll 的时候，都会有这样一个疑问，是先 epoll_create 得到 epoll fd 后在 fork 子进程，还是先 fork 子进程，然后每个子进程在 epoll_create 自己独立的 epoll fd 呢？有什么异同？

### 先 epoll_create 后 fork

这样，多个进程公用一个 epoll 实例(父子进程的 epoll fd 指向同一个内核 epoll 对象)，上面介绍的 epoll 核心机制流程，都是在同一个 epoll 对象上的，这种情况下，epoll 有以下这些特性：

- [1] epoll 在 ET 模式下不存在“惊群”现象，LT 模式是 epoll“惊群”的根源，并且 LT 模式下的“惊群”没办法避免。
- [2] LT 的“惊群”是链式唤醒的，唤醒过程直到当前 epi 的事件被处理了，无法获得到新的事件才会终止唤醒过程。 例如有 A、B、C、D...等多个进程 task 睡眠在 epoll 的睡眠队列上，并且都监控同一个 listen fd 的可读事件。一个请求上来，会首先唤醒 A 进程，A 在 epoll_wait 的处理过程中会唤醒进程 B，这样进程 B 在 epoll_wait 的处理过程中会唤醒 C，这个时候 A 的 epoll_wait 处理完成返回，进程 A 调用 accept 读取了当前这个请求，进程 C 在自己的 epoll_wait 处理过程中，从 epi 中获取不到事件了，于是终止了整个链式唤醒过程。
- [3] 多个进程的 epoll fd 由于指向同一个 epoll 内核对象，他们对 epoll fd 的相关 epoll_ctl 操作会相互影响。一不小心可能会出现一些比较诡异的行为。 想象这样一个场景(实际上应该不是这样用)，有一个服务在 1234，1235，1236 这 3 个端口上提供服务，于是它 epoll_create 得到 epoll fd 后，fork 出 3 个工作的子进程 A、B、C，它们分别在这 3 个端口创建 listen fd，然后加入到 epoll 中监听其可读事件。这个时候端口 1234 上来一个请求，A、B、C 同时被唤醒，A 在 epoll_wait 返回后，在进行 accept 前由于种种原因卡住了，没能及时 accept。B、C 在 epoll_wait 返回后去 accept 又不能 accept 到请求，这样 B、C 重新回到 epoll_wait，这个时候又被唤醒，这样只要 A 没有去处理这个请求之前，B、C 就一直被唤醒，然而 B、C 又无法处理该请求。
- [4] ET 模式下，一个 fd 上的同事多个事件上来，只会唤醒一个睡眠在 epoll 上的 task，如果该 task 没有处理完这些事件，在没有新的事件上来前，epoll 不会在通知 task 去处理

由于 ET 的事件通知模式，通常在 ET 模式下的 epoll_wait 返回，我们会循环 accept 来处理所有未处理的请求，直到 accept 返回 EAGAIN 才退出 accept 流程。否则，没处理遗留下来的请求，这个时候如果没有新的请求过来触发 epoll_wait 返回，这样遗留下来的请求就得不到及时处理。这种处理模式，会带来一种类"惊群"现象。考虑，下面的一个处理过程：

A、B、C 三个进程在监听 listen fd 的 EPOLLIN 事件，都睡眠在 epoll_wait 上，都是 ET 模式。

- [1] listen fd 上一个请求 C_1 上来，该请求唤醒了 A 进程，A 进程从 epoll_wait 返回准备去 accept 该请求来处理。
- [2] 这个时候，第二个请求 C_2 上来，由于睡眠队列上是 B、C，于是 epoll 唤醒 B 进程，B 进程从 epoll_wait 返回准备去 accept 该请求来处理。
- [3] A 进程在自己的 accept 循环中，首选 accept 得到 C_1，接着 A 进程在第二个循环继续 accept，继续得到 C_2。
- [4] B 进程在自己的 accept 循环中，调用 accept，由于 C_2 已经被 A 拿走了，于是 B 进程 accept 返回 EAGAIN 错误，于是 B 进程退出 accept 流程重新睡眠在 epoll_wait 上。
- [5] A 进程继续第三个循环，这个时候已经没有请求了， accept 返回 EAGAIN 错误，于是 A 进程也退出 accept 处理流程，进入请求的处理流程。

可以看到，B 进程被唤醒了，但是并没有事情可以做，同时，epoll 的 ET 这样的处理模式，负载容易出现不均衡。

### 先 fork 后 epoll_create

用法上，通常是在父进程创建了 listen fd 后，fork 多个 worker 子进程来共同处理同一个 listen fd 上的请求。这个时候，A、B、C...等多个子进程分别创建自己独立的 epoll fd，然后将同一个 listen fd 加入到 epoll 中，监听其可读事件。这种情况下，epoll 有以下这些特性：

- [1] 由于相对同一个 listen fd 而言， 多个进程之间的 epoll 是平等的，于是，listen fd 上的一个请求上来，会唤醒所有睡眠在 listen fd 睡眠队列上的 epoll，epoll 又唤醒对应的进程 task，从而唤醒所有的进程(这里不管 listen fd 是以 LT 还是 ET 模式加入到 epoll)。
- [2] 多个进程间的 epoll 是独立的，对 epoll fd 的相关 epoll_ctl 操作相互独立不影响

可以看出，在使用友好度方面，多进程独立 epoll 实例要比共用 epoll 实例的模式要好很多。独立 epoll 模式要解决 fd 的排他唤醒 epoll 即可。

## EPOLLEXCLUSIVE 排他唤醒 Epoll

linux4.5 以后的内核版本中，增加了 EPOLLEXCLUSIVE， 该选项只能通过 EPOLL_CTL_ADD 对需要监控的 fd(例如 listen fd)设置 EPOLLEXCLUSIVE 标记。这样 epoll entry 是通过排他方式挂载到 listen fd 等待队列的尾部的，睡眠在 listen fd 的等待队列上的 epoll entry 会加上 WQ_FLAG_EXCLUSIVE 标记。根据前面介绍的内核 wake up 机制，listen fd 上的事件上来，在遍历并唤醒等待队列上的 entry 的时候，遇到并唤醒第一个带 WQ_FLAG_EXCLUSIVE 标记的 entry 后，就结束遍历唤醒过程。于是，多进程独立 epoll 的"惊群"问题得到解决。

是否可以用 WQ_FLAG_EXCLUSIVE 标志进行 epoll 惊群避免呢？

先分析一种场景，用图示表示：

![5](/images/linux_socket/5.png)

task1 进程在 epoll_create 等操作后，fork 出子进程 task2，所以父进程和子进程共享 struct epoll 数据结构，共用 ep->wq 等待队列。接下来两个子进程分别处理完 accept 后，task1 建立连接 new_fd1，task2 建立新连接 new_fd2。此时这两个新连接不共享，分别属于各自的进程。

当 new_fd1 上有数据可读时，对于 epoll->wq 等待队列来说，它只知道有事件发生，并不知道这个事件属于 task1 还是 task2，唤醒 ep->wq 等待队列时，所有的任务都被唤醒，让用户自己去判断连接事件属于哪个进程。

所以，不能用 WQ_FLAG_EXCLUSIVE 标志解决惊群问题。

## "惊群"之 SO_REUSEPORT

"惊群"浪费资源的本质在于很多处理进程在别惊醒后，发现根本无事可做，造成白白被唤醒，做了无用功。但是，简单的避免"惊群"会造成同时并发上来的请求得不到及时处理(降低了效率)，为了避免这种情况，NGINX 允许配置成获得 Accept 权限的进程一次性循环 Accept 所有同时到达的全部请求，但是，这会造成短时间 worker 进程的负载不均衡。为此，我们希望的是均衡唤醒，也就是，假设有 4 个 worker 进程睡眠在 epoll_wait 上，那么此时同时并发过来 3 个请求，我们希望 3 个 worker 进程被唤醒去处理，而不是仅仅唤醒一个进程或全部唤醒。

然而要实现这样不是件容易的事情，其根本原因在于，对于大多采用 MPM 机制(multi processing module)TCP 服务而言，基本上都是多个进程或者线程同时在一个 Listen socket 上进行监听请求。根据前面介绍的 Linux 睡眠队列的唤醒方式，基本睡眠在这个 listen socket 上的 Task 只能要么全部被唤醒，要么被唤醒一个。

于是，基本的解决方案是起多个 listen socket，好在我们有 SO_REUSEPORT(linux 3.9 以上内核支持)，它支持多个进程或线程 bind 相同的 ip 和端口，支持以下特性

- [1] 允许多个 socket bind/listen 在相同的 IP，相同的 TCP/UDP 端口
- [2] 目的是同一个 IP、PORT 的请求在多个 listen socket 间负载均衡
- [3] 安全上，监听相同 IP、PORT 的 socket 只能位于同一个用户下

于是，在一个多核 CPU 的服务器上，我们通过 SO_REUSEPORT 来创建多个监听相同 IP、PORT 的 listen socket，每个进程监听不同的 listen socket。这样，在只有 1 个新请求到达监听的端口的时候，内核只会唤醒一个进程去 accept，而在同时并发多个请求来到的时候，内核会唤醒多个进程去 accept，并且在一定程度上保证唤醒的均衡性。SO_REUSEPORT 在一定程度上解决了"惊群"问题，但是，由于 SO_REUSEPORT 根据数据包的四元组和当前服务器上绑定同一个 IP、PORT 的 listen socket 数量，根据固定的 hash 算法来路由数据包的，其存在如下问题：

- [1] Listen Socket 数量发生变化的时候，会造成握手数据包的前一个数据包路由到 A listen socket，而后一个握手数据包路由到 B listen socket，这样会造成 client 的连接请求失败。
- [2] 短时间内各个 listen socket 间的负载不均衡

## 惊不"惊群"其实是个问题

很多时候，我们并不是害怕"惊群"，我们怕的"惊群"之后，做了很多无用功。相反在一个异常繁忙，并发请求很多的服务器上，为了能够及时处理到来的请求，我们希望能有多"惊群"就多"惊群"，因为根本做不了无用功，请求多到都来不及处理。于是出现下面的情形：

![6](/images/linux_socket/6.png)

![7](/images/linux_socket/7.png)

![8](/images/linux_socket/8.png)

从上可以看到各个 CPU 都很忙，但是实际有用的 CPU 时间却很少，大部分的 CPU 消耗在\_spin_lock 自旋锁上了，并且服务器并发吞吐量并没有随着 CPU 核数增加呈现线性增长，相反出现下降的情况。这是为什么呢？怎么解决？

### 问题原因

我们知道，一般一个 TCP 服务只有一个 listen socket、一个 accept 队列，而一个 TCP 服务一般有多个服务进程(一个核一个)来处理请求。于是并发请求到达 listen socket 处，那么多个服务进程势必存在竞争，竞争一存在，那么就需要用排队来解决竞态问题，于是似乎锁就无法避免了。在这里，有两类竞争主体，一类是内核协议栈(不可睡眠类)、一类是用户进程(可睡眠类)，这两类主体对 listen socket 发生三种类型的竞争

- [1] 协议栈内部之间的竞争
- [2] 用户进程内部之间的竞争
- [3] 协议栈和用户之间的竞争

由于内核协议栈是不可睡眠的，为此 linux 中采用两层锁定的 lock 结构，一把 listen_socket.lock 自旋锁，一把 listen_socket.own 排他标记锁。其中，listen_socket.lock 用于协议栈内部之间的竞争、协议栈和用户之间的竞争，而 listen_socket.own 用于用户进程内部之间的竞争，listen_socket.lock 作为 listen_socket.own 的排他保护(要获取 listen_socket.own 首先要获取到 listen_socket.lock)。

对于处理 TCP 请求而言，一个 SYN 包 syn_skb 到来，这个时候内核 Lock(RCU 锁)住全局的 listeners Table，查找 syn_skb 对应的 listen_socket，没找到则返回错误。否则，就需要进入三次握手处理，首先内核协议栈需要自旋获得 listen_socket.lock 锁，初始化一些数据结构，回复 syn_ack，然后释放 listen_socket.lock 锁。

接着，client 端的 ack 包到来，协议栈这个时候，需要自旋获得 listen_socket.lock 锁，构造 client 端的 socket 等数据结构，如果 accept 队列没有被用户进程占用，那么就将连接排入 accept 队列等待用户进程来 accept，否则就排入 backlog 队列(职责转移，连接排入 accept 队列的事情交给占有 accept 队列的用户进程)。可见，处理一个请求，协议栈需要竞争两次 listen_socket 的自旋锁。由于内核协议栈不能睡眠，于是它只能自旋不断地去尝试获取 listen_socket.lock 自旋锁，直到获取到自旋锁成功为止，中间不能停下来。自旋锁这种暴力、打架的抢锁方式，在一个高并发请求到来的服务器上，就有可能出现上面这种 80%多的 CPU 时间被内核占用，应用程序只能够分配到较少的 CPU 时钟周期的资源的情况。

### 问题的解决

解决这个问题无非两个方向：(1) 多队列化，减少竞争者 (2) listen_socket 无锁化。

#### 多队列化 - SO_REUSEPORT

通过上面的介绍，在 Linux kernel 3.9 以上，可以通过 SO_REUSEPORT 来创建多个 bind 相同 IP、PORT 的 listen_socket。我们可以每一个 CPU 核创建一个 listen_socket 来监听处理请求，这样就是每个 CPU 一个处理进程、一个 listen_socket、一个 accept 队列，多个进程同时并发处理请求，进程之间不再相互竞争 listen_socket。SO_REUSEPORT 可以做到多个 listen_socket 间的负载均衡，然而其负载均衡效果是取决于 hash 算法，可能会出现短时间内的负载极端不均衡。

SO_REUSEPORT 是在将一对多的问题变成多对多的问题，将 Listen Socket 无序暴力争抢 CPU 的现状变成更为有序的争抢。多队列化的优化必须要面对和解决的四个问题是：队列比 CPU 多，队列与 CPU 相等，队列比 CPU 少，根本就没有队列，于是，他们要解决队列发生变化的情况。

如果仅仅把 TCP 的 Listener 看作一个被协议栈处理的 Socket，它和 Client Socket 一起都在相互拼命抢 CPU 资源，那么就可能出现上面的，短时间大量并发请求过来的时候，大量的 CPU 时间被消耗在自旋锁的争抢上了。我们可以换个角度，如果把 TCP Listener 看作一个基础设施服务呢？Listener 为新来的连接请求提供连接服务，并产生 Client Socket 给用户进程，它可以通过一个或多个两类 Accept 队列提供一个服务窗口给用户进程来 accept Client Socket 来处理。仅仅在 Client Socket 需要排入 Accept 队列的是，细粒度锁住队列即可，多个有多个 Accept 队列(每 CPU 一个，那么连锁队列的操作都可以省了)。这样 Listener 就与用户进程无关了，用户进程的产生、退出、CPU 间跳跃、绑定，解除绑定等等都不会影响 TCP Listener 基础设施服务，受影响的是仅仅他们自己该从那个 Accept 队列获取 Client Socket 来处理。

于是一个解决思路是连接处理无锁化。

#### listen socket 无锁化- 旁门左道之 SYN Cookie

SYN Cookie 原理由 D.J. Bernstain 和 Eric Schenk 提出，专门用来防范 SYN Flood 攻击的一种手段。它的原理是，在 TCP 服务器接收到 SYN 包并返回 SYN ACK 包时，不分配一个专门的数据结构(避免浪费服务器资源)，而是根据这个 SYN 包计算出一个 cookie 值。这个 cookie 作为 SYN ACK 包的初始序列号。当客户端返回一个 ACK 包时，根据包头信息计算 cookie，与返回的确认序列号(初始序列号 + 1)进行对比，如果相同，则是一个正常连接，然后，分配资源，创建 Client Socket 排入 Accept 队列等等用户进程取出处理。于是，整个 TCP 连接处理过程实现了无状态的三次握手。SYN Cookie 机制实现了一定程度上的 listen socket 无锁化，但是它有以下几个缺点：

- （1）丢失 TCP 选项信息   在建立连接的过程中，不在服务器端保存任何信息，它会丢失很多选项协商信息，这些信息对 TCP 的性能至关重要，比如超时重传等。但是，如果使用时间戳选项，则会把 TCP 选项信息保存在 SYN ACK 段中 tsval 的低 6 位。
- （2）cookie 不能随地开启  Linux 采用动态资源分配机制，当分配了一定的资源后再采用 cookie 技术。同时为了避免另一种拒绝服务攻击方式，攻击者发送大量的 ACK 报文，服务器忙于计算验证 SYN Cookie。服务器对收到的 ACK 进行 Cookie 合法性验证前，需要确定最近确实发生了半连接队列溢出，不然攻击者只要随便发送一些 ACK，服务器便要忙于计算了。

#### listen socket 无锁化- Linux 4.4 内核给出的 Lockless TCP listener

SYN cookie 给出了 Lockless TCP listener 的一些思路，但是我们不想是无状态的三次握手，又不想请求的处理和 Listener 强相关，避免每次进行握手处理都需要 lock 住 listen socket，带来性能瓶颈。4.4 内核前的握手处理是以 listen socket 为主体，listen socket 管理着所有属于它的请求，于是进行三次握手的每个数据包的处理都需要操作这个 listener 本身，而一般情况下，一个 TCP 服务器只有一个 listener，于是在多核环境下，就需要加锁 listen socket 来安全处理握手过程了。我们可以换个角度，握手的处理不再以 listen socket 为主体，而是以连接本身为主体，需要记住的是该连接所属的 listen socket 即可。4.4 内核握手处理流程如下：

- [1] TCP 数据包 skb 到达本机，内核协议栈从全局 socket 表中查找 skb 的目的 socket(sk)，如果是 SYN 包，当然查找到的是 listen_socket 了，于是，协议栈根据 skb 构造出一个新的 socket(tmp_sk)，并将 tmp_sk 的 listener 标记为 listen_socket，并将 tmp_sk 的状态设置为 SYNRECV，同时将构造好的 tmp_sk 排入全局 socket 表中，并回复 syn_ack 给 client。
- [2] 如果到达本机的 skb 是 syn_ack 的 ack 数据包，那么查找到的将是 tmp_sk，并且 tmp_sk 的 state 是 SYNRECV，于是内核知道该数据包 skb 是 syn_ack 的 ack 包了，于是在 new_sk 中拿出连接所属的 listen_socket，并且根据 tmp_sk 和到来的 skb 构造出 client_socket，然后将 tmp_sk 从全局 socket 表中删除(它的使命结束了)，最后根据所属的 listen_socket 将 client_socket 排如 listen_socket 的 accept 队列中，整个握手过程结束。

  4.4 内核一改之前的以 listener 为主体，listener 管理所有 request 的方式，在 SYN 包到来的时候，进行控制反转，以 Request 为主体，构造出一个临时的 tmp_sk 并标记好其所属的 listener，然后平行插入到所有 socket 公共的 socket 哈希表中，从而解放掉 listener，实现 Lockless TCP listener。

## 其他相关资料

1. [SO_REUSEPORT 学习笔记](http://www.blogjava.net/yongboy/archive/2015/02/12/422893.html)
