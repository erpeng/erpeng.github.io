---
title: go bytes包/strings包源码分析
date: 2019-05-23
tags: go
---

## bytes包

### 包函数
```
func Equal(a, b []byte) bool
func Compare(a, b []byte) int
func IndexByte(b []byte, c byte) int
```
如上三个基础函数及其他一些函数分体系结构使用汇编分别编写

go doc bytes可以看到如下一些函数签名:
```
func Compare(a, b []byte) int
func Contains(b, subslice []byte) bool
func ContainsAny(b []byte, chars string) bool
func ContainsRune(b []byte, r rune) bool
func Count(s, sep []byte) int
func Equal(a, b []byte) bool
func EqualFold(s, t []byte) bool
func Fields(s []byte) [][]byte
func FieldsFunc(s []byte, f func(rune) bool) [][]byte
func HasPrefix(s, prefix []byte) bool
func HasSuffix(s, suffix []byte) bool
func Index(s, sep []byte) int
func IndexAny(s []byte, chars string) int
func IndexByte(b []byte, c byte) int
func IndexFunc(s []byte, f func(r rune) bool) int
func IndexRune(s []byte, r rune) int
func Join(s [][]byte, sep []byte) []byte
func LastIndex(s, sep []byte) int
func LastIndexAny(s []byte, chars string) int
func LastIndexByte(s []byte, c byte) int
func LastIndexFunc(s []byte, f func(r rune) bool) int
func Map(mapping func(r rune) rune, s []byte) []byte
func Repeat(b []byte, count int) []byte
func Replace(s, old, new []byte, n int) []byte
func Runes(s []byte) []rune
func Split(s, sep []byte) [][]byte
func SplitAfter(s, sep []byte) [][]byte
func SplitAfterN(s, sep []byte, n int) [][]byte
func SplitN(s, sep []byte, n int) [][]byte
func Title(s []byte) []byte
func ToLower(s []byte) []byte
func ToLowerSpecial(c unicode.SpecialCase, s []byte) []byte
func ToTitle(s []byte) []byte
func ToTitleSpecial(c unicode.SpecialCase, s []byte) []byte
func ToUpper(s []byte) []byte
func ToUpperSpecial(c unicode.SpecialCase, s []byte) []byte
func Trim(s []byte, cutset string) []byte
func TrimFunc(s []byte, f func(r rune) bool) []byte
func TrimLeft(s []byte, cutset string) []byte
func TrimLeftFunc(s []byte, f func(r rune) bool) []byte
func TrimPrefix(s, prefix []byte) []byte
func TrimRight(s []byte, cutset string) []byte
func TrimRightFunc(s []byte, f func(r rune) bool) []byte
func TrimSpace(s []byte) []byte
func TrimSuffix(s, suffix []byte) []byte
```

### Buffer

```
go doc -u bytes.Buffer
type Buffer struct {
	buf       []byte   // contents are the bytes buf[off : len(buf)]
	off       int      // read at &buf[off], write at &buf[len(buf)]
	bootstrap [64]byte // memory to hold first slice; helps small buffers avoid allocation.
	lastRead  readOp   // last read operation, so that Unread* can work correctly.

}

func NewBuffer(buf []byte) *Buffer
func NewBufferString(s string) *Buffer
func (b *Buffer) Bytes() []byte
func (b *Buffer) Cap() int
func (b *Buffer) Grow(n int)
func (b *Buffer) Len() int
func (b *Buffer) Next(n int) []byte
func (b *Buffer) Read(p []byte) (n int, err error)
func (b *Buffer) ReadByte() (byte, error)
func (b *Buffer) ReadBytes(delim byte) (line []byte, err error)
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error)
func (b *Buffer) ReadRune() (r rune, size int, err error)
func (b *Buffer) ReadString(delim byte) (line string, err error)
func (b *Buffer) Reset()
func (b *Buffer) String() string
func (b *Buffer) Truncate(n int)
func (b *Buffer) UnreadByte() error
func (b *Buffer) UnreadRune() error
func (b *Buffer) Write(p []byte) (n int, err error)
func (b *Buffer) WriteByte(c byte) error
func (b *Buffer) WriteRune(r rune) (n int, err error)
func (b *Buffer) WriteString(s string) (n int, err error)
func (b *Buffer) WriteTo(w io.Writer) (n int64, err error)
func (b *Buffer) empty() bool
func (b *Buffer) grow(n int) int
func (b *Buffer) readSlice(delim byte) (line []byte, err error)
func (b *Buffer) tryGrowByReslice(n int) (int, bool)
```

该结构可以用来高效的拼接生成一个字符串.可以读取写入并且按需增长buffer,还可以UnreadByte.


### Reader

```
go doc -u bytes.Reader
type Reader struct {
	s        []byte
	i        int64 // current reading index
	prevRune int   // index of previous rune; or < 0
}
    A Reader implements the io.Reader, io.ReaderAt, io.WriterTo, io.Seeker,
    io.ByteScanner, and io.RuneScanner interfaces by reading from a byte slice.
    Unlike a Buffer, a Reader is read-only and supports seeking.


func NewReader(b []byte) *Reader
func (r *Reader) Len() int
func (r *Reader) Read(b []byte) (n int, err error)
func (r *Reader) ReadAt(b []byte, off int64) (n int, err error)
func (r *Reader) ReadByte() (byte, error)
func (r *Reader) ReadRune() (ch rune, size int, err error)
func (r *Reader) Reset(b []byte)
func (r *Reader) Seek(offset int64, whence int) (int64, error)
func (r *Reader) Size() int64
func (r *Reader) UnreadByte() error
func (r *Reader) UnreadRune() error
func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
```

使用NewReader输入一个byte slice后,可以指定位置读取,逐字节读取,甚至UnreadByte

## strings包
如下命令可以查看各API及功能,不赘述

```
go doc strings
go doc -u strings.Builder
go doc -u strings.Reader
go doc -u strings.Replacer
```
