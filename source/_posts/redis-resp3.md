---
title: Redis resp3协议解析
date: 2019-07-12
tags: Redis
---


## 背景 

redis客户端和服务端通过resp协议进行交互,现在使用的版本是resp2.那么resp2有哪些缺点呢？

* 没有足够的语义表述:例如 lrange,smembers,hgetall在resp2中都会返回一个multi bulk reply(参考resp2返回类型的描述),而实际上这三个命令分别返回一个array,set和map.因此客户端实际处理时需要根据命令来判断到底返回的是什么类型

* 缺乏一些重要的数据类型:例如浮点数和布尔值返回的是一个string和integer类型.

* 没有办法返回一个二进制安全的errors.
现在返回的错误格式为
```
"-Error message\r\n"
```
所以错误信息里边不能有回车或者换行符

除了上述三个已知的缺点,resp3中还会加入一些新的功能,主要也是三点:

* server端主动push数据的功能
* 流式传输功能,即在未完整计算出需要传输的数据长度时就开始传输数据
* 服务端可以返回一些attributes,即跟请求无关但有助于客户端解析返回值的数据

(对比http协议,http2支持了server端push功能;http1.1以后支持chunk传输,不需要先计算content-length;resp协议2和3都是text protocol,和http1.1相同,但http2变为了binary protocol.)

## resp3数据类型

类似于resp2的类型
* Array
* Blob string:二进制安全
```
"$11\r\nhelloworld\r\n"
```
* Simple string
```
"+hello world\r\n"
```
* Simple error
```
"-ERR this is the error description\r\n"
```
* Number :64位有符号类型
```
":1234\r\n"
```

resp3新引入的类型:
* Null :代表resp2中的 *-1\r\n 或者 $-1\r\n
```
_\r\n
```
* Double
```
",1.23\r\n"
```
* Boolean: true or false
```
#t\r\n
#f\r\n
```
* Blob error:二进制安全的error
```
"!21\r\nSYNTAX invalid syntax\r\n"
```
* Verbatim string
```
=15<CR><LF>
txt:Some string<CR><LF>
```
* Map
```
%2<CR><LF>
+first<CR><LF>
:1<CR><LF>
+second<CR><LF>
:2<CR><LF>
```
* Set
```
~5<CR><LF>
+orange<CR><LF>
+apple<CR><LF>
#t<CR><LF>
:100<CR><LF>
:999<CR><LF>
```
* Attribute
```
MGET a b
|1<CR><LF>
    +key-popularity<CR><LF>
    %2<CR><LF>
        $1<CR><LF>
        a<CR><LF>
        ,0.1923<CR><LF>
        $1<CR><LF>
        b<CR><LF>
        ,0.0012<CR><LF>
*2<CR><LF>
    :2039123<CR><LF>
    :9543892<CR><LF>
返回一个属性 
{:key-popularity => {:a => 0.1923, :b => 0.0012}}

```
* Push
```
返回一个pub/sub模式的push
>4<CR><LF>
+pubsub<CR><LF>
+message<CR><LF>
+somechannel<CR><LF>
+this is the message<CR><LF>
```

* Hello:建立连接时发送,服务端会返回服务端名称,版本等等
```
Client: HELLO 3 AUTH default mypassword
Server: -ERR invalid password
(the connection remains in RESP2 mode)
```
* Big number:Number表示不了的大数
```
"(3492890328409238509324850943850943825024385\r\n"
```
* 流式sting传输
```
$EOF:<40 bytes marker><CR><LF>
... any number of bytes of data here not containing the marker ...
<40 bytes marker>
```


## thrift

我们比较一下thrift中binary protocol如何传输一个map

```
 <map> ::= <map-begin> <field-datum>* <map-end>

 <map-begin> ::= <map-key-type> <map-value-type> <map-size>

 <map-key-type> ::= <field-type>

 <map-value-type> ::= <field-type>

 <map-size> ::= I32

 <field-type> ::= T_BOOL | T_BYTE | T_I8 | T_I16 | T_I32 | T_I64 | T_DOUBLE
                     | T_STRING | T_BINARY | T_STRUCT | T_MAP | T_SET | T_LIST
 <field-data> ::= I8 | I16 | I32 | I64 | DOUBLE | STRING | BINARY
                     <struct> | <map> | <list> | <set>
```
依次为 key-type|value-type|map-size|key-value-pairs
key-value-pairs在key-type和value-type类型固定后也就知道应该怎么解析了

thrift中有一个compact protocol,继续以map为例,看下正常binary protocol和compact protocol的区别:

binary protocol:
```
Binary protocol map (6+ bytes) and key value pairs:
+--------+--------+--------+--------+--------+--------+--------+...+--------+
|kkkkkkkk|vvvvvvvv| size                              | key value pairs     |
+--------+--------+--------+--------+--------+--------+--------+...+--------+
```
key-type:1byte
value-type:1byte
size:4byte
key-value pairs

compact protocol:
```
Compact protocol map header (2+ bytes, non empty map) and key value pairs:
+--------+...+--------+--------+--------+...+--------+
| size                |kkkkvvvv| key value pairs     |
+--------+...+--------+--------+--------+...+--------+
```
size:变长int
key-type:4bit
value-type:4bit

可以看到是一个时间换空间的做法,compact protocol会减少编码后的长度,但编解码会消耗更多cpu资源




















