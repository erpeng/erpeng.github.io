---
title: codis proxy处理流程
date: 2019-01-10 15:14:04
tags: codis
---

## proxy启动
cmd/proxy/main.go文件

解析配置文件之后重点是proxy.New(config)函数

该函数中，首先会创建一个Proxy结构体，如下:
```go
type Proxy struct {
    mu sync.Mutex

	...
    config *Config
    router *Router //Router中比较重要的是连接池和slots
	...
	lproxy net.Listener //19000端口的Listener
    ladmin net.Listener //11080端口的Listener
	...
}
```
然后起两个协程,分别处理11080和19000端口的请求
```go
    go s.serveAdmin()
    go s.serveProxy()
```
我们重点看s.serveProxy()的处理流程，即redis client连接19000端口后proxy如何分发到codis server并且将结果返回到客户端

## Proxy处理
s.serverProxy也启动了两个协程，一个协程对router中连接池中的连接进行连接可用性检测，另一个协程是一个死循环，accept lproxy端口的连接，并且启动一个新的Session进行处理，代码流程如下:
```go
    go func(l net.Listener) (err error) {
        defer func() {
            eh <- err
        }()
        for {
            c, err := s.acceptConn(l)//accept连接
            if err != nil {
                return err
            }
            NewSession(c, s.config).Start(s.router)//启动一个新的session进行处理
        }
    }(s.lproxy)//s为proxy,s.lproxy即19000端口的监听
```
首先介绍一下Request结构体，该结构体会贯穿整个流程
```go
type Request struct {
    Multi []*redis.Resp  //保存请求命令,按redis的resp协议类型将请求保存到Multi字段中
    Batch *sync.WaitGroup //返回响应时,会在Batch处等待,r.Batch.Wait(),所以可以做到当请求执行完成后才会执行返回函数

    Group *sync.WaitGroup

    Broken *atomic2.Bool

    OpStr string
    OpFlag

    Database int32
    UnixNano int64

    *redis.Resp //保存响应数据,也是redis的resp协议类型
    Err error

    Coalesce func() error //聚合函数,适用于mget/mset等需要聚合响应的操作命令
}
```
Start函数处理流程如下:
```go
        tasks := NewRequestChanBuffer(1024)//tasks是一个指向RequestChan的指针,RequestChan结构体中有一个data字段,data字段是个数组，保存1024个指向Request的指针

        go func() {
            s.loopWriter(tasks)//从RequestChan的data中取出请求并且返回给客户端，如果是mget/mset这种需要聚合相应的请求,则会等待所有拆分的子请求执行完毕后执行聚合函数，然后将结果返回给客户端
            decrSessions()
        }()

        go func() {
            s.loopReader(tasks, d)//首先根据key计算该key分配到哪个slot.在此步骤中只会将slot对应的连接取出，然后将请求放到连接的input字段中。
            tasks.Close()
        }()
```
可以看到,s.loopWriter只是从RequestChan的data字段中取出请求并且返回给客户端，通过上文Request结构体的介绍，可以看到，通过在request的Batch执行wait操作，只有请求处理完成后loopWriter才会执行

下边我们看loopReader的执行流程

```go
  		r := &Request{}   //新建一个Request结构体，该结构体会贯穿请求的始终，请求字段，响应字段都放在Request中
        r.Multi = multi
        r.Batch = &sync.WaitGroup{}
        r.Database = s.database
        r.UnixNano = start.UnixNano()

        if err := s.handleRequest(r, d); err != nil {  //执行handleRequest函数，处理请求
            r.Resp = redis.NewErrorf("ERR handle request, %s", err) 
            tasks.PushBack(r)
            if breakOnFailure {
                return err
            }
        } else {
            tasks.PushBack(r) //如果handleRequest执行成功，将请求r放入tasks(即上文的RequestChan)的data字段中。loopWriter会从该字段中获取请求并且返回给客户端
        }

```
看handleRequest函数如何处理请求,重点是router的dispatch函数
```go
func (s *Router) dispatch(r *Request) error {
    hkey := getHashKey(r.Multi, r.OpStr)//hkey为请求的key
    var id = Hash(hkey) % MaxSlotNum //hash请求的key之后对1024取模,获取该key分配到哪个slot
    slot := &s.slots[id] //slot都保存在router的slots数组中,获取对应的slot
    return slot.forward(r, hkey)//执行slot的forward函数
}
```
forward函数调用process函数，返回一个BackendConn结构,然后调用其PushBack函数将请求放入bc.input中
```go
func (d *forwardSync) Forward(s *Slot, r *Request, hkey []byte) error {
    s.lock.RLock()
    bc, err := d.process(s, r, hkey) //返回一个连接,并且将请求放入BackendConn的input中
    s.lock.RUnlock()
    if err != nil {
        return err
    }
    bc.PushBack(r)
    return nil
}
bc.PushBack(r)函数如下:

func (bc *BackendConn) PushBack(r *Request) {
    if r.Batch != nil {
        r.Batch.Add(1) //将请求的Batch执行add 1的操作，注意前文中的loopWriter会在Batch处等待
    }
    bc.input <- r //将请求放入bc.input channel
}
```
至此可以看到,Proxy的处理流程
```
loopWriter->RuquestChan的data字段中读取请求并且返回。在Batch处等待

loopReader->将请求放入RequestChan的data字段中，并且将请求放入bc.input channel中。在Batch处加1
```
很明显,Proxy并没有真正处理请求,肯定会有goroutine从bc.input中读取请求并且处理完成后在Batch处减1，这样当请求执行完成后,loopWriter就可以返回给客户端端响应了。

## BackendConn的处理流程
从上文得知,proxy结构体中有一个router字段，类型为Router,结构体类型如下:
```go
type Router struct {
    mu sync.RWMutex
    pool struct {
        primary *sharedBackendConnPool //连接池
        replica *sharedBackendConnPool
    }
    slots [MaxSlotNum]Slot //slot
	...
}
```
Router的pool中管理连接池,执行fillSlot时会真正生成连接，放入Slot结构体的backend字段的bc字段中，Slot结构体如下:
```go
type Slot struct {
    id   int
    ...
    backend, migrate struct {
        id int
        bc *sharedBackendConn
    }
	...
    method forwardMethod
}
```
我们看一下bc字段的结构体sharedBackendConn:
```go
type sharedBackendConn struct {
    addr string //codis server的地址
    host []byte //codis server主机名
    port []byte //codis server的端口

    owner *sharedBackendConnPool //属于哪个连接池
    conns [][]*BackendConn //二维数组,一般codis server会有16个db,第一个维度为0-15的数组,每个db可以有多个BackendConn连接

    single []*BackendConn //如果每个db只有一个BackendConn连接，则直接放入single中。当每个db有多个连接时会从conns中选一个返回，而每个db只有一个连接时，直接从single中返回

    refcnt int
}
```
每个BackendConn中有一个 input chan *Request字段,是一个channel,channel中的内容为Request指针。也就是第二章节loopReader选取一个BackendConn后，会将请求放入input中。

下边我们看看处理BackendConn input字段中数据的协程是如何启动并处理数据的。代码路径为pkg/proxy/backend.go的newBackendConn函数

```go

func NewBackendConn(addr string, database int, config *Config) *BackendConn {
    bc := &BackendConn{
        addr: addr, config: config, database: database,
    }
    //1024长度的管道,存放1024个*Request
    bc.input = make(chan *Request, 1024)
    bc.retry.delay = &DelayExp2{
        Min: 50, Max: 5000,
        Unit: time.Millisecond,
    }

    go bc.run()

    return bc
}
```
可以看到，在此处创建的BackendConn结构，并且初始化bc.input字段。连接池的建立是在proxy初始化启动的时候就会建立好。继续看bc.run()函数的处理流程
```go
func (bc *BackendConn) run() {
    log.Warnf("backend conn [%p] to %s, db-%d start service",
        bc, bc.addr, bc.database)
    for round := 0; bc.closed.IsFalse(); round++ {
        log.Warnf("backend conn [%p] to %s, db-%d round-[%d]",
            bc, bc.addr, bc.database, round)
        if err := bc.loopWriter(round); err != nil { //执行loopWriter函数，此处的loopWriter和第二章节的loopWriter只是名称相同，是两个不同的处理函数
            bc.delayBeforeRetry()
        }
    }
    log.Warnf("backend conn [%p] to %s, db-%d stop and exit",
        bc, bc.addr, bc.database)
}
 
func (bc *BackendConn) loopWriter(round int) (err error) {
    ...
    c, tasks, err := bc.newBackendReader(round, bc.config) //调用newBackendReader函数。注意此处的tasks也是一个存放*Request的channel,用来此处的loopWriter和loopReader交流信息
    if err != nil {
        return err
    }
    ...

    for r := range bc.input { //可以看到,此处的loopWriter会从bc.input中取出数据并且处理
		...
        if err := p.EncodeMultiBulk(r.Multi); err != nil { //将请求编码并且发送到codis server
            return bc.setResponse(r, nil, fmt.Errorf("backend conn failure, %s", err))
        }
        if err := p.Flush(len(bc.input) == 0); err != nil {
            return bc.setResponse(r, nil, fmt.Errorf("backend conn failure, %s", err))
        } else {
            tasks <- r  //将请求放入tasks这个channel中
        }
    }
    return nil
}
```
注意此处的loopWriter会从bc.input中取出数据发送到codis server,bc.newBackendReader会起一个loopReader,从codis server中读取数据并且写到request结构体中，此处的loopReader和loopWriter通过tasks这个channel通信。
```go
func (bc *BackendConn) newBackendReader(round int, config *Config) (*redis.Conn, chan<- *Request, error) {
    ...
    tasks := make(chan *Request, config.BackendMaxPipeline)//创建task这个channel并且返回给loopWriter
    go bc.loopReader(tasks, c, round)//启动loopReader

    return c, tasks, nil
}
func (bc *BackendConn) loopReader(tasks <-chan *Request, c *redis.Conn, round int) (err error) {
   	...
    for r := range tasks {  //从tasks中取出响应
        resp, err := c.Decode()
        if err != nil {
            return bc.setResponse(r, nil, fmt.Errorf("backend conn failure, %s", err))
        }
        ...
        bc.setResponse(r, resp, nil)//设置响应数据到request结构体中
    }
    return nil
}

func (bc *BackendConn) setResponse(r *Request, resp *redis.Resp, err error) error {
    r.Resp, r.Err = resp, err //Request的Resp字段设置为响应值
    if r.Group != nil {
        r.Group.Done()
    }
    if r.Batch != nil {
        r.Batch.Done() //注意此处会对Batch执行减1操作，这样proxy中的loopWriter可以聚合响应并返回
    }
    return err
}
```
总结一下,BackendConn中的函数功能如下
```
loopWriter->从bc.input中取出请求并且发给codis server,并且将请求放到tasks channel中

loopReader->从tasks中取出请求，设置codis server的响应字段到Request的Resp字段中，并且将Batch执行减1操作
```

小结
一图胜千言，图片版权归李老师，如下

![codis](/img/codis1.png)