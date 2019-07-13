---
layout:     post
title:      记一次GO语言内存泄漏——pprof heap分析
category: golang
tags: [golang]
description: 最近用GO写了一个类似于数据采集汇总的程序，但是却发现了内存泄漏的情况
---


## 发现过程

最近用GO写了一个类似于数据采集汇总的程序，但是却发现了内存泄漏的情况。整个服务本身主要是监听网络端口并收集上报数据，常驻在后台服务运行，单机 QPS 峰值目前约为500-600。发现问题初期初步猜测是有变量一直在被引用导致没有被GC成功。

程序运行一段时间后发现机器可用内存几乎为零，查看内存占用情况，数据采集程序占用了接近90%的内存。查看机器情况，以下贴上机器的内存图。

![内存图](/images/pprof_heap/mem.png)

可以看到，内存几乎是成线性的状态一直在上涨中，期间恢复是因为手动重启过程序所以占用内存断崖式地减少了。断定是程序存在内存泄漏的情况。

## 排查过程

Go官方提供的工具非常多样丰富，其中一个名为 pprof 可以非常有效地分析我们程序中的CPU、内存、Goroutine等资源的使用情况。我们这次主要使用 pprof 提供的 heap 分析功能。

我们先在代码中加入运行时数据采集的相关代码，可以实时查询程序的运行情况，对代码的入侵性也不大。

对于 非 longrunning 程序，我们可以使用  pprof.WriteHeapProfile(file)  导出 heap 情况。

对于 longruning 的程序，我们可以使用 net/http/pprof 开启一个web服务，实时导出程序运行情况。

我们这里使用 net/http/pprof 对程序运行情况的监测。

```
import (
    "net/http"
    _ "net/http/pprof"
)

func main(){
    // 在程序入口处加入web端口的监听代码
    go func() {
        netHttp.ListenAndServe("127.0.0.1:6060", nil)
    }()

    ......
}
```

 在程序的入口处加入以上代码，可以通过访问 http://127.0.0.1:6060/debug/pprof/heap 导出我们的heap 使用情况

浏览器访问 http://127.0.0.1:6060/debug/pprof/heap?debug=1 可以查看具体的堆信息

或者  wget http://127.0.0.1:6060/debug/pprof/heap  下载heap文件，利用 pprof 更简单地分析堆内容。

先看一看命令行下 pprof 的使用说明

```
go tool pprof <format> [options] [binary] <source>
```

 这里也有一些比较有用的选项，方便我们进一步地分析程序

```
-inuse_space           Same as -sample_index=inuse_space
-inuse_objects         Same as -sample_index=inuse_objects
-alloc_space           Same as -sample_index=alloc_space
-alloc_objects         Same as -sample_index=alloc_objects
```

这里我们利用刚刚导出的 heap 文件，使用 -insue_space 选项分析正在使用的内存情况。

```
go tool pprof -inuse_space jdy_service_bin http://127.0.0.1:6060/debug/pprof/heap

(pprof) top 20
Showing nodes accounting for 7170.21kB, 100% of 7170.21kB total
Showing top 20 nodes out of 31
      flat  flat%   sum%        cum   cum%
 1536.21kB 21.42% 21.42%  4096.38kB 57.13%  github.com/cihub/seelog.NewAsyncLoopLogger
 1536.12kB 21.42% 42.85%  2560.16kB 35.71%  github.com/cihub/seelog.newAsyncLogger
 1024.38kB 14.29% 57.14%  1024.38kB 14.29%  runtime.malg
 1024.05kB 14.28% 71.42%  1024.05kB 14.28%  github.com/cihub/seelog.newCommonLogger
  513.31kB  7.16% 78.58%   513.31kB  7.16%  regexp.onePassCopy
  512.08kB  7.14% 85.72%  1025.39kB 14.30%  regexp.compile
  512.05kB  7.14% 92.86%   512.05kB  7.14%  sync.runtime_notifyListWait
  512.02kB  7.14%   100%   512.02kB  7.14%  github.com/satori/go%2euuid.UUID.String
         0     0%   100%  1025.39kB 14.30%  github.com/asaskevich/govalidator.init
         0     0%   100%  4096.38kB 57.13%  github.com/cihub/seelog.CloneLogger
         0     0%   100%  1025.39kB 14.30%  main.init
         0     0%   100%  4608.40kB 64.27%  net/http.(*ServeMux).ServeHTTP
         0     0%   100%  5120.45kB 71.41%  net/http.(*conn).serve
         0     0%   100%   512.05kB  7.14%  net/http.(*connReader).abortPendingRead
         0     0%   100%   512.05kB  7.14%  net/http.(*response).finishRequest
         0     0%   100%  4608.40kB 64.27%  net/http.HandlerFunc.ServeHTTP
         0     0%   100%  4608.40kB 64.27%  net/http.serverHandler.ServeHTTP
         0     0%   100%  4608.40kB 64.27%  qcloud.com/jdy_proj/jdy_service/vendor/qcloud.com/wellgo.(*Http).(qcloud.com/jdy_proj/jdy_service/vendor/qcloud.com/wellgo.httpHandler)-fm
         0     0%   100%  4608.40kB 64.27%  qcloud.com/jdy_proj/jdy_service/vendor/qcloud.com/wellgo.(*Http).httpHandler
         0     0%   100%  4096.38kB 57.13%  qcloud.com/jdy_proj/jdy_service/vendor/qcloud.com/wellgo.NewLogger
```

同时也可以导出调用关系图，详细看看整体的调用过程。

```
go tool pprof -png -output heap.png -inuse_space jdy_service_bin http://127.0.0.1:6060/debug/pprof/heap
```

![pprof](/images/pprof_heap/pprof.png)

从图中分析， Logger 相关的代码占用的内存都是比较大的，并且可能存在没有及时释放的问题。

这里 Logger 的内存占用占了大头，初步可以定位到日志模块出问题了。

对代码经过一轮分析后，猜测原因是：每一次 HTTP 请求会新建一个 Logger 实例 用于处理请求生命周期中的日志记录，但是在使用完成后没有主动关闭关闭实例。

```
.....

ctx.Logger, err = NewLogger()
defer ctx.Logger.Close()  // clone logger 后关闭logger

.....
```

 修改代码后重新跟踪定位，查看 prrof 调用关系图

```
go tool pprof -png -output heap.png -inuse_space jdy_service_bin http://127.0.0.1:6060/debug/pprof/heap
 ```

![pprof](/images/pprof_heap/pprof2.png)

可以看到这里 Logger 不再占用大量内存，继续观察内存情况。

![内存](/images/pprof_heap/mem2.png)

可以看到应用使用内存稳定在750M左右，没有再出现内存线性上涨的情况，运行过程中内存使用量保持平稳。问题得以解决

## 问题复盘

由于程序每个请求需要有一个独立的上下文信息，所以没有采取全局的单例 Logger 而改为每一个请求创建一个 Logger 实例以保存独立的上下文信息，并在日志中体现相关上下文数据。

具体流程是

```
Request-> Init Context -> NewLogger().SetContext(context) -> ....Logic.... -> Response
```

因此会创建出大量的 Logger 实例，如果没有及时关闭则不会被系统 GC 回收，占用大量内存并且消耗一定的 CPU 资源。

这里具体来看一下整个 NewLogger 的流程是怎么样的，为什么会没有被 GC 回收。

```
func NewLogger() (seelog.LoggerInterface, error) {
        // 自己封装的一个方法，调用 seelog.CloneLogger 复制一个系统全局单例的 Logger 并返回
	return seelog.CloneLogger(logger)
}
func CloneLogger(logger LoggerInterface) (LoggerInterface, error) {
	switch logger := logger.(type) {
	default:
		return nil, fmt.Errorf("unexpected type %T", logger)
        ........
	
	case *asyncTimerLogger:
		clone, err := NewAsyncTimerLogger(logger.commonLogger.config, logger.interval)
		if err != nil {
			return nil, err
		}
		return clone, nil
	
        ........
}
// NewAsyncLoopLogger creates a new asynchronous loop logger
func NewAsyncTimerLogger(config *logConfig, interval time.Duration) (*asyncTimerLogger, error) {
        ..........
	// 创建新的 AsyncTimerLogger 时会开启一个 goroutine
	go asnTimerLogger.processQueue()
	
        .......
}
func (asnTimerLogger *asyncTimerLogger) processQueue() {
	for !asnTimerLogger.Closed() {
                // 如果 Logger 没有被关闭，则循环调用自身的 processItem() 方法
		closed := asnTimerLogger.processItem()

		if closed {
			break
		}

		<-time.After(asnTimerLogger.interval)
	}
}
```

Logger 用的是第三方库 seelog。从代码分析，在新建一个 Logger 时，会产生一个 goroutine 定时轮询日志队列并处理日志，只要 Logger 状态没有被设置为 closed 则循环会一直进行下去。所以每次 Logger 用完需要回收时，需要主动关闭Logger。这样就不会再有泄漏的问题发生，GC也能够及时有效地进行。


**参考**

1. [https://golang.org/pkg/net/http/pprof/](https://golang.org/pkg/net/http/pprof/)
2. [https://studygolang.com/articles/4539](https://studygolang.com/articles/4539)
3. [https://juejin.im/entry/5ac9cf3a518825556534c76e](https://juejin.im/entry/5ac9cf3a518825556534c76e)
