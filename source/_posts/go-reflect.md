---
title: Go语言反射
date: 2022-04-29
tags: Go
---
> Go语言版本1.17.6


## 入口

reflect.TypeOf和reflect.ValueOf两个函数会分别返回Type和Value两个结构，顾名思义，分别代表一个Go数据的类型和值。通过查看Type和Value的方法可以得出反射可以实现的功能。
```
type Value struct {
	typ *rtype
	ptr unsafe.Pointer
	flag
}
```
Value结构体中也包括Type，因此可以通过
```
(v Value) Type() Type
```
方法直接返回Type。Value结构体中ptr指向的是实际数据


## 结构体

Type接口中关于结构体有如下几个方法：

```
type StructField struct {
	Name string
	PkgPath string
	Type      Type      // field type
	Tag       StructTag // field tag string
	Offset    uintptr   // offset within struct, in bytes
	Index     []int     // index sequence for Type.FieldByIndex
	Anonymous bool      // is an embedded field
}


Field(i int) StructField
FieldByIndex(index []int) StructField
FieldByName(name string) (StructField, bool)
FieldByNameFunc(match func(string) bool) (StructField, bool)
NumField() int
```
结构体通过TypeOf函数之后实际内存布局为如下结构：
```
type structType struct {
	rtype
	pkgPath name
	fields  []structField // sorted by offset
}
```
其中rtype为实现了Type接口的结构体，pkgPath为包名，fields为一个包括所有结构体字段的切片，因此NumField()返回len(fields)即可。同理，Field(i int)根据索引查找该字段并且生成一个StructField结构并且返回

继续查看Value类型中对struct的操作
```
func (v Value) Field(i int) Value {
	if v.kind() != Struct {
		panic(&ValueError{"reflect.Value.Field", v.kind()})
	}
	tt := (*structType)(unsafe.Pointer(v.typ))
	if uint(i) >= uint(len(tt.fields)) {
		panic("reflect: Field index out of range")
	}
	field := &tt.fields[i]
	typ := field.typ

	...
	ptr := add(v.ptr, field.offset(), "same as non-reflect &v.field")
	return Value{typ, ptr, fl}
}
```
通过值加字段偏移量并且生成一个新的Value然后返回

对struct的反射操作示例，通过json.Marshal看看如何将一个结构体转换为json，重点函数如下：
```
func newStructEncoder(t reflect.Type) encoderFunc {
	se := structEncoder{fields: cachedTypeFields(t)}
	return se.encode
}
```
生成一个struct的encoderFunc，通过对一个结构体的反射，遍历机构体，将json tag取出，然后通过如下函数生成json
```
func (se structEncoder) encode(e *encodeState, v reflect.Value, opts encOpts) {
	next := byte('{') 
FieldLoop:
	for i := range se.fields.list {
		f := &se.fields.list[i]
		fv := v
		for _, i := range f.index {
			if fv.Kind() == reflect.Ptr {
				if fv.IsNil() {
					continue FieldLoop
				}
				fv = fv.Elem()
			}
			fv = fv.Field(i) //调用Value的Field方法获取具体的值
		}

		if f.omitEmpty && isEmptyValue(fv) {
			continue
		}
		e.WriteByte(next) //输出json最外层大括号
		next = ','
		if opts.escapeHTML {
			e.WriteString(f.nameEscHTML)
		} else {
			e.WriteString(f.nameNonEsc)
		}
		opts.quoted = f.quoted
		f.encoder(e, fv, opts) //根据结构体的每个字段，分别调用该字段的编码函数
	}
	if next == '{' {
		e.WriteString("{}")
	} else {
		e.WriteByte('}')
	}
}
// 如果字段为int，则通过调用Value的Int方法转为int类型然后写入json。
func intEncoder(e *encodeState, v reflect.Value, opts encOpts) {
	b := strconv.AppendInt(e.scratch[:0], v.Int(), 10)
	if opts.quoted {
		e.WriteByte('"')
	}
	e.Write(b)
	if opts.quoted {
		e.WriteByte('"')
	}
}
```

## 方法

一个struct如果有方法，则内存布局变为：
```
type structTypeUncommon struct {
	structType
	u uncommonType
}
type uncommonType struct {
	pkgPath nameOff // import path; empty for built-in types like int, string
	mcount  uint16  // number of methods
	xcount  uint16  // number of exported methods
	moff    uint32  // offset from this uncommontype to [mcount]method
	_       uint32  // unused
}
```
一个结构体的Value值加上uncommonType中的moff就是结构体的方法数组。Value的MethodByName通过一个方法名称可以返回一个方法
```
func (t *rtype) MethodByName(name string) (m Method, ok bool) {
	...
	ut := t.uncommon()
	...
	for i, p := range ut.exportedMethods() {
		if t.nameOff(p.name).name() == name {
			return t.Method(i), true
		}
	}
	return Method{}, false
}
```
然后通过调用Call可以调用该方法，Call签名如下：
```
func (v Value) Call(in []Value) []Value {
	v.mustBe(Func)
	v.mustBeExported()
	return v.call("Call", in)
}
```
我们看一个通过反射实现继承的例子：
```
package main

import "fmt"
import "reflect"

type T struct {}

func (t *T) Geeks() {
        fmt.Println("parent")
}

type T1 struct {
        T
}

func (t *T1) Geeks() {
        fmt.Println("child 1")
}


type T2 struct {
        T
}

func main() {
        var t T
        var t1 T1
        var t2 T2
        fmt.Println(invoke(&t))
        fmt.Println(invoke(&t1))
        fmt.Println(invoke(&t2))
}

func invoke( v interface{}) []reflect.Value {
        val := reflect.ValueOf(v).MethodByName("Geeks").Call([]reflect.Value{})
        return val
}
```
例子中相当于T1、T2继承了T，T1重载了Geeks方法，T2使用父类的方法

## 总结

文中介绍了结构体的反射，其他类型未详细描述，需要可以自行查询文档或者代码。
关于需要反射的其他场景，可以参考如下文章：
https://golangbot.com/reflection/