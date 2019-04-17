---
title: Redis module开发
date: 2019-04-17 
tags: Redis
---

## 1.module的作用

redis通过对外提供一套API和一些数据类型, 可以供开发者开发自己的模块并且加载到redis中.通过API可以直接操作redis中的数据,也可以通过调用redis命令来操作数据(类似lua script).
通过编写模块可以注册自己的命令到redis中.

## 2.编写一个module

我们通过编写一个简单的module来体验一下该功能.该module对外提供两个命令,一个是启动一个定时任务,每隔5s将redis持久化相关的信息发送到pinfo这个channel中,另一个是关闭该定时任务.

注册该模块后,我们可以通过"subscribe pinfo"来订阅该渠道,然后就可以定时收到redis持久化相关的信息,以便做一些监控或相应的应对措施 

### 2.1 注册命令到redis中
```
//每个模块都必须有该函数.该函数是redis加载模块的入口,我们通过该函数可以注册相关的命令进去
int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    REDISMODULE_NOT_USED(argv);
    REDISMODULE_NOT_USED(argc);
    //注册模块,模块名称为'pushpersistenceinfo'
    if (RedisModule_Init(ctx,"pushpersistenceinfo",1,REDISMODULE_APIVER_1)
        == REDISMODULE_ERR) return REDISMODULE_ERR;
    //在redis中创建命令.第二部分为命令名称,第三部分为执行该命令时的回调函数
    if (RedisModule_CreateCommand(ctx,"pushpersistenceinfo.timer",
        TimerCommand_RedisCommand,"readonly",0,0,0) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

    if (RedisModule_CreateCommand(ctx,"pushpersistenceinfo.stop",
        TimerStopCommand_RedisCommand,"readonly",0,0,0) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

    return REDISMODULE_OK;
}
```
如上函数类似一个模板,只需要填充自己的模快名称和相应的命令即可.重点是调用RedisModule_CreateCommand时的第三个参数-即回调函数.

### 2.2 定义回调函数
```
#define REDISMODULE_EXPERIMENTAL_API
#include "../redismodule.h"
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <string.h>
#include <errno.h>



RedisModuleString *infoStr;
int off=0 ;//定时器开关
void  getPersistenceStatus(RedisModuleCtx *ctx,RedisModuleString **infoStr);
void  publishPersistenceStatus(RedisModuleCtx *ctx,RedisModuleString **infoStr);

/* Timer callback. */
//时间任务的回调函数
void timerHandler(RedisModuleCtx *ctx,void *data) {
    if(off == 1) return;//如果关闭了定时器,则返回退出
    REDISMODULE_NOT_USED(ctx);
    getPersistenceStatus(ctx,&infoStr);//获取redis持久化相关的信息并放入infoStr中
    publishPersistenceStatus(ctx,&infoStr);//publish redis持久化相关的信息到pinfo渠道
    RedisModule_CreateTimer(ctx,5000,timerHandler,NULL);//创建定时任务,5s后执行

}

int TimerCommand_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    off = 0;
    REDISMODULE_NOT_USED(argv);
    REDISMODULE_NOT_USED(argc);
    RedisModule_AutoMemory(ctx);//开启内存的自动管理
    getPersistenceStatus(ctx,&infoStr);//获取redis持久化相关的信息并放入infoStr中
    publishPersistenceStatus(ctx,&infoStr);//publish redis持久化相关的信息到pinfo渠道

    RedisModule_CreateTimer(ctx,5000,timerHandler,NULL);//创建定时任务,5s后执行.回调函数为timerHandler
    return RedisModule_ReplyWithSimpleString(ctx, "OK");//给客户端返回字符串"OK"
}
int TimerStopCommand_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    off = 1;//关闭定时任务
    return RedisModule_ReplyWithSimpleString(ctx, "OK");//给客户端返回字符串"OK"
}

void  getPersistenceStatus(RedisModuleCtx *ctx,RedisModuleString **infoStr){
    RedisModuleCallReply *reply;
//调用info Persistence获取redis持久化相关的信息
    reply = RedisModule_Call(ctx,"info","c","Persistence");

    *infoStr = RedisModule_CreateStringFromCallReply(reply);
}

void  publishPersistenceStatus(RedisModuleCtx *ctx,RedisModuleString **infoStr){
      //调用publish pinfo xxxxx将持久化信息推送到pinfo渠道

    RedisModule_Call(ctx,"publish","cs","pinfo",*infoStr);
}
```

执行pushpersistenceinfo.timer和pushpersistenceinfo.stop命令后会分别回调TimerCommand_RedisCommand和TimerStopCommand_RedisCommand这两个回调函数.前者会创建一个定时任务,定时任务回调函数为timerHandler,如果off不为1,则回调函数中会再次创建定时任务;后者会将off置为1,不再执行定时任务.

### 2.3 演示
将模块置于redis源码目录的src/modules/目录中,然后执行如下命令编译模块

```
gcc -fPIC -std=gnu99 -c -o pushpersistenceinfo.o pushpersistenceinfo.c
ld -o pushpersistenceinfo.so pushpersistenceinfo.o -shared -Bsymbolic -lc

```

加载模块

```
127.0.0.1:1234> module load /home/xiaoju/redis-5.0.0/src/modules/pushpersistenceinfo.so
OK
127.0.0.1:1234> module list
1) 1) "name"
   2) "pushpersistenceinfo"
   3) "ver"
   4) (integer)
```

执行命令(首先订阅pinfo渠道)
```
redis-cli -p 1234 subscribe pinfo
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "pinfo"
3) (integer) 1
```
执行模块中的命令

```
127.0.0.1:1234> PUSHPERSISTENCEINFO.timer
OK
```

查看输出(可以看到,每隔5s会输出一次)
```
1) "message"
2) "pinfo"
3) "# Persistence\r\nloading:0\r\nrdb_changes_since_last_save:15\r\nrdb_bgsave_in_progress:0\r\nrdb_last_save_time:1555494610\r\nrdb_last_bgsave_status:ok\r\nrdb_last_bgsave_time_sec:-1\r\nrdb_current_bgsave_time_sec:-1\r\nrdb_last_cow_size:0\r\naof_enabled:1\r\naof_rewrite_in_progress:0\r\naof_rewrite_scheduled:0\r\naof_last_rewrite_time_sec:-1\r\naof_current_rewrite_time_sec:-1\r\naof_last_bgrewrite_status:ok\r\naof_last_write_status:ok\r\naof_last_cow_size:0\r\naof_current_size:631\r\naof_base_size:631\r\naof_pending_rewrite:0\r\naof_buffer_length:0\r\naof_rewrite_buffer_length:0\r\naof_pending_bio_fsync:0\r\naof_delayed_fsync:0\r\n"
1) "message"
2) "pinfo"
3) "# Persistence\r\nloading:0\r\nrdb_changes_since_last_save:15\r\nrdb_bgsave_in_progress:0\r\nrdb_last_save_time:1555494610\r\nrdb_last_bgsave_status:ok\r\nrdb_last_bgsave_time_sec:-1\r\nrdb_current_bgsave_time_sec:-1\r\nrdb_last_cow_size:0\r\naof_enabled:1\r\naof_rewrite_in_progress:0\r\naof_rewrite_scheduled:0\r\naof_last_rewrite_time_sec:-1\r\naof_current_rewrite_time_sec:-1\r\naof_last_bgrewrite_status:ok\r\naof_last_write_status:ok\r\naof_last_cow_size:0\r\naof_current_size:631\r\naof_base_size:631\r\naof_pending_rewrite:0\r\naof_buffer_length:0\r\naof_rewrite_buffer_length:0\r\naof_pending_bio_fsync:0\r\naof_delayed_fsync:0\r\n"
```

停止定时器

```
127.0.0.1:1234> PUSHPERSISTENCEINFO.stop
OK
```

卸载模块

```
127.0.0.1:1234> module unload pushpersistenceinfo
OK
```

## 3 参考文档
如上代码地址:https://github.com/erpeng/redis-modules
* https://redislabs.com/blog/writing-redis-modules/ 
* https://redis.io/topics/modules-intro
* https://redis.io/topics/modules-api-ref
* https://redis.io/topics/modules-native-types