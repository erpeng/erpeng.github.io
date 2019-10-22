---
title: OpenTracing and OpenCensus-2
date: 2019-10-22
tags:
 - go
 - trace
 - metric
---

>上一篇着重介绍OpenTracing spec相关内容.本章通过一些实例介绍OpenTracing.

## 说明
参考https://github.com/yurishkuro/opentracing-tutorial/tree/master/go
注意该代码仓库中引入jaeger-client-go使用的路径是"github.com/uber/jaeger-client-go/config",而jaeger已经托管到了jaegertracing仓库,下载之后修改目录名或者修改import导入路径名均可
使用Jaeger作为后端,首先启动一个Jaeger的docker镜像
```
docker run \
  --rm \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 16686:16686 \
  jaegertracing/all-in-one:1.7 \
  --log-level=debug
```

通过浏览器访问http://localhost:16686 查看

## 示例1
为代码库中lesson2.单体程序
```
go run hello.go ke.com
```
```
func main() {
	if len(os.Args) != 2 {
		panic("ERROR: Expecting one argument")
	}

	tracer, closer := tracing.Init("hello-world") //定义一个hello-world的service
	defer closer.Close()
	opentracing.SetGlobalTracer(tracer)

	helloTo := os.Args[1]

	span := tracer.StartSpan("say-hello")  //开启一个名称为say-hello的span
	span.SetTag("hello-to", helloTo) //设置一个tag,名称为"hello-to",value为运行时的参数.UI界面可以根据tag进行检索
	defer span.Finish()

	ctx := opentracing.ContextWithSpan(context.Background(), span)//通过context将span中的traceId,spanId传递

	helloStr := formatString(ctx, helloTo)
	printHello(ctx, helloStr)
}

func formatString(ctx context.Context, helloTo string) string {
	span, _ := opentracing.StartSpanFromContext(ctx, "formatString") //首先将span extract
	defer span.Finish()

	helloStr := fmt.Sprintf("Hello, %s!", helloTo)
	span.LogFields(
		log.String("event", "string-format"),
		log.String("value", helloStr),
	)  //设置span中的日志字段

	return helloStr
}

func printHello(ctx context.Context, helloStr string) {
	span, _ := opentracing.StartSpanFromContext(ctx, "printHello") //将span extract
	defer span.Finish()

	println(helloStr)
	span.LogKV("event", "println") //设置日志属性
}
```
看看UI显示:
![总览](/img/oc1.png)
左侧检索方式解释如下(具体值参考代码):
* Service:本例中为hello-world
* Operation:本例中为say-hello,printHello,formatString
* Tags:写法为
```
https.status_code=200 error=true
```
本例中为hello-to=ke.com
* Lookback:回溯时间 Min Duration:最短执行时长 Max Duration:最长执行时长 Limit Resuls:显示条数

![详情](/img/oc2.png)
* 执行时间轴
* 每个operation具体的logs field以及执行时间


## 示例2
为lesson4,调用http请求,为微服务模式.如下执行
```
go run hello.go ke.com baggage
```
代码如下:
```
func main() {
	if len(os.Args) != 3 {
		panic("ERROR: Expecting two arguments")
	}

	tracer, closer := tracing.Init("hello-world")
	defer closer.Close()
	opentracing.SetGlobalTracer(tracer)

	helloTo := os.Args[1]
	greeting := os.Args[2]

	span := tracer.StartSpan("say-hello")
	span.SetTag("hello-to", helloTo)
	span.SetBaggageItem("greeting", greeting) //设置baggage
	defer span.Finish()

	ctx := opentracing.ContextWithSpan(context.Background(), span)

	helloStr := formatString(ctx, helloTo)
	printHello(ctx, helloStr)
}

func formatString(ctx context.Context, helloTo string) string {
	span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
	defer span.Finish()

	v := url.Values{}
	v.Set("helloTo", helloTo)
	url := "http://localhost:8081/format?" + v.Encode()
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		panic(err.Error())
	}

	ext.SpanKindRPCClient.Set(span) //ext包中封装语法糖,实际仍为set tag.本句为设置spankind为client
	ext.HTTPUrl.Set(span, url) //设置span httpurl
	ext.HTTPMethod.Set(span, "GET") //设置span method
	span.Tracer().Inject( //以http header为载体,将span上下文传递.
		span.Context(),
		opentracing.HTTPHeaders,
		opentracing.HTTPHeadersCarrier(req.Header),
	)

	resp, err := xhttp.Do(req)
	if err != nil {
		panic(err.Error())
	}

	helloStr := string(resp)

	span.LogFields(
		log.String("event", "string-format"),
		log.String("value", helloStr),
	)

	return helloStr
}
...
```
看一下其中一个http服务:
```
func main() {
	tracer, closer := tracing.Init("formatter")
	defer closer.Close()

	http.HandleFunc("/format", func(w http.ResponseWriter, r *http.Request) {
		spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
        //从http头中将span上下文取出
		span := tracer.StartSpan("format", ext.RPCServerOption(spanCtx))
		defer span.Finish()

		greeting := span.BaggageItem("greeting") //取出baggage
		if greeting == "" {
			greeting = "Hello"
		}

		helloTo := r.FormValue("helloTo")
		helloStr := fmt.Sprintf("%s, %s!", greeting, helloTo)
		span.LogFields(
			otlog.String("event", "string-format"),
			otlog.String("value", helloStr),
		)
		w.Write([]byte(helloStr))
	})

	log.Fatal(http.ListenAndServe(":8081", nil))
}
```
具体示意图可自行实验查看

## 参考连接:

* https://opentracing.io/specification/
* https://opentracing.io/specification/conventions/
* https://github.com/yurishkuro/opentracing-tutorial/tree/master/go
* https://zhuanlan.zhihu.com/p/34318538
* http://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html
