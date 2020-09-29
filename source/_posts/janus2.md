---
title: Janus笔记-2
date: 2020-09-28
tags:
 - Janus
---

> Janus是一个开源的通用WebRTC server

## 安装

CentOS依赖安装:
```
yum install libmicrohttpd-devel jansson-devel \
   openssl-devel libsrtp-devel sofia-sip-devel glib2-devel \
   opus-devel libogg-devel libcurl-devel pkgconfig gengetopt \
   libconfig-devel libtool autoconf automake meson doxygen graphviz

```

nice:
```
git clone https://gitlab.freedesktop.org/libnice/libnice
cd libnice
meson --prefix=/usr build && ninja -C build && sudo ninja -C build install
```

libsrtp:
```
wget https://github.com/cisco/libsrtp/archive/v1.5.4.tar.gz
tar xfv v1.5.4.tar.gz
cd libsrtp-1.5.4
./configure --prefix=/usr --enable-openssl --libdir=/usr/lib64
make shared_library && sudo make install
```

usrsctp

```
git clone https://github.com/sctplab/usrsctp
cd usrsctp
./bootstrap
./configure --prefix=/usr && make && sudo make install
```

libwebsockets:
```
git clone https://github.com/warmcat/libwebsockets.git
cd libwebsockets
# If you want the stable version of libwebsockets, uncomment the next line
# git checkout v3.2-stable
mkdir build
cd build
# See https://github.com/meetecho/janus-gateway/issues/732 re: LWS_MAX_SMP
cmake -DLWS_MAX_SMP=1 -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" ..
make && sudo make install
```

mqtt:
```
git clone https://github.com/eclipse/paho.mqtt.c.git
cd paho.mqtt.c
make && sudo make install
```

## 编译
```
git clone https://github.com/meetecho/janus-gateway.git
cd janus-gateway

```

```
sh autogen.sh
./configure --prefix=/opt/janus -disable-data-channels --disable-websockets --disable-rabbitmq 
make
make install
make configs

```



