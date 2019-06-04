---
layout:     post
title:      golang net/http标准库学习
category: golang
tags: [golang]
description: 学习golang有一年多时间了，平时也有很多业务场景会应用到http协议请求服务，但对于http标准库的具体实现不甚了解；怎么建立连接的、如何发送请求包、超时机制是怎样的，于是专门花时间研究下其实现方式。
---

## 本文主要分为两个个方面来介绍

- http具体实现
- 超时机制

### 1. http具体实现

首先分析一下client的工作流程。 下面是一般我们进行一个请求时的代码事例：

```
func DoRequest(req *http.Request) (MyResponse, error) {
    client := &http.Client{}
    resp, err := client.Do(req)
    if resp != nil {
        defer resp.Body.Close()
    }
    if err != nil {
        return nil, err
    }
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    response := MyResponse{}
    response.Header = resp.Header
    ...
    response.Body = body

    return response, nil
}
```

代码中我们首先创建一个http.Client, 所有的值都是默认值，然后调用client.Do发请求，req是我们请求的结构体。这里我们也可以用client.Get, client.Post等函数来调用，从他们的源码来看都是调用的client.Do。
client.Do的实现在net/http包的go/src/net/http/client.go源文件中。可以看到函数内部主要是实现了一些参数检查，默认值设置，以及对于多跳请求的处理，最为核心的就是:

```
if resp, didTimeout, err = c.send(req, deadline); err != nil {
            // c.send() always closes req.Body
            reqBodyClosed = true
            if !deadline.IsZero() && didTimeout() {
                err = &httpError{
                    err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",
                    timeout: true,
                }
            }
            return nil, uerr(err)
        }

var shouldRedirect bool
redirectMethod, shouldRedirect, includeBody = redirectBehavior(req.Method, resp, reqs[0])
if !shouldRedirect {
    return resp, nil
}
```

这里真正发请求的函数就是c.send, 这个函数的实现也比较简单, 主要是调用了send函数，这个函数的实现主要如下:

```
// didTimeout is non-nil only if err != nil.
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    if c.Jar != nil {
        for _, cookie := range c.Jar.Cookies(req.URL) {
            req.AddCookie(cookie)
        }
    }
    resp, didTimeout, err = send(req, c.transport(), deadline)
    if err != nil {
        return nil, didTimeout, err
    }
    if c.Jar != nil {
        if rc := resp.Cookies(); len(rc) > 0 {
            c.Jar.SetCookies(req.URL, rc)
        }
    }
    return resp, nil, nil
}

// send issues an HTTP request.
// Caller should close resp.Body when done reading from it.
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    ...
        stopTimer, didTimeout := setRequestCancel(req, rt, deadline)
    ...
        resp, err = rt.RoundTrip(req)
    ...
        return resp, nil, nil
}
```

这里真正进行网络交互的定位到的函数是rt.RoundTrip,这个函数的定义是一个interface

```
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

说白了，就是你给它一个request,它给你一个response
下面我们来看一下他的实现，对应源文件net/http/transport.go，这里是http package里面的精髓所在，go里面一个struct就跟一个类一样，transport这个类长这样的

```
type Transport struct {
    idleMu     sync.Mutex
    wantIdle   bool // user has requested to close all idle conns
    idleConn   map[connectMethodKey][]*persistConn
    idleConnCh map[connectMethodKey]chan *persistConn

    reqMu       sync.Mutex
    reqCanceler map[*Request]func()

    altMu    sync.RWMutex
    altProto map[string]RoundTripper // nil or map of URI scheme => RoundTripper
    //Dial获取一个tcp 连接，也就是net.Conn结构，你就记住可以往里面写request
    //然后从里面搞到response就行了
    Dial func(network, addr string) (net.Conn, error)
}
```


两个 map 为 idleConn、idleConnCh，idleConn 是保存从 connectMethodKey （代表着不同的协议 不同的host，也就是不同的请求）到 persistConn 的映射， idleConnCh 用来在并发http请求的时候在多个 goroutine 里面相互发送持久连接，也就是说， 这些持久连接是可以重复利用的， 你的http请求用某个persistConn用完了，通过这个channel发送给其他http请求使用这个persistConn，然后我们找到transport的RoundTrip方法



```
func (t *Transport) RoundTrip(req *Request) (*Response, error) {
 ...
 for {
 ...
 pconn, err := t.getConn(treq, cm)
 ...
 if pconn.alt != nil {
 // HTTP/2 path.
 t.setReqCanceler(req, nil) // not cancelable with CancelRequest
 resp, err = pconn.alt.RoundTrip(req)
 } else {
 resp, err = pconn.roundTrip(treq)
 }
 }
 ...
}
```

忽略错误处理等其他逻辑，函数主逻辑实现两个功能 : t.getConn 和 pconn.roundTrip(treq) 

第一步 ： 通过 t.getConn 获取一个有效连接

```
func (t *Transport) getConn(req *Request, cm connectMethod) (*persistConn, error) {
    if pc := t.getIdleConn(cm); pc != nil {
        // set request canceler to some non-nil function so we
        // can detect whether it was cleared between now and when
        // we enter roundTrip
        t.setReqCanceler(req, func() {})
        return pc, nil
    }
 
    type dialRes struct {
        pc  *persistConn
        err error
    }
    dialc := make(chan dialRes)
    //定义了一个发送 persistConn的channel

    prePendingDial := prePendingDial
    postPendingDial := postPendingDial

    handlePendingDial := func() {
        if prePendingDial != nil {
            prePendingDial()
        }
        go func() {
            if v := <-dialc; v.err == nil {
                t.putIdleConn(v.pc)
            }
            if postPendingDial != nil {
                postPendingDial()
            }
        }()
    }

    cancelc := make(chan struct{})
    t.setReqCanceler(req, func() { close(cancelc) })
 
    // 启动了一个goroutine, 这个goroutine 获取里面调用dialConn搞到
    // persistConn, 然后发送到上面建立的channel  dialc里面，    
    go func() {
        pc, err := t.dialConn(cm)
        dialc <- dialRes{pc, err}
    }()

    idleConnCh := t.getIdleConnCh(cm)
    select {
    case v := <-dialc:
        // dialc 我们的 dial 方法先搞到通过 dialc通道发过来了
        return v.pc, v.err
    case pc := <-idleConnCh:
        // 这里代表其他的http请求用完了归还的persistConn通过idleConnCh这个    
        // channel发送来的
        handlePendingDial()
        return pc, nil
    case <-req.Cancel:
        handlePendingDial()
        return nil, errors.New("net/http: request canceled while waiting for connection")
    case <-cancelc:
        handlePendingDial()
        return nil, errors.New("net/http: request canceled while waiting for connection")
    }
}
```

下面是这个过程的流程图：


![liucheng](/images/http/liucheng.png)



从上面可以看到，先调用getIdleConn去获取已经建立但是目前闲置的链接。

在getIdleConn函数内将运行一个死循环，循环内部会在idleConn这个map内根据connectMethod类型内的一些和请求有关的信息，找到一条满足条件的链接返回给外部使用。

在只有一条空闲链接的时候没有选择，只有使用这条
超过2条及以上的空闲链接，选取最近最久未使用的那一条
如果连接池中没有可用的连接，则会创建一个连接或者从刚刚释放的连接中获取一个，这两个过程时同时进行的，谁先获取到连接就用谁的。

这里要注意的是 idleConnCh 这个通道里面发送来的是其他的http请求用完了归还的persistConn， 如果从这个通道里面搞到了，dialc这个通道也等着发呢，不能浪费，就通过handlePendingDial这个方法把dialc通道里面的persistConn也发到idleConnCh，等待后续给其他http请求使用。

每个新建的persistConn的时候都把tcp连接里地输入流，和输出流用br（br *bufio.Reader）,和bw(bw *bufio.Writer)包装了一下，往bw写就写到tcp输入流里面了，读输出流也是通过br读，并启动了读循环和写循环

```
pconn.br = bufio.NewReader(noteEOFReader{pconn.conn, &pconn.sawEOF})
pconn.bw = bufio.NewWriter(pconn.conn)
go pconn.readLoop()
go pconn.writeLoop()
```

其中readLoop主要是读取从server返回的数据,writeLoop主要发送请求到server,

其具体创建流程如下图所示 ： 

![liucheng](/images/http/conn.png)

里面是各种channel,  有两个channel writeRequest 和 requestAndChan 

```
reqch     chan requestAndChan     // written by roundTrip; read by readLoop
writech   chan writeRequest      // written by roundTrip; read by writeLoop
```

主goroutine 往writeRequest里面写，写循环从writeRequest里面接受

主goroutine 往requestAndChan里面写，读循环从requestAndChan里面接受。

注意这里的channel都是双向channel，也就是channel 的struct里面有一个chan类型的字段， 比如 reqch chan requestAndChan 这里的 requestAndChan 里面的 ch chan responseAndError。

主 goroutine 通过 reqch 发送requestAndChan 给读循环，然后读循环搞到response后通过 requestAndChan 里面的通道responseAndError把response返给主goroutine

```
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    ... 忽略
    // Write the request concurrently with waiting for a response,
    // in case the server decides to reply before reading our full
    // request body.
    writeErrCh := make(chan error, 1)
    pc.writech <- writeRequest{req, writeErrCh}
    //把request发送给写循环
    resc := make(chan responseAndError, 1)
    pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}
    //发送给读循环
    var re responseAndError
    var respHeaderTimer <-chan time.Time
    cancelChan := req.Request.Cancel
WaitResponse:
    for {
        select {
        case err := <-writeErrCh:
            if isNetWriteError(err) {
                //写循环通过这个channel报告错误
                select {
                case re = <-resc:
                    pc.close()
                    break WaitResponse
                case <-time.After(50 * time.Millisecond):
                    // Fall through.
                }
            }
            if err != nil {
                re = responseAndError{nil, err}
                pc.close()
                break WaitResponse
            }
            if d := pc.t.ResponseHeaderTimeout; d > 0 {
                timer := time.NewTimer(d)
                defer timer.Stop() // prevent leaks
                respHeaderTimer = timer.C
            }
        case <-pc.closech:
            // 如果长连接挂了， 这里的channel有数据， 进入这个case, 进行处理
            
            select {
            case re = <-resc:
                if fn := testHookPersistConnClosedGotRes; fn != nil {
                    fn()
                }
            default:
                re = responseAndError{err: errClosed}
                if pc.isCanceled() {
                    re = responseAndError{err: errRequestCanceled}
                }
            }
            break WaitResponse
        case <-respHeaderTimer:
            pc.close()
            re = responseAndError{err: errTimeout}
            break WaitResponse
            // 如果timeout，这里的channel有数据， break掉for循环
        case re = <-resc:
            break WaitResponse
           // 获取到读循环的response, break掉 for循环
        case <-cancelChan:
            pc.t.CancelRequest(req.Request)
            cancelChan = nil
        }
    }

    if re.err != nil {
        pc.t.setReqCanceler(req.Request, nil)
    }
    return re.res, re.err
}
```

这段代码主要干了三件事

主goroutine ->requestAndChan -> 读循环goroutine

主goroutine ->writeRequest-> 写循环goroutine

主goroutine 通过select 监听各个channel上的数据， 比如请求取消， timeout，长连接挂了，写流出错，读流出错， 都是其他goroutine 发送过来的， 跟中断一样，然后相应处理，上面也提到了，有些channel是主goroutine通过channel发送给其他goroutine的struct里面包含的channel, 比如 case err := <-writeErrCh: case re = <-resc:



第二步pconn.roundTrip 调用这个持久连接persistConn 这个struct的roundTrip方法

```
type persistConn struct {
    t        *Transport
    cacheKey connectMethodKey
    conn     net.Conn
    tlsState *tls.ConnectionState
    br       *bufio.Reader       // 从tcp输出流里面读
    sawEOF   bool                // whether we've seen EOF from conn; owned by readLoop
    bw       *bufio.Writer       // 写到tcp输入流
    reqch    chan requestAndChan // 主goroutine 往channnel里面写，读循环从     
                                 // channnel里面接受
    writech  chan writeRequest   // 主goroutine 往channnel里面写                                      
                                 // 写循环从channel里面接受
    closech  chan struct{}       // 通知关闭tcp连接的channel 
    
    writeErrCh chan error

    lk                   sync.Mutex // guards following fields
    numExpectedResponses int
    closed               bool // whether conn has been closed
    broken               bool // an error has happened on this connection; marked broken so it's not reused.
    canceled             bool // whether this conn was broken due a CancelRequest
    // mutateHeaderFunc is an optional func to modify extra
    // headers on each outbound request before it's written. (the
    // original Request given to RoundTrip is not modified)
    mutateHeaderFunc func(Header)
}
```

写循环代码：

```
func (pc *persistConn)writeLoop() {
    for {
        select {
        case wr := <-pc.writech:   //接受主goroutine的 request
            if pc.isBroken() {
                wr.ch <- errors.New("http: can't write HTTP request on broken connection")
                continue
            }
            err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra)   //写入tcp输入流
            if err == nil {
                err = pc.bw.Flush()
            }
            if err != nil {
                pc.markBroken()
                wr.req.Request.closeBody()
            }
            pc.writeErrCh <- err 
            wr.ch <- err         //  出错的时候返给主goroutineto 
        case <-pc.closech:
            return
        }
    }
}
```

pc.writech这个用于写循环的管道在接收了roundtrip方法对其的赋值操作后，有三个关键的操作。

```
err = pc.bw.Flush()
wr.ch <- err // to the roundTrip function
pc.writeErrCh <- err
```

第一个操作是调用了bw的Flush方法，跳转bw成员的定义，可以看到它就是为了向链接以及链接另一边链接的服务端写数据用的，他是一个bufio.Writer类型的对象。也就是说，当经过前期的一些处理之后，我们就可以针对使用的链接写入数据。写入数据之后，无论写入的有没有问题，都会把结果通过writech内的ch成员传递给外层的roundTrip，也会传给writeErrCh这个channel。

既然writech.ch这个成员是为了通知外部的roundTrip方法的，那么writeErrCh这个channel又是用来做什么的呢？首先，在看channel作用的时候，还是要遵守一个原则，channel肯定是连接了两个goroutine。那么这两个goroutine是哪两个呢？查看这个channel的引用我们就可以看到，在readLoop中，我们会调用一个wroteRequest的方法，这个方法内会接受writeErrch这个channel所发送过来的消息。也就是说这个channel是负责链接writeloop和readloop这两个gorountine的。

源码中的注释说，writeErrCh这个channel，是为了能将请求写入的错误从写循环的goroutine中传递到读循环的goroutine中，以此来判定目前的这条连接是否可用，如果在这条连接上写入数据的时候都出错了，那么想在这条连接读数据也会有问题。 看完了写循环中的几个关键操作

roundTrip中的后半部分，起了一个死循环外加一个select，用来监控这条连接上的请求和响应的情况。可以看出在监控writeErrCh这个channel的case上，框架先检查在连接上写入数据有误错误，如果没有，那么就要设置定时器，后面应该会等待响应的传输。 基本上writeloop所做的事情就是这样。



读循环代码：

```
func(pc *persistConn) readLoop() {
    
    ... 忽略
    alive := true
    for alive {
        
        ... 忽略
        rc := <-pc.reqch

        var resp *Response
        if err == nil {
            resp, err = ReadResponse(pc.br, rc.req)
            if err == nil && resp.StatusCode == 100 {
                //100  Continue  初始的请求已经接受，客户应当继续发送请求的其 
                // 余部分
                resp, err = ReadResponse(pc.br, rc.req)
                // 读pc.br（tcp输出流）中的数据，这里的代码在response里面
                //解析statusCode，头字段， 转成标准的内存中的response 类型
                //  http在tcp数据流里面，head和body以 /r/n/r/n分开， 各个头
                // 字段 以/r/n分开
            }
        }

        if resp != nil {
            resp.TLS = pc.tlsState
        }

        ...忽略
        //上面处理一些http协议的一些逻辑行为，
        rc.ch <- responseAndError{resp, err} //把读到的response返回给    
                                             //主goroutine

        .. 忽略
        //忽略部分， 处理cancel req中断， 发送idleConnCh归还pc（持久连接）到持久连接池中（map）    
    pc.close()
}
```

读循环goroutine 通过channel requestAndChan 接受主goroutine发送的request(rc := <-pc.reqch), 并从tcp输出流中读取response， 然后反序列化到结构体中， 最后通过channel 返给主goroutine (rc.ch <- responseAndError{resp, err} )

roundTrip在select中监控了传递给readloop的pc.reqch这个变量里面的ch成员，也就是resc这个channel。可以看出，在成功接收了这个channel返回回来的response之后，roundTrip的任务就完成了，如果期间没有发生任何错误的话，那么这个response就会返回给这个客户端的调用者。

运行到rc := <- pc.reqch这一句的时候，如果pc.reqch这个channel如果还没有接受到数据的时候，会在这里卡住。当外层的roundTrip方法向这个channel发送数据的时候，rc接收到数据，开始进入pc.readResponse的逻辑，这个逻辑就是从连接中读取响应数据的关键。

在pc.readResponse这个方法里，会调用一个同名的大写方法resp, err = ReadResponse(pc.br, rc.req)。这个方法将会从pc.br这个*bufio.Reader类型的对象内，不断的读取response的内容。先逐行的将Http响应的头部的基本信息都解析出来，最后通过readTransfer函数将body以及其他的响应信息都解析出来。

最终结果通过 ioutil.ReadAll(res.Body) 来读取， 实际是通过 bytes/buffer.go 中的 ReadFrom 函数来循环读取数据，直到 io.EOF或返回错误，然后结束循环读取

```
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
    b.lastRead = opInvalid
    for {
        i := b.grow(MinRead)
        m, e := r.Read(b.buf[i:cap(b.buf)])
        if m < 0 {
            panic(errNegativeRead)
        }

        b.buf = b.buf[:i+m]
        n += int64(m)
        if e == io.EOF {
            return n, nil // e is EOF, so return nil explicitly
        }
        if e != nil {
            return n, e
        }
    }
}
```


其中body结构及read()定义在 net/http/transfer.go line 744 


获取并解析完body之后，readLoop中会发分几种情况来处理这个响应。

接收这个响应的时候有错误，那么直接将错误返回给外层的roundTrip
接收的响应没有Body,直接将解析出来的response返回
如果有body，那么对body要做一些处理。最终返回给外部。

### 2. 超时机制

Client端的超时设置，最简单的方式就是使用http.Client的 Timeout字段。 它的时间计算包括从连接(Dial)到读完response body。

```
c := &http.Client{
    Timeout: 15 * time.Second,
}
resp, err := c.Get("https://www.qq.com")
```

主要通过time定时器来实现 

```
src/net/http/client.go -> func (c *Client) Do(req *Request) (*Response, error) {
    ....
    if resp, didTimeout, err = c.send(req, deadline); err != nil {
        // c.send() always closes req.Body reqBodyClosed = true
        if !deadline.IsZero() && didTimeout() {
            err = &httpError{
                err: err.Error() + " (Client.Timeout exceeded while awaiting headers)",
                timeout: true,
            }
        }
        return nil, uerr(err)
    }
}


setRequestCancel(req *Request, rt RoundTripper, deadline time.Time)
    select {
        case <-initialReqCancel:
            doCancel()
            timer.Stop()
        case <-timer.C:
            timedOut.setTrue()
            doCancel()
        case <-stopTimerCh:
            timer.Stop()
    }
```

有一些更细粒度的超时控制：

net.Dialer.Timeout 限制建立TCP连接的时间
http.Transport.TLSHandshakeTimeout 限制 TLS握手的时间
http.Transport.ResponseHeaderTimeout 限制读取response header的时间
http.Transport.ExpectContinueTimeout 限制client在发送包含 Expect: 100-continue的header到收到继续发送body的response之间的时间等待。
其中 ResponseHeaderTimeout 实现是在 net/http/transport.go 中

```
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
    ...
    case <-respHeaderTimer:
        if debugRoundTrip {
            req.logf("timeout waiting for response headers.") 
        } 
        pc.close(errTimeout) 
        return nil, errTimeout
}

...

    case err := <-writeErrCh: 
    if debugRoundTrip { 
        req.logf("writeErrCh resv: %T/%#v", err, err) 
    } 
    if err != nil { 
        pc.close(fmt.Errorf("write error: %v", err)) 
        return nil, pc.mapRoundTripError(req, startBytesWritten, err) 
    } 
    if d := pc.t.ResponseHeaderTimeout; d > 0 { 
        if debugRoundTrip { 
            req.logf("starting timer for %v", d) 
        } 
        timer := time.NewTimer(d) 
        defer timer.Stop() // prevent leaks 
        respHeaderTimer = timer.C 
    }
```

通过 SetDeadlines 设置超时的实现

由conn->fd 返回 

1. src/internal/poll/fd_poll_runtime.go 
func runtime_pollSetDeadline(ctx uintptr, d int64, mode int) 

2. src/net/net.go 
func (c *conn) Read(b []byte) (int, error)
