---
layout: post
title:  "ngx_lua中的协程调度(四)"
date:   2017-03-02 22:57:00 +0800
tags: Openresty Nginx Lua 协程
categories: Openresty
---

## ngx_lua中访问多个第三方服务

ngx_lua中提供了ngx.socket API，可以方便的访问第三方网络服务。如下面的代码,通过get_response函数从两个（或者更多）的源服务器获取数据，再生成响应发给客户端。

```lua
location / {
    content_by_lua_block {
        local get_response(host, port)
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect(host, port)
            if not ok then
                return nil, err
            end
            local data, err = sock:receive()
            if not data then
                return nil, err
            end

            return data
        end

        local first = get_response("lua.org", 8080)
        local second = get_response("nginx.org", 8080)
        ngx.say(first .. second)
    }
}
```

如果需要10个第三方网络服务，需要调用get_response 10次。总的响应时间与需要连接源的数量成正比。那么如何缩短源的响应时间呢？ngx.thread就是用来解决这种问题的。

## ngx.thread API

lua-nginx-module提供了三个API
* ngx.thread.spawn
* ngx.thread.wait
* ngx.thread.kill

关于API的介绍可以参考[官方文档](https://github.com/openresty/lua-nginx-module#ngxthreadspawn)。

通过ngx.thread.spawn可以生成一个"light thread"，一个”light thread“和Lua的协程类似，区别在于"light thread"是由ngx_lua模块进行调度的，多个"light thread"同时运行。

## "light thread"，协程 和 进程

"light thread"比Lua中的协程更像操作系统中的进程。
* fork生成新的进程，生成的多个进程可以同时运行，而ngx.thread.spawn生成新的协程，多个协程同时在跑。
* kill可以杀死不需要的子进程，ngx.thread.kill可以杀死不需要的"light thread"
* wait可以等待子进程结束并取得子进程退出状态，ngx.thread.wait可以等待"light thread"结束并获取其返回值。

## ngx.thread的使用
用ngx.thread重写上面的代码
```lua
location / {
    content_by_lua_block {
        local get_response(host, port)
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect(host, port)
            if not ok then
                return nil, err
            end
            local data, err = sock:receive()
            if not data then
                return nil, err
            end

            return data
        end

        local t1 = ngx.thread.spawn(get_response, "lua.org", 8080)
        local t2 = ngx.thread.spawn(get_response, "nginx.org", 8080)
        local ok, res1, res2 = ngx.thread.wait(t1, t2)
        ngx.say(res1 .. res2)
    }
}
```
生成的两个"light thread"可以同时运行，总的耗时只相当于访问一个源服务器的时间，即使需要访问的源服务器增加，耗时没有太大的变化。

## "light thread"的调度

Linux中的fork生成新的子进程，父进程与子进程谁先运行呢？都有可能，和系统的调度有关。

把调用ngx.thread.spawn的这个Lua协程称为父协程，生成的"light thread"和父协程谁先运行呢? 在ngx_lua的调度逻辑中，是生成的"light thread"先运行，运行结束或者被挂起后，父协程才会继续运行。实际的代码在ngx_http_lua_run_thread函数中，这个函数比较复杂，涉及的东西太多，稍后再细说。

## "light thread"的限制

"light thread"毕竟是基于依附于请求的，如在content_by_lua中创建的"light thread"，是完全与当前的请求关联的，如果"light thread"没有退出，当前请求也无法结束。同样如果当前请求因为错误退出，或调用ngx.exit强制退出时，处于运行状态的"light thread"也会被kill掉。不像操作系统的进程，父进程退出后，子进程可以被init进程"收养"。

如下面的代码，没有调用ngx.thread.wait去等待"light thread"的结束。
```lua
        location / {
            content_by_lua_block {
                local function f(name)
                    ngx.log(ngx.ERR, "thread name: ", name, ", now start")
                    ngx.sleep(4)
                    ngx.log(ngx.ERR, "thread name: ", name, ", now end")
                end
                
                local t1 = ngx.thread.spawn(f, "first")
                local t2 = ngx.thread.spawn(f, "second")
                ngx.log(ngx.ERR, "main thread end")
            }
        }

```
由Nginx的日志中可以看到当前的请求一直延迟到t1,t2两个"light thread"最后退出才会结束。
Nginx中日志的顺序也可以看出父协程和两个"light thread"的执行那个顺序。
```shell
2017/03/02 22:43:21 [error] 2142#0: *1 [lua] content_by_lua(nginx.conf:55):3: thread name: first, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:43:21 [error] 2142#0: *1 [lua] content_by_lua(nginx.conf:55):3: thread name: second, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:43:21 [error] 2142#0: *1 [lua] content_by_lua(nginx.conf:55):10: main thread end, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:43:25 [error] 2142#0: *1 [lua] content_by_lua(nginx.conf:55):5: thread name: first, now end, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:43:25 [error] 2142#0: *1 [lua] content_by_lua(nginx.conf:55):5: thread name: second, now end, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
```
而如果代码中主动调用了ngx.exit()结束请求，那么t1，t2两个没有打印出完全的信息就被kill掉了。
```lua
location / {
    content_by_lua_block {
        local get_response(host, port)
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect(host, port)
            if not ok then
                return nil, err
            end
            local data, err = sock:receive()
            if not data then
                return nil, err
            end

            return data
        end

        local t1 = ngx.thread.spawn(get_response, "lua.org", 8080)
        local t2 = ngx.thread.spawn(get_response, "nginx.org", 8080)
        ngx.exit(100)
    }
}
```
相应的Nginx日志
```
2017/03/02 22:48:01 [error] 2227#0: *1 [lua] content_by_lua(nginx.conf:56):3: thread name: first, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:48:01 [error] 2227#0: *1 [lua] content_by_lua(nginx.conf:56):3: thread name: second, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:48:01 [error] 2227#0: *1 [lua] content_by_lua(nginx.conf:56):10: main thread end, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:48:35 [error] 2227#0: *2 [lua] content_by_lua(nginx.conf:56):3: thread name: first, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:48:35 [error] 2227#0: *2 [lua] content_by_lua(nginx.conf:56):3: thread name: second, now start, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
2017/03/02 22:48:35 [error] 2227#0: *2 [lua] content_by_lua(nginx.conf:56):10: main thread end, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", host: "127.0.0.1"
```