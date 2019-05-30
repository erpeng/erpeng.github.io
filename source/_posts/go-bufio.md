---
title: go io与bufio包分析
date: 2019-05-30
tags: go
---
>go中io处理相关的包

## io包
```
go doc -u io.Reader
type Reader interface {
	Read(p []byte) (n int, err error)
}
func LimitReader(r Reader, n int64) Reader
func MultiReader(readers ...Reader) Reader
func TeeReader(r Reader, w Writer) Reader

go doc io.Writer
type Writer interface {
	Write(p []byte) (n int, err error)
}
func MultiWriter(writers ...Writer) Writer
```
之前介绍的os.File结构体实现了Read和Write函数,可以作为Reader和Writer使用

```
type multiReader struct {
	readers []Reader
}
```
multiReader与multiWriter就是一个Reader和Writer的切片,定义了自己的Read和Write方法(对切片中的每个元素分别进行Read与Write)

LimitReader可以限定只读取n个字节

TeeReader会从r中读取数据后再写入w

```
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
type Closer interface {
	Close() error
}

```
Reader,Writer,Seeker,Closer四个接口可以交互组合,例如:

```
type ReadWriteSeeker interface {
	Reader
	Writer
	Seeker
}

```
Copy从src复制到dst,直到遇到EOF或者error
```
func Copy(dst Writer, src Reader) (written int64, err error)
```

## bufio包

```
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)

func ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)
```
ScanBytes逐字节读取
ScanLines根据"\r\n"或者"\n"依次取出每行,数据来源为data,advance为下个索引位置
ScanRunes逐rune读取
ScanWords按空格分隔读取

```
type Scanner struct {
	r            io.Reader // The reader provided by the client.
	split        SplitFunc // The function to split the tokens.
	maxTokenSize int       // Maximum size of a token; modified by tests.
	token        []byte    // Last token returned by split.
	buf          []byte    // Buffer used as argument to split.
	start        int       // First non-processed byte in buf.
	end          int       // End of data in buf.
	err          error     // Sticky error.
	empties      int       // Count of successive empty tokens.
	scanCalled   bool      // Scan has been called; buffer is in use.
	done         bool      // Scan has finished.
}
func NewScanner(r io.Reader) *Scanner
func (s *Scanner) Buffer(buf []byte, max int)
func (s *Scanner) Bytes() []byte
func (s *Scanner) Err() error
func (s *Scanner) Scan() bool
func (s *Scanner) Split(split SplitFunc)
func (s *Scanner) Text() string
func (s *Scanner) advance(n int) bool
func (s *Scanner) setErr(err error)
```

Scanner的Split可以设置SplitFunc,不设置时默认为ScanLines.Scan方法会按SplitFunc读取数据并放置到Scanner的token字段中.Text方法将token字段中数据转为string并返回,Bytes方法返回字节序列.



