---
title: '源码走读: fasthttp 为什么这么快?'
tags: ['fasthttp']
categories: ['golang', '实战', '性能优化']
series: ['Golang知识汇总']
author: ['zhu733756']
date: 2024-12-27T16:57:56+08:00
---

## 前戏

> 今天我们聊聊 `golang` 的 `web` 框架, `fasthttp` 和 `gin` 到底谁更丝滑?

## fasthttp 简介

### 安装

```
go get -u github.com/valyala/fasthttp
```

### fasthttp 性能吊打 net/http

简而言之，`fasthttp`服务器的速度比`net/http`快多达 10 倍, 具体参数可以查看官网[https://github.com/valyala/fasthttp](https://github.com/valyala/fasthttp)

### fasthttp 使用方式与 net/http 有所差异

```golang
// using the given handler.
func ListenAndServe(addr string, handler RequestHandler) error {
	s := &Server{
		Handler: handler,
	}
	return s.ListenAndServe(addr)
}

type RequestHandler func(ctx *RequestCtx)
```

可以这样使用:

```go
type MyHandler struct {
	foobar string
}

// request handler in net/http style, i.e. method bound to MyHandler struct.
func (h *MyHandler) HandleFastHTTP(ctx *fasthttp.RequestCtx) {
	// notice that we may access MyHandler properties here - see h.foobar.
	fmt.Fprintf(ctx, "Hello, world! Requested path is %q. Foobar is %q",
		ctx.Path(), h.foobar)
}

// pass bound struct method to fasthttp
myHandler := &MyHandler{
	foobar: "foobar",
}
fasthttp.ListenAndServe(":8080", myHandler.HandleFastHTTP)
```

在多个`handler`的时候需要这样使用:

```go
// the corresponding fasthttp code
m := func(ctx *fasthttp.RequestCtx) {
	switch string(ctx.Path()) {
	case "/foo":
		fooHandlerFunc(ctx)
	case "/bar":
		barHandlerFunc(ctx)
	case "/baz":
		bazHandler.HandlerFunc(ctx)
	default:
		ctx.Error("not found", fasthttp.StatusNotFound)
	}
}

fasthttp.ListenAndServe(":80", m)
```

### fasthttp 源码走读

推荐一个在线`vscode`页面: `https://github1s.com/valyala/fasthttp/tree/master`(https://github1s.com/valyala/fasthttp/tree/master)

![vscode online](/posts/golang_fasthttp/vscode_online.png)

#### 极致对象复用, 减缓频繁的 GC

1.`request`重用:

```golang
func AcquireRequest() *Request {
	v := requestPool.Get()
	if v == nil {
		return &Request{}
	}
	return v.(*Request)
}

func ReleaseRequest(req *Request) {
	req.Reset()
	requestPool.Put(req)
}
```

2. `response`重用:

```golang
func AcquireResponse() *Response {
	v := responsePool.Get()
	if v == nil {
		return &Response{}
	}
	return v.(*Response)
}

func ReleaseResponse(resp *Response) {
	resp.Reset()
	responsePool.Put(resp)
}
```

3. `args` 重用:

```golang
func AcquireArgs() *Args {
	return argsPool.Get().(*Args)
}

func ReleaseArgs(a *Args) {
	a.Reset()
	argsPool.Put(a)
}

var argsPool = &sync.Pool{
	New: func() any {
		return &Args{}
	},
}
```

4. `reader` 重用:

```golang
func (c *HostClient) acquireReader(conn net.Conn) *bufio.Reader {
	var v any
	if c.clientReaderPool != nil {
		v = c.clientReaderPool.Get()
		if v == nil {
			n := c.ReadBufferSize
			if n <= 0 {
				n = defaultReadBufferSize
			}
			return bufio.NewReaderSize(conn, n)
		}
	} else {
		v = c.readerPool.Get()
		if v == nil {
			n := c.ReadBufferSize
			if n <= 0 {
				n = defaultReadBufferSize
			}
			return bufio.NewReaderSize(conn, n)
		}
	}

	br := v.(*bufio.Reader)
	br.Reset(conn)
	return br
}

func (c *HostClient) releaseReader(br *bufio.Reader) {
	if c.clientReaderPool != nil {
		c.clientReaderPool.Put(br)
	} else {
		c.readerPool.Put(br)
	}
}
```

5. `writer` 重用:

```golang
func acquireWriter(ctx *RequestCtx) *bufio.Writer {
	v := ctx.s.writerPool.Get()
	if v == nil {
		n := ctx.s.WriteBufferSize
		if n <= 0 {
			n = defaultWriteBufferSize
		}
		return bufio.NewWriterSize(ctx.c, n)
	}
	w := v.(*bufio.Writer)
	w.Reset(ctx.c)
	return w
}

func releaseWriter(s *Server, w *bufio.Writer) {
	s.writerPool.Put(w)
}
```

6. `conn` 重用:

```golang
func ServeConn(c net.Conn, handler RequestHandler) error {
	v := serverPool.Get()
	if v == nil {
		v = &Server{}
	}
	s := v.(*Server)
	s.Handler = handler
	err := s.ServeConn(c)
	s.Handler = nil
	serverPool.Put(v)
	return err
}

var serverPool sync.Pool
```

7. `bytes`重用

```golang
type byteBuffer struct {
	b []byte
}

func acquireByteBuffer() *byteBuffer {
	return byteBufferPool.Get().(*byteBuffer)
}

func releaseByteBuffer(b *byteBuffer) {
	if b != nil {
		byteBufferPool.Put(b)
	}
}

var byteBufferPool = &sync.Pool{
	New: func() any {
		return &byteBuffer{
			b: make([]byte, 1024),
		}
	},
}
```

8. ... 基本能重用的地方都用了`sync.Pool`

#### 工作池 workerPool

```golang
// Serve serves incoming connections from the given listener.
//
// Serve blocks until the given listener returns permanent error.
func (s *Server) Serve(ln net.Listener) error {
	var lastOverflowErrorTime time.Time
	var lastPerIPErrorTime time.Time

	maxWorkersCount := s.getConcurrency()

	s.mu.Lock()
	s.ln = append(s.ln, ln)
	if s.done == nil {
		s.done = make(chan struct{})
	}
	if s.concurrencyCh == nil {
		s.concurrencyCh = make(chan struct{}, maxWorkersCount)
	}
	s.mu.Unlock()

	wp := &workerPool{
		WorkerFunc:            s.serveConn,
		MaxWorkersCount:       maxWorkersCount,
		LogAllErrors:          s.LogAllErrors,
		MaxIdleWorkerDuration: s.MaxIdleWorkerDuration,
		Logger:                s.logger(),
		connState:             s.setState,
	}
	wp.Start()

	// Count our waiting to accept a connection as an open connection.
	// This way we can't get into any weird state where just after accepting
	// a connection Shutdown is called which reads open as 0 because it isn't
	// incremented yet.
	atomic.AddInt32(&s.open, 1)
	defer atomic.AddInt32(&s.open, -1)

	for {
		c, err := acceptConn(s, ln, &lastPerIPErrorTime)
		if err != nil {
			wp.Stop()
			if err == io.EOF {
				return nil
			}
			return err
		}
		s.setState(c, StateNew)
		atomic.AddInt32(&s.open, 1)
		if !wp.Serve(c) {
			atomic.AddInt32(&s.open, -1)
			atomic.AddUint32(&s.rejectedRequestsCount, 1)
			s.writeFastError(c, StatusServiceUnavailable,
				"The connection cannot be served because Server.Concurrency limit exceeded")
			c.Close()
			s.setState(c, StateClosed)
			if time.Since(lastOverflowErrorTime) > time.Minute {
				s.logger().Printf("The incoming connection cannot be served, because %d concurrent connections are served. "+
					"Try increasing Server.Concurrency", maxWorkersCount)
				lastOverflowErrorTime = time.Now()
			}

			// The current server reached concurrency limit,
			// so give other concurrently running servers a chance
			// accepting incoming connections on the same address.
			//
			// There is a hope other servers didn't reach their
			// concurrency limits yet :)
			//
			// See also: https://github.com/valyala/fasthttp/pull/485#discussion_r239994990
			if s.SleepWhenConcurrencyLimitsExceeded > 0 {
				time.Sleep(s.SleepWhenConcurrencyLimitsExceeded)
			}
		}
	}
}
```

1. 工作池启动

```golang
func (wp *workerPool) Start() {
	if wp.stopCh != nil {
		return
	}
	wp.stopCh = make(chan struct{})
	stopCh := wp.stopCh
	wp.workerChanPool.New = func() any {
		return &workerChan{
			ch: make(chan net.Conn, workerChanCap),
		}
	}
	go func() {
		var scratch []*workerChan
		for {
      // Clean least recently used workers if they didn't serve connections
	    // for more than maxIdleWorkerDuration.
			wp.clean(&scratch)
			select {
			case <-stopCh:
				return
			default:
				time.Sleep(wp.getMaxIdleWorkerDuration())
			}
		}
	}()
}
```

2. 处理新来的连接

```golang
func (wp *workerPool) Serve(c net.Conn) bool {
	ch := wp.getCh()
	if ch == nil {
		return false
	}
	ch.ch <- c
	return true
}

func (wp *workerPool) getCh() *workerChan {
	var ch *workerChan
	createWorker := false

	wp.lock.Lock()
	ready := wp.ready
	n := len(ready) - 1
	if n < 0 {
		if wp.workersCount < wp.MaxWorkersCount {
			createWorker = true
			wp.workersCount++
		}
	} else {
		ch = ready[n]
		ready[n] = nil
		wp.ready = ready[:n]
	}
	wp.lock.Unlock()

	if ch == nil {
		if !createWorker {
			return nil
		}
		vch := wp.workerChanPool.Get()
		ch = vch.(*workerChan)
		go func() {
      // workerFunc 这个方法启动一个携程处理连接
			wp.workerFunc(ch)
			wp.workerChanPool.Put(vch)
		}()
	}
	return ch
}
```

简单的流程如下:

- getCh() 根据限制的 `worker`数量, 从`workerChanPool`复用一个`workerChan`
- workerFunc() 启动一个携程，处理连接, 直接连接处理完毕, 遇到`hajicked`错误, 或者要求强制停止
- 回收 workerChan

3. 工作池关闭

#### 官方推荐的一些细节性能优化摘录

- 推荐使用`srcLen := len(src)`, 而不是:

```go
srcLen := 0
if src != nil {
	srcLen = len(src)
}
```

2. []byte 缓冲区可以扩展到其容量。

```go
buf := make([]byte, 100)
a := buf[:10]  // len(a) == 10, cap(a) == 100.
b := a[:100]  // 由于cap(a) == 100，这是有效的。
```

3. 字符串和[]byte 缓冲区可以在不进行内存分配的情况下转换

```go
func b2s(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

func s2b(s string) (b []byte) {
    bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh.Data = sh.Data
    bh.Cap = sh.Len
    bh.Len = sh.Len
    return b
}
```

## 小尾巴

> fasthttp 之所以快, 是因为大量使用对象复用, 工作池复用`goroutine`, 在细节上处理了内存的分配, 让程序更有效~

> 软件没有银弹, 由于有大量的对象复用, 有可能 QPS 过猛会导致对象复用后[数据错乱](https://cloud.tencent.com/developer/news/462918), 其次, 也要考虑大对象复用给小对象的内存浪费~

> 如果没有极致的性能优化需求, 一般`gin`够用了~
