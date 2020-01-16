---
title: gin框架分析
date: 2020-01-16 
tags:
 - gin
 - go
---

> 版本说明:gin v1.5.0版本.

## 结构说明


首先看下各文件代码行数:

```
$find ./gin/ -name *.go|grep -v test|xargs wc -l|sort -nr|head -n 10
    5698 total
    1065 ./gin//context.go
     689 ./gin//tree.go
     499 ./gin//gin.go
     350 ./gin//binding/form_mapping.go
     271 ./gin//logger.go
     230 ./gin//routergroup.go
     191 ./gin//render/json.go
     169 ./gin//errors.go
     159 ./gin//ginS/gins.go
 ```
 
关键文件说明如下:
* context.go gin中最重要的部分,管理请求流,控制处理过程中的变量传递以及控制输入和输出
* tree.go 路由Radix Trie,是一个路由前缀树,可以高效进行路由的插入和获取
* gin.go gin入口
* routergroup.go 提供路由包装函数

该四个文件代码行数在整体结构中也是处于top 10.所以拆解源码时可以大致通过代码行数推测其重要性

## 源码实现

接着从源码角度拆解
gin中有三个关键结构体Engine,Context,RouterGroup,分别位于gin.go,context.go以及routergroup.go. 每个结构体的关键属性以及关键方法参考下图

![GinArch.png](/img/GinArch.png)



我们通过如下示例介绍Engine,Context,RrouteGroup之间如何相互关联:

```
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

### 路由定义

gin.Default返回一个Engine类型,Engine类型中有一个匿名结构体RouterGroup,所以r.Get或者r.Post等路由包装函数其实都是委托给RouterGroup类型实现.

而RouterGroup中所有的路由包装函数最终调用的其实是Engine的addRoute函数,addRoute会将路由生成一颗Radix Tree(位于tree.go)

### 服务运行

Engine中的Run方法会启动程序,其实使用的是go http包的ListenAndServe.因此重点看Engine的ServeHttp方法,该方法会调用handleHttpRequest.我们看看该函数的实现:

```
func (engine *Engine) handleHTTPRequest(c *Context) {
    ...
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		value := root.getValue(rPath, c.Params, unescape)
		if value.handlers != nil {
			c.handlers = value.handlers
			c.Params = value.params
			c.fullPath = value.fullPath
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
	    ...
	}

	...
}
```
gin中每种http method有一颗独立的Radix Trie,可以看到关键步骤为通过http method找到相应的路由前缀树,然后从路由前缀树中的相应节点取出handler,最后调用Context的c.Next()依次执行handler.这也就是之前提到的由Context控制请求流程的原因,先依次执行各种中间件,然后执行请求的处理函数

### 控制流

看看Context的Next函数:

```
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```
很简单,middleware和请求处理函数会放到handlers切片中,依次执行即可.具体实现可参考源码

## Trie树

gin框架中比较难于理解的是tree.go中实现的路由前缀树,为了便于理解,可以通过一个示例将Trie树打印出来.

在tree_test.go中增加如下代码:
```
func TestPrintTree(t *testing.T) {
     tree := &node{}

     routes := [...]string{
          "/hi",
          "/contact",
          "/co",
          "/c",
          "/a",
          "/ab",
          "/doc/",
          "/doc/go_faq.html",
          "/doc/go1.html",
          "/cmd/:tool/",
          "/cmd/:tool/:sub",
          "/src/*filepath",
     }
     for _, route := range routes {
          tree.addRoute(route, fakeHandler(route))
     }
     PrintTree(t, tree, 0)
}
func PrintTree(t *testing.T, root *node, indent int) {
     fmt.Printf("%s%+v\n", strings.Repeat(" ", indent), root)
     for _, c := range root.children {
          PrintTree(t, c, indent+10)
     }
}
```
执行该测试用例, 可以将Trie树结构打印.为便于理解,做成下图:

![Trie.png](/img/Trie.png)

途中绿色背景的为有相应handler的路由.可以看到每个父节点的priority就是该父节点及其所有子节点有效路由的总数.每个node的属性以及类型参考下图:
![TrieNode.png](/img/TrieNode.png)

##  结语

gin框架整体比较简洁明了,我觉着最大的价值就是路由.比go原生的http包路由管理更加强大,通过路由的分组管理辅之分组中不同的中间件可以更好的组织大型代码

但在go语言体系中,http框架其实也就是一层封装而已