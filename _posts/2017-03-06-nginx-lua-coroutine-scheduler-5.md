---
layout: post
title:  "ngx_lua的协程调度(五)之ngx_http_lua_run_thread"
date:   2017-03-06 22:46:00 +0800
tags: Openresty Nginx Lua 协程
categories: Openresty
---

lua-nginx-module中，在Log，Header_filter，Body_filter几个阶段直接调用lua_pcall执行Lua脚本，而在Access，Rewrite，Content等阶段Lua脚本的执行是在ngx_http_lua_run_thread函数中调用lua_resume实现的。再根据lua_resume的返回值进行处理。

# lua_resume函数

```c
int lua_resume (lua_State *L, int narg);
```

这是lua_resume的声明，按照返回值可分为几种情况
* LUA_YIELD: 协程yield
* 0: 协程执行结束
* 其他: 运行出错，如内存不足等


# 协程yield的处理

导致协程yield的API主要分以下几种

## ngx.exit, ngx.exec和ngx.redirect

这里还是可以拿C语言来比较下，C语言中return从当前的函数返回，而exit直接返回到顶层，导致进程的结束。在Lua中return也是从函数返回，但是Lua没有提共exit类似的方法，如果函数调用层次太深，需要结束协程的运行就变得很困难，lua-nginx-module中是通过协程的yield来实现类似的功能。lua_resume返回1时判断协程是否可以直接结束。

ngx.exit，ngx.exec和ngx.redirect都采用了这种方法。

```c
                if (r->uri_changed) {
                    return ngx_http_lua_handle_rewrite_jump(L, r, ctx);
                }

                if (ctx->exited) {
                    return ngx_http_lua_handle_exit(L, r, ctx);
                }

                if (ctx->exec_uri.len) {
                    return ngx_http_lua_handle_exec(L, r, ctx);
                }
```

* r->uri_changed为true表明调用了ngx.redirect
* ctx->exited为true表明调用了ngx.exit
* ctx->exec_uri.len为true表明调用了ngx.exec

这三种情况都不需要协程继续运行了，退出执行相应的处理

## ngx.socket和ngx.sleep

前面介绍了这两个API需要异步的执行，导致协程yield，这时ngx_http_lua_run_thread返回NGX_AGAIN，等到socket或定时器事件触发后继续运行。
ctx->co_op代表协程yield的原因，其值为NGX_HTTP_LUA_USER_CORO_NOP时表明是由于ngx.socket或者ngx.sleep导致的。

```c
                switch(ctx->co_op) {
                case NGX_HTTP_LUA_USER_CORO_NOP:
                    dd("hit! it is the API yield");

                    ngx_http_lua_assert(lua_gettop(ctx->cur_co_ctx->co) == 0);

                    ctx->cur_co_ctx = NULL;

                    return NGX_AGAIN;
```

## ngx.thread

之前提到过ngx.thread.spawn生成新的"light thread"时的执行顺序，新的"light thread"会先执行。这个逻辑如何实现呢？在ngx.thread.spawn中生成新的"light thread"后，执行了下面的操作

```c
    coctx->co_status = NGX_HTTP_LUA_CO_RUNNING;
    ctx->co_op = NGX_HTTP_LUA_USER_THREAD_RESUME;

    ctx->cur_co_ctx->thread_spawn_yielded = 1;

    if (ngx_http_lua_post_thread(r, ctx, ctx->cur_co_ctx) != NGX_OK) {
        return luaL_error(L, "no memory");
    }

    coctx->parent_co_ctx = ctx->cur_co_ctx;
    ctx->cur_co_ctx = coctx;

    ngx_http_lua_probe_user_thread_spawn(r, L, coctx->co);

    dd("yielding with arg %s, top=%d, index-1:%s", luaL_typename(L, -1),
       (int) lua_gettop(L), luaL_typename(L, 1));
    return lua_yield(L, 1);
```
上面的代码中coctx代表生成的"light thread"，ctx->co_o表示协程yield的原因。主要关注以下几点

* 将ctx->co_op设置为NGX_HTTP_LUA_USER_THREAD_RESUME
* ngx_http_lua_post_thread将父协程放到待运行的队列中
* 将ctx->cur_co_ctx设置为coctx，ctx->cur_co_ctx代表之后需要执行的协程(这里的light thread也是协程)

在lua_resume返回LUA_YIELD后判断ctx->co_op

```c
                case NGX_HTTP_LUA_USER_THREAD_RESUME:

                    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                   "lua user thread resume");

                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;
                    nrets = lua_gettop(ctx->cur_co_ctx->co) - 1;
                    dd("nrets = %d", nrets);

#ifdef NGX_LUA_USE_ASSERT
                    /* ignore the return value (the thread) already pushed */
                    orig_coctx->co_top--;
#endif
                   break;
```
注意这里最后是break，而不是return。当时是因为ngx.thread.spawn导致父协程yield，下一个循环中会执行新生成的"light thread"。

## coroutine

lua-nginx-module中的coroutine API和原生Lua中类似，和ngx.thread不同，coroutine.create创建的协程需要手动去运行，所以resume和yield都需要在ngx_http_lua_run_thread中进行协程的切换。

```c
                case NGX_HTTP_LUA_USER_CORO_RESUME:
                    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                   "lua coroutine: resume");

                    /*
                     * the target coroutine lies at the base of the
                     * parent's stack
                     */
                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;

                    old_co = ctx->cur_co_ctx->parent_co_ctx->co;

                    nrets = lua_gettop(old_co);
                    if (nrets) {
                        dd("moving %d return values to parent", nrets);
                        lua_xmove(old_co, ctx->cur_co_ctx->co, nrets);

#ifdef NGX_LUA_USE_ASSERT
                        ctx->cur_co_ctx->parent_co_ctx->co_top -= nrets;
#endif
                    }

                    break;

                default:
                    /* ctx->co_op == NGX_HTTP_LUA_USER_CORO_YIELD */
                    /* 此处省略  */
```

# lua_resume返回0

lua_resume返回值为0表明协程执行完毕，一般情况下执行完毕就真的结束了。而这里因为有ngx.thread API的存在，可能有多个"light thread"在跑，需要等到父协程和所有的"light thread"全部结束才能真正返回，进入Nginx的下一个阶段。

如果父进程执行结束了，会判断ctx->uthreads的值，ctx->uthreads代表还在运行的"light thread"的个数。如果不为零返回NGX_AGAIN，如果为0返回NGX_OK后结束当前阶段的处理。
```c
                if (ngx_http_lua_is_entry_thread(ctx)) {

                    lua_settop(L, 0);

                    ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);

                    dd("uthreads: %d", (int) ctx->uthreads);

                    if (ctx->uthreads) {

                        ctx->cur_co_ctx = NULL;
                        return NGX_AGAIN;
                    }

                    /* all user threads terminated already */
                    goto done;
                }
```


如果是"light thread"结束了，同样判断父协程和其他的"light thread"是否全部结束了，如果是返回NGX_OK结束，否则放回NGX_AGAIN。
```c

                if (ctx->cur_co_ctx->is_uthread) {
                    /* being a user thread */
                    
                    /*   此处省略   */

                    ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);
                    ctx->uthreads--;

                    if (ctx->uthreads == 0) {
                        if (ngx_http_lua_entry_thread_alive(ctx)) {
                            ctx->cur_co_ctx = NULL;
                            return NGX_AGAIN;
                        }

                        /* all threads terminated already */
                        goto done;
                    }

                    /* some other user threads still running */
                    ctx->cur_co_ctx = NULL;
                    return NGX_AGAIN;
                }
```

# ngx_http_lua_run_thread的处理流程图

此图只为显示主要流程，省略了很多次要的细节处理。

![输入图片说明](https://static.oschina.net/uploads/img/201703/06224854_QsHa.png "在这里输入图片标题")