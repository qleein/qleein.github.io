---
layout: post
title:  "ngx_lua中的协程调度(三)"
date:   2017-03-01 22:42:00 +0800
---

通过lua-nginx-module的ngx.socket可以方便的建立与其他服务器的连接和数据传输，这些也是lua-resty-redis，lua-resty-mysql等众多请求第三方服务的模块的基础。这里只介绍ngx.socket.tcp，udp的实现类似。

### 通过lua\_resume返回值

在Lua中通过下面的方式使用ngx.socket API

```lua
local sock = ngx.socket.tcp()
local ok, err = sock:connect("google.com", 80)
if not ok then
    return
end

local ok, err = sock:send("GET / HTTP/1.0\r\nHost:google.com\r\n\r\n")
if not ok then
    return
end

local data, err = sock:receive()
if not data then
    return
end
```

sock:connect，sock:send，sock:receive这三个都可能不能立即完成，都是通过与ngx.sleep类似的方式，先调用lua\_yield将协程挂起，等到事件就绪（或连接建立，或收到数据包等）再调用lua_resume继续运行协程。这样底层是异步结构，在Lua协程中看一个Lua脚本确实连续的在执行。

不同的是ngx.sleep没有任何返回值，socket操作确是需要的，建立连接需要知道成功了还是失败了，发送接受数据需要知道是否成功，失败时也需要知道原因。这些方法的返回值是通过lua\_resume实现的。

以connect为例下面这行代码可以分成两部分，

* 1.sock:connect函数调用
* 2.将函数调用的结果赋值给ok,err两个局部变量

```lua
local ok, err = sock:connect("GET")
```

第一步调用sock:connect函数的时候发生了协程的挂起(lua\_yield)，协程继续运行时是从第二步开始的。那么给ok,err这两个局部变量赋于什么值呢？值从哪里来呢？很明显只能通过lua\_resume来操作。

下面是lua_resume的声明，第二个参数是返回值的个数。

```c
int lua_resume (lua_State *L, int narg);
```

在执行lua\_resume前将返回值压到栈上，调用lua\_resume的第二个参数控制返回值的个数。

connect操作成功时，通过下面的操作

```c
lua_pushboolean(L, 1);
lua_resume(L, 1);
```

将ok赋值为true，err赋值为nil，表明操作成功。

失败时，用下面的方法，将ok赋值为nil，err赋值为错误信息，这里使用"connect refused"只是为了示例。

```c
lua_pushnil(L);
lua_pushliteral(L, "connect refuled");
lua_resume(L, 2);
```


### 注册API

lua-nginx-module通过ngx\_http\_lua\_inject\_socket\_tcp\_api将tcp相关的API注册到ngx中。这里其实只是注册了ngx.socket.tcp(ngx.socket.stream)和ngx.socket.connect。ngx.socket.connect只是一个语法糖，即使去掉也没有什么影响。

```c
void
ngx_http_lua_inject_socket_tcp_api(ngx_log_t *log, lua_State *L)
{
    ngx_int_t         rc;

    lua_createtable(L, 0, 4 /* nrec */);    /* ngx.socket */

    lua_pushcfunction(L, ngx_http_lua_socket_tcp);
    lua_pushvalue(L, -1);
    lua_setfield(L, -3, "tcp");
    lua_setfield(L, -2, "stream");

    {
        const char  buf[] = "local sock = ngx.socket.tcp()"
                            " local ok, err = sock:connect(...)"
                            " if ok then return sock else return nil, err end";

        rc = luaL_loadbuffer(L, buf, sizeof(buf) - 1, "=ngx.socket.connect");
    }

    if (rc != NGX_OK) {
        ngx_log_error(NGX_LOG_CRIT, log, 0,
                      "failed to load Lua code for ngx.socket.connect(): %i",
                      rc);

    } else {
        lua_setfield(L, -2, "connect");
    }

    lua_setfield(L, -2, "socket");
```

为什么这里没有connect, send, receive等方法呢？这个涉及到ngx.socket.tcp方法的实现。

### ngx.socket.tcp

调用ngx.socket.tcp时，实际执行的是ngx\_http\_lua\_socket\_tcp这个函数。这里只是创建了一个Lua中的table并将其返回，并没有真正创建一个socket。

注意这里的lua\_setmetatable(L, -2)。 在创建table的同时给这个table设置了一个元表，将socket需要执行的connect，send，receive等方法放在了元表中。

```c
static int
ngx_http_lua_socket_tcp(lua_State *L)
{
    ngx_http_request_t      *r;
    ngx_http_lua_ctx_t      *ctx;

    if (lua_gettop(L) != 0) {
        return luaL_error(L, "expecting zero arguments, but got %d",
                          lua_gettop(L));
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx found");
    }

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT
                               | NGX_HTTP_LUA_CONTEXT_SSL_SESS_FETCH);

    lua_createtable(L, 3 /* narr */, 1 /* nrec */);
    lua_pushlightuserdata(L, &ngx_http_lua_tcp_socket_metatable_key);
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);

    dd("top: %d", lua_gettop(L));

    return 1;
}
```

类似这样的使用, sock只是一个空的table，调用connect方法时利用了Lua中元表的特性来实现。

```lua
local sock = ngx.socket.tcp()
local ok, err = sock:connect("1.1.1.1", 80)
```

### 非阻塞socket操作的处理

其余connect， send， receive等的实现与之前介绍的sleep等类似。这里用的是非阻塞的socket，如果操作没有立即结束，会将当前的协程挂起，等到操作完成后再恢复协程运行。

socket的connect， send，receive等操作均可能导致协程的挂起。以发送数据为例，函数ngx\_http\_lua\_socket\_tcp\_send中，如果当前socket缓冲区已满，此时rc == NGX\_AGAIN, 先设置r->write\_event\_handler回调函数，最后通过lua\_yield挂起协程。

```c
    /* rc == NGX_AGAIN */

    coctx = ctx->cur_co_ctx;

    ngx_http_lua_cleanup_pending_operation(coctx);
    coctx->cleanup = ngx_http_lua_coctx_cleanup;
    coctx->data = u;

    if (u->raw_downstream) {
        ctx->writing_raw_req_socket = 1;
    }

    if (ctx->entered_content_phase) {
        r->write_event_handler = ngx_http_lua_content_wev_handler;

    } else {
        r->write_event_handler = ngx_http_core_run_phases;
    }

    u->write_co_ctx = coctx;
    u->write_waiting = 1;
    u->write_prepare_retvals = ngx_http_lua_socket_tcp_send_retval_handler;

    dd("setting data to %p", u);

    return lua_yield(L, 0);
```

等到socket的缓冲区空闲，此时socket可写，会调用r->write\_event\_handler继续运行，最终通过ngx\_http\_lua\_socket\_tcp\_resume\_helper继续协程的运行。

```c
static ngx_int_t
ngx_http_lua_socket_tcp_resume_helper(ngx_http_request_t *r, int socket_op)
{
    int                          nret;
    lua_State                   *vm;
    ngx_int_t                    rc;
    ngx_connection_t            *c;
    ngx_http_lua_ctx_t          *ctx;
    ngx_http_lua_co_ctx_t       *coctx;

    ngx_http_lua_socket_tcp_retval_handler  prepare_retvals;

    ngx_http_lua_socket_tcp_upstream_t      *u;

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return NGX_ERROR;
    }

    ctx->resume_handler = ngx_http_lua_wev_handler;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua tcp operation done, resuming lua thread");

    coctx = ctx->cur_co_ctx;

    dd("coctx: %p", coctx);

    u = coctx->data;

    switch (socket_op) {
    case SOCKET_OP_CONNECT:
    case SOCKET_OP_WRITE:
        prepare_retvals = u->write_prepare_retvals;
        break;

    case SOCKET_OP_READ:
        prepare_retvals = u->read_prepare_retvals;
        break;

    default:
        /* impossible to reach here */
        return NGX_ERROR;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua tcp socket calling prepare retvals handler %p, "
                   "u:%p", prepare_retvals, u);

    nret = prepare_retvals(r, u, ctx->cur_co_ctx->co);
    if (nret == NGX_AGAIN) {
        return NGX_DONE;
    }

    c = r->connection;
    vm = ngx_http_lua_get_lua_vm(r, ctx);

    rc = ngx_http_lua_run_thread(vm, r, ctx, nret);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua run thread returned %d", rc);

    if (rc == NGX_AGAIN) {
        return ngx_http_lua_run_posted_threads(c, vm, r, ctx);
    }

    if (rc == NGX_DONE) {
        ngx_http_lua_finalize_request(r, NGX_DONE);
        return ngx_http_lua_run_posted_threads(c, vm, r, ctx);
    }

    if (ctx->entered_content_phase) {
        ngx_http_lua_finalize_request(r, rc);
        return NGX_DONE;
    }

    return rc;
}
```

还是同样的套路，ngx\_http\_lua\_get\_lua\_vm, 然后通过ngx\_http\_lua\_run\_thread运行协程，根据返回值做相应的处理。

