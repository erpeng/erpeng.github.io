---
title: ffmpeg笔记
date: 2020-09-03 
tags:
 - ffmpge
---



## Components

组件描述
* libavutil:描述时间、视频大小、帧率、码率、颜色、渠道语法格式
	* 包括字符串处理函数、随机数生成器、数学函数、加密函数
* libswscale:图形变换，图片rescaling和像素格式变换
	* 视频rescaling:改变视频大小
	* 改变图片格式和颜色空间
* libswresample:音频重采样
	* 音频48KHz降采到8KHz
	* 多声道到单声道
* libavcodec:编解码
	* Bitstream Filters:执行bitstream级别的修改,但不执行解码
* libavformat:混流与分流.支持多种不同协议的读取,例如rtmp,hls
	* 解封和封装一些音视频容器格式
* libavdevice:类似mux和demux,从不同的输入设备demux,然后到输出设备mux
* libavfilter:

## 命令行

### 语法:
https://ffmpeg.org/ffmpeg.html#Stream-specifiers-1
```
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...
```
* 所有在命令行中无法解释为option的都当输出考虑
* -map选项制定输入和输出的对应关系
* 选项中制定输入文件可以用从0开始的index制定,stream也是从0开始指定
* copy选项说明不需要进行decode和encode,适用于改变容器类型

* stream specifiers:
	* -codec:a:1 ac3 第二个输入文件的audio使用ac3 codec
	* stream_index 流索引
	* stream_type[:additional_stream_specifier] v:video,a:audio,s:subtitle,d:data,t:attachments
	* p:program_id[:additional_stream_specifier]
	* #stream_id or i:stream_id PID in MPEG-TS container
	* m:key[:value] 匹配metadata tag中有特定key和值的流
	* u
* 通用命令
```
ffmpge -codecs 列出可用codec
ffmpeg -muxers 列出可用muxers
```



