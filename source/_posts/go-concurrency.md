---
title: go并发编程
date: 2019-06-01
tags: go
---
>并发编程常见问题及go的锁,条件变量及原子操作

## 竞态条件
第一个版本,不带锁,起10个协程并发的修改Counter,可以看到结果每次都不一样,并且没有规律
```
package main

import (
	"fmt"
	"sync"
	"time"
)

type Counter struct {
	count int
}

func (c *Counter) increment() {
	c.count++
}

func (c *Counter) getCount() int {
	return c.count
}
func main() {
	var wg sync.WaitGroup
	c := &Counter{count: 0}
	nums := 10
	for i := 1; i <= nums; i++ {
		wg.Add(1)
		go func(c *Counter, i int) {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				c.increment()
			}
			fmt.Printf("goroutine %d:%d\n", i, c.getCount())
		}(c, i)
	}
	time.Sleep(1 * time.Microsecond)

	wg.Wait()
	fmt.Printf("main:%d\n", c.getCount())

}

```
结果如下:
```
goroutine 5:1000
goroutine 8:7000
goroutine 9:8000
goroutine 4:9591
goroutine 3:6000
goroutine 1:3000
goroutine 6:4000
goroutine 2:2000
goroutine 7:5000
goroutine 10:9000
main:9591
```
c.count++的操作是读取,修改,写入,如果两个goroutine同时读取c.count,假设值为100,则都会修改为101,并且第二个goroutine写入101后覆盖掉第一个goroutine写入的101.

加锁版本如下:
```
type Counter struct {
	count int
	sync.Mutex
}

func (c *Counter) Increment() {
	c.Lock()
	c.count++
	c.Unlock()
}

func (c *Counter) GetCount() int {
	return c.count
}
```
结果如下:
```
goroutine 5:2600
goroutine 2:3497
goroutine 4:1572
goroutine 6:5191
goroutine 3:6317
goroutine 1:7902
goroutine 8:8743
goroutine 9:8656
goroutine 10:9235
goroutine 7:10000
main:10000
```
可以看到,最终结果是固定的10000
上边的竞态条件比较简单,我们可以直接使用原子操作,如下:
```
type Counter struct {
	count int64
}

func (c *Counter) Increment() {
	atomic.AddInt64(&c.count, 1)
}

func (c *Counter) GetCount() int64 {
	return c.count
}
```
由于atomic包只有AddInt64和AddInt32方法,因此修改count的类型为int64,结果如下:
```
goroutine 2:1000
goroutine 1:5000
goroutine 9:6000
goroutine 6:3000
goroutine 7:4000
goroutine 8:7235
goroutine 10:8167
goroutine 3:9104
goroutine 4:10000
goroutine 5:2000
main:10000
```
结果也是固定为10000

go有内置的竞态检测机制(当两个goroutine同时访问同一个变量,并且至少一个是写的时候就会发生竞态),如下:
```
localhost:copywriter.io didi$ go run -race concurrency/origin.go 
==================
goroutine 1:1000
WARNING: DATA RACE
Read at 0x00c000096010 by goroutine 7:
  main.main.func1()
      /Users/didi/go/src/copywriter.io/concurrency/origin.go:14 +0x7d

Previous write at 0x00c000096010 by goroutine 6:
  main.main.func1()
      /Users/didi/go/src/copywriter.io/concurrency/origin.go:14 +0x96

Goroutine 7 (running) created at:
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/origin.go:26 +0xf6

Goroutine 6 (running) created at:
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/origin.go:26 +0xf6
==================
goroutine 2:2182
goroutine 3:2232
goroutine 4:2316
goroutine 10:2521
goroutine 6:3526
goroutine 5:3550
goroutine 8:3634
goroutine 9:3744
goroutine 7:4562
main:4562
Found 1 data race(s)
exit status 66
```
使用go的benchmark测试一下sync.Mutex和atomic的性能:
```
localhost:concurrency didi$ go test -bench=.
goos: darwin
goarch: amd64
pkg: copywriter.io/concurrency
BenchmarkLock-4             3000            481995 ns/op
BenchmarkAtomi-4           10000            165756 ns/op
PASS
ok      copywriter.io/concurrency       3.176s
```
Mutex是Atomic性能的1/3

## 乱序执行
```
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```
如上代码,可能会打印出2,0.原因为编译器或者CPU可能会乱序执行 a=1和b=2两个语句,当执行g()的时候b已经更新为2,但是a仍然为1

## 可见性
```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```
上述代码首先不能保证会打印出"hello,world",因为可能会乱序.更糟糕的是,
main函数可能会永远无法退出,因为done的可见性不能保证.虽然在另一个协程中更新了done,但main函数中不能保证会读取到正确的done

使用竞态检测器检测结果如下:
```
localhost:concurrency didi$ go run -race condition.go 
==================
WARNING: DATA RACE
Write at 0x0000012107c7 by goroutine 6:
  main.setup()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:10 +0x70

Previous read at 0x0000012107c7 by main goroutine:
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:15 +0x56

Goroutine 6 (running) created at:
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:14 +0x46
==================
==================
WARNING: DATA RACE
Read at 0x0000011f4230 by main goroutine:
  runtime.convT2Estring()
      /usr/local/go/src/runtime/iface.go:332 +0x0
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:17 +0x86

Previous write at 0x0000011f4230 by goroutine 6:
  main.setup()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:9 +0x3e

Goroutine 6 (finished) created at:
  main.main()
      /Users/didi/go/src/copywriter.io/concurrency/condition.go:14 +0x46
==================
hello, world
Found 2 data race(s)
exit status 66
```
使用条件变量改写如下:
```
package main

import (
	"fmt"
	"sync"
)

var a string
var done bool
var lock sync.Mutex
var cond *sync.Cond

func setup() {
	defer lock.Unlock()
	lock.Lock()
	a = "hello, world"
	done = true
	cond.Signal()
}

func main() {
	defer lock.Unlock()
	lock.Lock()
	cond = sync.NewCond(&lock)
	go setup()
	for !done {
		cond.Wait()
	}
	fmt.Println(a)
}
```
检测器输出结果为:
```
localhost:concurrency didi$ go run  -race condition1.go 
hello, world
```
压测前后两个版本,比较增加锁之后的性能:
```
BenchmarkCondition1-4            2000000               624 ns/op
BenchmarkCondition2-4           10000000               182 ns/op
```
将条件变量改为channel,如下:
```
package main

import "fmt"

var a3 = make(chan string, 1)
var done3 = make(chan bool, 1)

func setup3() {
	a3 <- "hello, world"
	done3 <- true
}

func main() {

	go setup3()
	<-done3
	fmt.Println(<-a3)
}
```
压测结果如下:
```
BenchmarkCondition1-4            2000000               612 ns/op
BenchmarkCondition2-4           10000000               192 ns/op
BenchmarkCondition3-4            3000000               534 ns/op
```
condition1为加条件变量版本,2为有竞态问题的版本,3为channel版本
## 参考链接
* https://golang.org/doc/articles/race_detector.html
* https://golang.org/ref/mem