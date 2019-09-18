---
title: NGINX 平滑升级
date: 2019-09-17
tags: NGINX
---

>平滑升级或者优雅升级或者优雅重启的目的是不影响当前正在执行请求的情况下实现升级或者重启

## 平滑升级步骤

* 备份旧的二进制文件,然后将新的二进制文件覆盖旧的二进制文件
* 向nginx master pid发送USR2信号
    ```
    kill -USR2 nginx-master-pid
    ```
    此时nginx首先会将保存pid值的nginx.pid rename为nginx.pid.oldbin,然后使用新的二进制文件开启nginx
    此时会有两组master和两组worker同时提供服务
* 向旧的nginx master pid发送WINCH信号
    ```
    kill -WINCH nginx-master-pid
    ```
    旧的master收到该信号后会向旧的worker发送信息,要求他们优雅关闭-即处理完请求后退出

* 此时有两种情况,如果新的nginx工作正常,那么发送QUIT信号给旧的master,升级完毕
  如果新的nginx工作不正常,那么可以发送HUP信号给旧的master,旧master会重新启动worker而且不会重读配置(即保持旧有的配置不变),然后发送QUIT信号给新的master要求退出,也可以直接发送TERM命令给新master,新master退出后旧master会重启worker process.


## 实现原理
nginx版本1.17.3

我们从源码层面看一下USER2信号和WINCH信号以及QUIT信号,nginx master如何处理,最后总结一下如何实现优雅重启或者优雅升级

可以先思考一下一些难点:
* 在操作系统层面是不允许两个进程监听同一个端口的,那么优雅升级时如何让新旧进程同时处理监听端口上的请求
* 如何让旧的worker process 不再接收请求,并且处理完当前请求后退出

nginx中信号定义宏如下:

```
#define NGX_REOPEN_SIGNAL        USR1
#define NGX_CHANGEBIN_SIGNAL     USR2
#define NGX_SHUTDOWN_SIGNAL      QUIT
#define NGX_TERMINATE_SIGNAL     TERM
#define NGX_NOACCEPT_SIGNAL      WINCH
#define NGX_RECONFIGURE_SIGNAL   HUP
```

### USR2信号

我们以NGX_CHANGEBIN_SIGNAL全局搜索代码,可以搜到信号处理函数为ngx_signal_handler,接受到USR2信号后处理逻辑如下:
```
        case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
            ...
            ngx_change_binary = 1;
            action = ", changing binary";
            break;
```

继续按ngx_change_binary搜索,可以看到在ngx_master_process_cycle中执行主要逻辑:
```

        if (ngx_change_binary) {
            ngx_change_binary = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "changing binary");
            ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
        }
```
ngx_exec_new_binary函数如下:
```
ngx_pid_t
ngx_exec_new_binary(ngx_cycle_t *cycle, char *const *argv)
{
    ...
    p = ngx_cpymem(var, NGINX_VAR "=", sizeof(NGINX_VAR));

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        p = ngx_sprintf(p, "%ud;", ls[i].fd);
    }

    *p = '\0';
    ...
}
```
比较核心的点是导出所有监听句柄到NGINX=1;2;3这种变量中,另一点是重命名pid文件.接着执行execve生成新的二进制程序

那么如何处理导出的变量呢,看看NGINX的启动过程

```
static ngx_int_t
ngx_add_inherited_sockets(ngx_cycle_t *cycle)
{
    ...
    inherited = (u_char *) getenv(NGINX_VAR);

    if (inherited == NULL) {
        return NGX_OK;
    }
    ...
}
```
在该函数中会将导出的句柄全部初始化到cycle->listening中.注意使用execve时除非指定close_on_exec否则新的进程会继承旧的所有file descriptors,然后通过正常的抢锁流程即可处理请求


### WINCH信号

同理,继续查看主流程代码:

```
        case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
            if (ngx_daemonized) {
                ngx_noaccept = 1;
                action = ", stop accepting connections";
            }
            break;
```

```
   if (ngx_noaccept) {
            ngx_noaccept = 0;
            ngx_noaccepting = 1;
            ngx_signal_worker_processes(cycle,
                                        ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
```

```
static void
ngx_signal_worker_processes(ngx_cycle_t *cycle, int signo){
    ...
        case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
        ch.command = NGX_CMD_QUIT;
        break;
    ...
        if (ch.command) {
            if (ngx_write_channel(ngx_processes[i].channel[0],
                                  &ch, sizeof(ngx_channel_t), cycle->log)
                == NGX_OK)
            {
                if (signo != ngx_signal_value(NGX_REOPEN_SIGNAL)) {
                    ngx_processes[i].exiting = 1;
                }

                continue;
            }
        }
    ...
}
```
向worker管道中发送NGX_CMD_QUIT命令,并且置exiting为1

```
static void
ngx_channel_handler(ngx_event_t *ev)
{
        ...
        switch (ch.command) {

        case NGX_CMD_QUIT:
            ngx_quit = 1;
            break;
        }
        ...
}
```

```
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ...

        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                          "gracefully shutting down");
            ngx_setproctitle("worker process is shutting down");

            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);//关闭监听句柄
                ngx_close_idle_connections(cycle);
            }
        }
    ...
}
```
关键节点是关闭监听句柄,这样有新的请求时不会分配到旧的worker.
**Q:如何保证下次不会再次抢锁并且处理请求**

### QUIT信号
收到QUIT信号后master首先shutdown子进程,并关闭监听句柄

##  小结
对应于上文的两个关键问题,解决方法如下:
* 以环境变量的形式导出文件句柄,由于execve会继承句柄上下文,所以可以同时监听处理连接
* 关闭监听套接字后旧的进程不会继续处理请求

