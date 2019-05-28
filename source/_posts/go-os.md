---
title: go os包分析
date: 2019-05-27
tags: go
---
>os包主要实现操作系统相关的一些操作接口, 例如chown,chmod,getgid,getpid等.不详述,重点关注一些定义的结构体与特殊处理

## 错误处理相关函数
```
go doc os |grep Is
func IsExist(err error) bool //是否报已存在的错误
func IsNotExist(err error) bool //是否报不存在的错误
func IsPermission(err error) bool //是否报权限相关错误
func IsTimeout(err error) bool //是否报超时相关的错误
```

一个例子:
```
package main

import (
	"fmt"
	"os"
)

func main() {
	filename := "a-nonexistent-file"
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		fmt.Println("file does not exist")
		fmt.Println(err)
	}
}

output:
file does not exist
stat a-nonexistent-file: no such file or directory
```

os.Stat实际报错为如下结构:
```
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
```
可以看到,Op为stat(动作),Path为实际文件名,Err中保存的为实际错误

同理,定义了LinkError
```
type LinkError struct {
	Op  string
	Old string
	New string
	Err error
}

func (e *LinkError) Error() string {
	return e.Op + " " + e.Old + " " + e.New + ": " + e.Err.Error()
}
```

## 一些结构体

### File

```
func Open(name string) (*File, error)
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```
注意Open函数默认打开为O_RDONLY只读模式

```
func Open(name string) (*File, error) {
	return OpenFile(name, O_RDONLY, 0)
}
```


### FileInfo

```
func Lstat(name string) (FileInfo, error)
func Stat(name string) (FileInfo, error)
type FileInfo interface {
	Name() string       // base name of the file
	Size() int64        // length in bytes for regular files; system-dependent for others
	Mode() FileMode     // file mode bits
	ModTime() time.Time // modification time
	IsDir() bool        // abbreviation for Mode().IsDir()
	Sys() interface{}   // underlying data source (can return nil)
}
```

FileMode定义为
```
type FileMode uint32
```

一个例子:
```
package main

import (
	"fmt"
	"log"
	"os"
)

func main() {
	fi, err := os.Lstat("some-filename")
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("permissions: %#o\n", fi.Mode().Perm()) // 0400, 0777, etc.
	switch mode := fi.Mode(); {
	case mode.IsRegular():
		fmt.Println("regular file")
	case mode.IsDir():
		fmt.Println("directory")
	case mode&os.ModeSymlink != 0:
		fmt.Println("symbolic link")
	case mode&os.ModeNamedPipe != 0:
		fmt.Println("named pipe")
	}
}

```
可以看到,一般perm写成8进制0755或者0400,二进制比特占用9个.
FileMode整体为32个bit位,最低9位表示权限,其余位置代表是否是普通文件或者是否是目录或者命名管道或者链接等等


### Process

```
type ProcAttr struct {
	Dir string
	Env []string
	Files []*File
	Sys *syscall.SysProcAttr
}

type Process struct {
	Pid    int
	handle uintptr      // handle is accessed atomically on Windows
	isdone uint32       // process has been successfully waited on, non zero if true
	sigMu  sync.RWMutex // avoid race between wait and signal
}

type ProcessState struct {
	pid    int                // The process's id.
	status syscall.WaitStatus // System-dependent status info.
	rusage *syscall.Rusage
}

type Signal interface {
	String() string
	Signal() // to distinguish from other Stringers
}
```
一个例子 
```
	argv := []string{"/etc/"}
	attr := &os.ProcAttr{Env: []string{"foo=bar", "foo1=bar1"}}
	p, err := os.StartProcess("/bin/ls", argv, attr) //开启一个进程,函数签名如下:
	//func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Println(p.Pid)
	ps, err := p.Wait() //释放资源退出并且返回一个ProcessState结构
	if err != nil {
		fmt.Println(err)
		os.Exit(2)
	}
	fmt.Println(ps.Pid())
	fmt.Println(ps.UserTime())
	fmt.Println(ps.SystemTime())

	fmt.Println(ps.String())
	fmt.Println(ps.Exited())
	fmt.Println(ps.Sys())
输出如下:
	44294
	44294
	632µs
	1.049ms
	exit status 0
	true
	0
```



