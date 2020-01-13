---
layout: post
title:  "openresty的agent进程"
date:   2017-05-15 23:39:00 +0800
---

最近Openresty项目增加了一个新的功能，可以在Nginx中开启一个agent进程，这个agent进程不像Nginx的worker进程那样监听服务端口然后对外提供服务，而是继承了master进程的用户权限。

出于安全的考虑，Nginx一般以root身份启动，启动后worker进程通过setuid和setgid以nobody方式运行，只有master进程保留了root身份。而agent进程拥有与master进程相同的权限，便可以实现对Nginx自身的控制，如Nginx的重载等功能。

## 安装

### 通过openresty安装

通过openresty仓库下载安装[openresty](https://github.com/openresty/openresty)

### 从源码编译

- 1 需要对Nginx源码打补丁， openresty的[补丁](https://github.com/openresty/openresty/blob/master/patches/nginx-1.11.2-privileged_agent_process.patch)。
- 2 下载lua-nginx-module，编译Nginx，[具体方法](https://github.com/openresty/lua-nginx-module#installation)
- 3 需要lua-resty-core的支持。

## 开启agent进程

在Nginx中，master进程生成worker进程、cache-manager进程、agent进程等都是在init和init_worker两个阶段之间进行的，要启用agent进程，必须要init_by_lua中进行设置。如下所示开启agent进程。

```
    init_by_lua_block {
        local process = require "ngx.process"
        local ok, err = process.enable_privileged_agent()
        if not ok then
           ngx.log(ngx.ERR, "enable privileged agent failed")
           return
        end
    }
```



## 为agent进程添加任务

Nginx以事件驱动的方式工作，时间的来源主要有IO事件和定时器时间两种。agent进程关闭了监听的端口，无法通过网络IO的方式来驱动，貌似只能通过定时器的方式来实现。

方法是在init_worker_by_lua中，通过ngx.timer.at设置定时器，在定时器中完成相应的工作。

```
init_worker_by_lua_block {
    function do_work()
        while true do
            ngx.log(ngx.ERR, "privileged agent process")
            ngx.sleep(5)
        end
    end    
    local process = require "ngx.process"
    if process.type() == "privileged agent" then
         ngx.timer.at(0, do_work)
     end
}
```

当然直接在init_worker_by_lua中执行工作，不用定时器也可以，只是一直在init_worker阶段，功能受到限制，无法使用ngx.socket，ngx.sleep，ngx.thread等API。

## 利用agent进程实现Nginx自身的控制

如下面的代码，每隔一小时向Nginx的master进程发送重载信号。

```
    lua_package_path "./lib/?.lua;;";
    init_by_lua_block {
        local process = require "ngx.process"
        local ok, err = process.enable_privileged_agent()
        if not ok then
           ngx.log(ngx.ERR, "enable privileged agent failed")
           return
        end
    }

    init_worker_by_lua_block {
        local process = require "ngx.process"
        if process.type() ~= "privileged agent" then
            return
        end

        local function do_work()
            while true do
                ngx.sleep(3600)
                os.execute([[kill -HUP `ps -ef|grep "nginx: master process" |grep -v grep |awk '{print $2}'`]])
            end
        end

        ngx.timer.at(0, do_work)
    }
```

  这里为了方便直接用os.execute执行，实际代码中os.execute效率比较低。

## agent进程实现原理

openresty修改了Nginx源码以实现agent进程的支持。

### 生成agent进程

在ngx_master_process_cycle中调用ngx_start_privileged_agent_processes生成agent进程。

```
static void
ngx_start_privileged_agent_processes(ngx_cycle_t *cycle, ngx_uint_t respawn)
{
    ngx_channel_t          ch;
    ngx_core_conf_t       *ccf;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                           ngx_core_module);

    if (!ccf->privileged_agent) {
        return;
    }

    ngx_spawn_process(cycle, ngx_privileged_agent_process_cycle,
                      "privileged agent process", "privileged agent process",
                      respawn ? NGX_PROCESS_JUST_RESPAWN : NGX_PROCESS_RESPAWN);

    ngx_memzero(&ch, sizeof(ngx_channel_t));

    ch.command = NGX_CMD_OPEN_CHANNEL;
    ch.pid = ngx_processes[ngx_process_slot].pid;
    ch.slot = ngx_process_slot;
    ch.fd = ngx_processes[ngx_process_slot].channel[0];

    ngx_pass_open_channel(cycle, &ch);
}
```

### agent进程执行
生成argent进程后，执行ngx_privileged_agent_process_cycle函数。ngx_privileged_agent_process_cycle会执行ngx_close_listening_sockets关闭所有的监听端口。然后通过ngx_worker_process_init进行一些初始化工作。最后在for循环中和worker进程一样，通过ngx_process_events_and_timers执行相应工作。

```
static void
ngx_privileged_agent_process_cycle(ngx_cycle_t *cycle, void *data)
{
    char   *name = data;

    /*
     * Set correct process type since closing listening Unix domain socket
     * in a master process also removes the Unix domain socket file.
     */
    ngx_process = NGX_PROCESS_HELPER;
    ngx_is_privileged_agent = 1;

    ngx_close_listening_sockets(cycle);

    ngx_worker_process_init(cycle, -1);

    ngx_use_accept_mutex = 0;

    ngx_setproctitle(name);

    for ( ;; ) {

        if (ngx_terminate || ngx_quit) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }

        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }

        ngx_process_events_and_timers(cycle);
    }
}
```

### agent进程初始化
agent执行ngx_worker_process_init进行进程的初始化时，与普通的worker进程主要区别是没有执行setgid和setuid操作，保持了和master进程相同的权限。

如下面的代码所示，agent进程的ngx_is_privileged_agent为1，因此不会执行setgid和setuid。

```
static void
ngx_worker_process_init(ngx_cycle_t *cycle, ngx_int_t worker)
{
      ...............

    if (!ngx_is_privileged_agent && geteuid() == 0) {
        if (setgid(ccf->group) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "setgid(%d) failed", ccf->group);
            /* fatal */
            exit(2);
        }

        if (initgroups(ccf->username, ccf->group) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "initgroups(%s, %d) failed",
                          ccf->username, ccf->group);
        }

        if (setuid(ccf->user) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
                          "setuid(%d) failed", ccf->user);
            /* fatal */
            exit(2);
        }
    }

    .........
}
```

## 总结

- agent进程不监听服务端口，其工作任务需要在init_worker时设置。
- agent继承了master进程的用户权限，可以向master进行发送信号，控制Nginx进行重载，关闭等操作。
- 如果Nginx以root启动，agent进程会拥有root权限，可以控制整个操作系统，使用需非常谨慎。