---
title: go context包源码分析
date: 2019-05-19
tags: go
---
>go的context包可以用来实现goroutine间的数据同步与控制goroutine的生命周期

## 最佳实践
* Context最好显式的作为函数的第一个函数进行传递,而非保存在一个结构体中
* 不要传递一个nil的context,如果不知道需要使用哪种context,传递context.TODO
* 使用context Values保存请求级别的数据而不是用来传递函数的参数
* WithCancel,WithDeadline,WithTimeout接收一个parent context并且返回一个衍生出的child context和一个CancelFunc.调用CancelFunc之后会取消child context和child context对应的children,并且删除掉parent context对child context的引用,停止相关连的timers.

## 代码分析

### WithCancel

关键结构体:
```
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}

type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

type cancelCtx struct { 
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
cancelCtx中children字段保存context的父子关系.WithCancel函数创建子context以及cancel()函数中取消一个context的时候会相应的建立children结构以及取消子context.下文对应的函数中详述.
                 
首先看一下WithCancel()函数:
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
首先将cancelCtx中的Context匿名字段赋值为parent,然后在cancelCtx的children字段中建立父子关系.最后返回cancelCtx,以及一个CancelFunc,CancelFunc可以看到调用时会执行cancelCtx的cancel函数.我们看看cancel函数

```
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	...
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	...
}
```
cancel函数中比较关键的有三个部分:
* 将c.err字段赋值为err
* close(c.done),即将cancelCtx中的done channel关闭
* 遍历c.children,依次执行cancel函数,即父context cancel之后,所有的子context也会依次cancel.

而Context接口中的Done和Err函数在cancelCtx中实现如下:
```
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```
很简单,Done函数获取cancelCtx结构中的done字段,Err()函数获取cancelCtx中的err字段.二者都是在cancel()函数调用时进行的关闭与赋值.

通过WithCancel建立cancelCtx之后通过在goroutine中检测ctx.Done()函数,当上层cancel之后该会关闭该管道,此时goroutine可以退出

### WithDeadline

```

type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```
WithDeadline函数返回一个timerCtx和timerCtx定义的cancel()函数

```
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
WithDeadline()做了如下几个判断:
* 如果parent context已经是一个timerCtx,并且deadline早于现在要设置的deadline,则不再设置定时器,直接调用WithCancel()并返回
* 如果deadline小于当前时间,直接取消后返回
* 否则设置一个定时器然后返回

```
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
timerCtx的cancel()函数多了一个停止定时器的步骤

### WithTimeout
WithTimeout函数很简单,将当前时间加timeout即可转换为WithDeadline函数
```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

### WithValue

```
type valueCtx struct {
	Context
	key, val interface{}
}
```
WithValue返回的是一个valueCtx结构体,除了Context匿名字段之外,包括了一个key和val字段,二者都是接口类型

```
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```
WithValue函数返回一个valueCtx
```
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}

```
Value函数通过key获取相应的值,会逐层依次寻找