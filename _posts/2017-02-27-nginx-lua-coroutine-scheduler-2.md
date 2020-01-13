---
layout: post
title:  "ngx_lua中的协程调度（二）阻塞API的处理"
date:   2017-02-27 20:34:00 +0800
---

## 协程的挂起与回复

lua-nginx-module使用Lua拓展Nginx功能的一个优点就是用同步的方式写代码，实现异步的功能。典型的一个API就是ngx.sleep。在C语言中如果调用sleep会使整个线程休眠，对于Nginx这样单进程异步处理流程来说是不可以接受的，要实现将某个请求延迟处理，需要很多额外的代码，增加了开发的难度，而在ngx\_lua中ngx.sleep只会暂停当前的协程，不影响其他的协程工作。从这方面看协程更像是用户态线程的简化。

Lua主要作为嵌入式编程语言，只提供了基础的功能，并没有golang中那样对并发原生的支持，对于sleep，socket等的处理都需要开发者来实现，这里以sleep为例。

### ngx.sleep实现

Lua提供了两个C语言接口，lua\_yield可以将一个协程挂起，lua\_resume使协程恢复运行。要使协程休眠一段时间后再运行，可以通过下面的步骤实现。

* 1.添加定时器，一段时间后执行回调函数
* 2.调用lua\_yield挂起协程
* 3.在回调函数中调用lua\_resume运行挂起的协程

在Lua中调用ngx.sleep(4)时，最终执行的是ngx\_http\_test\_ngx\_sleep，如下所示，主要功能是利用ngx\_add\_timer设置一个定时器，超时后执行ngx\_http\_test\_sleep\_handler。

```c
static int
ngx_http_test_ngx_sleep(lua_State *L)
{
    ngx_int_t       delay = luaL_checkint(L, 1);
    ngx_http_request_t *r = lua_getglobal(L, ngx_http_test_req_key);

    ngx_event_t     *sleep = ngx_pcalloc(r->pool, sizeof(ngx_event_t));
    sleep->handler = ngx_http_test_sleep_handler;
    sleep->data = r;

    ngx_add_timer(sleep, (ngx_msec_t) delay);

    return lua_yield(L, 0);
}
```

### 协程的挂起

在ngx\_http\_test\_handler中调用lua\_resume(L, 0)执行Lua脚本，如果执行完成返回值为0，这里ngx.sleep会导致lua\_yield的调用，这是lua\_resume的返回值为1,因此需要判断lua\_resume的返回值。

* 返回值为0时，脚本执行结束，返回NGX\_OK或NGX\_DECLINED
* 返回值为1时，协程被挂起，返回NGX\_DONE，Nginx会暂停当前请求的处理
* 返回其它值时，脚本执行出错。

原先的逻辑中直接调用主协程执行lua代码，这里有可能出现协程的挂起，表明当前的Lua代码没有执行完毕，这就需要对每个请求，创建单独的协程进行处理，保证多个并发请求可以同时处理。

### GC的影响

Lua中GC采用标记清除的方式，每个变量必须有其他变量引用，否则就可能被GC回收掉。Lua中的协程也是一个GC对象，多个协程同时存在时，必须为每个协程添加引用，以免被回收掉。

仿照lua-nginx-module的做法，在注册表中创建了一个table。

```c
    lua_pushlightuserdata(L, &ngx_http_test_coroutines_key);
    lua_createtable(L, 0, 0);
    lua_rawset(L, LUA_REGISTRYINDEX);
```

创建协程并通过luaL\_ref添加到table中

```c
static ngx_int_t
ngx_http_test_new_thread(lua_State *L, ngx_http_request_t *r,
    ngx_http_test_ctx_t *ctx)
{
    lua_State *vm = lua_newthread(L);

    lua_pushlightuserdata(vm, &ngx_http_test_coroutines_key);
    lua_rawget(vm, LUA_REGISTRYINDEX);

    /* 引用协程以免GC的影响 */
    ctx->ref = luaL_ref(vm, -1);
    ctx->vm = vm;
    ctx->entered_access_phase = 1;

    /* 注册ngx API */
    lua_createtable(vm, 0, 0);
    lua_pushcfunction(vm, ngx_http_test_ngx_exit);
    lua_setfield(vm, -2, "exit");

    lua_pushcfunction(vm, ngx_http_test_ngx_sleep);
    lua_setfield(vm, -2, "sleep");

    lua_setglobal(vm, "ngx");

    /* 将r保存到全局变量中，key为ngx_http_test_req_key */
    lua_pushlightuserdata(vm, r);
    lua_setglobal(vm, ngx_http_test_req_key);

    return NGX_OK;
}
```

协程不再需要时从table中删掉

```c
static ngx_int_t
ngx_http_test_del_thread(ngx_http_test_ctx_t *ctx)
{
    lua_State *L = ctx->vm;
    lua_pushlightuserdata(L, &ngx_http_test_coroutines_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    luaL_unref(L, -1, ctx->ref);
    ctx->ref = LUA_NOREF;
    return NGX_OK;
}
```


### 协程的恢复运行

定时器到期后，Nginx会调用ngx\_http\_test\_sleep\_handler，从这里开始，继续处理请求。

Nginx为了便于异步处理，将请求的处理分了多个阶段，按照阶段的次序依次处理。逻辑在ngx\_http\_core\_run\_phases中，在ngx\_http\_test\_module处理完成后需要交由下一阶段继续处理，为了保持依阶段处理的逻辑，这里不在ngx\_http\_test\_sleep\_handler中直接调用lua\_resume继续协程运行，而是调用ngx\_http\_core\_run\_phases, 这就导致了ngx\_http\_test\_handler回调函数的第二次调用。

```c
static void
ngx_http_test_sleep_handler(ngx_event_t *ev)
{
    ngx_http_request_t *r = ev->data;
    ngx_http_test_ctx_t *ctx = ngx_http_get_module_ctx(r, ngx_http_test_module);

    ctx->resume_handler = ngx_http_test_sleep_resume;
    ngx_http_core_run_phases(r);

    return;
}
```

为了进行区分，在模块的上下文中增加成员entered\_access\_phase，用来标志是否是回调函数的第一次调用。如果entered\_access\_phase，直接调用ctx->resume_handler执行即可，不需要再新建协程。

```c
    if (ctx->entered_access_phase) {
        int ret = ctx->resume_handler(r);
        if (ret == 1) return NGX_DONE;

        ngx_http_test_del_thread(ctx);
        if (ctx->status == 403)
            return NGX_HTTP_FORBIDDEN;
        return NGX_DECLINED;
    }
```

实现ngx.sleep的完整代码

```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>


typedef struct {
    lua_State   *vm;
    ngx_str_t   script;
} ngx_http_test_loc_conf_t;


typedef struct {
    lua_State           *vm;
    int                  ref;
    ngx_http_handler_pt  resume_handler;
    int                  entered_access_phase;
    int                  status;
} ngx_http_test_ctx_t;


static ngx_int_t
ngx_http_test_handler(ngx_http_request_t *r);
static ngx_int_t
ngx_http_test_init(ngx_conf_t *cf);
static void *
ngx_http_test_create_loc_conf(ngx_conf_t *cf);

static ngx_int_t
ngx_http_test_new_thread(lua_State *L, ngx_http_request_t *r,
    ngx_http_test_ctx_t *ctx);
static ngx_int_t
ngx_http_test_del_thread(ngx_http_test_ctx_t *ctx);
static int
ngx_http_test_ngx_exit(lua_State *L);
static ngx_int_t
ngx_http_test_sleep_resume(ngx_http_request_t *r);
static void
ngx_http_test_sleep_handler(ngx_event_t *ev);
static int
ngx_http_test_ngx_sleep(lua_State *L);



static ngx_command_t ngx_http_test_commands[] = {
    {
        ngx_string("access_by_lua"),
        NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_conf_set_str_slot,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_test_loc_conf_t, script),
        NULL },
    ngx_null_command
};


static ngx_http_module_t ngx_http_test_module_ctx = {
    NULL,
    ngx_http_test_init,
    NULL,
    NULL,
    NULL,
    NULL,
    ngx_http_test_create_loc_conf,
    NULL
};


ngx_module_t ngx_http_test_module = {
    NGX_MODULE_V1,
    &ngx_http_test_module_ctx,
    ngx_http_test_commands,
    NGX_HTTP_MODULE,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING
};

#define ngx_http_test_req_key    "__ngx_req"
static char ngx_http_test_coroutines_key;


static ngx_int_t
ngx_http_test_new_thread(lua_State *L, ngx_http_request_t *r,
    ngx_http_test_ctx_t *ctx)
{
    lua_State *vm = lua_newthread(L);

    lua_pushlightuserdata(vm, &ngx_http_test_coroutines_key);
    lua_rawget(vm, LUA_REGISTRYINDEX);

    /* 引用协程以免GC的影响 */
    ctx->ref = luaL_ref(vm, -1);
    ctx->vm = vm;
    ctx->entered_access_phase = 1;

    /* 注册ngx API */
    lua_createtable(vm, 0, 0);
    lua_pushcfunction(vm, ngx_http_test_ngx_exit);
    lua_setfield(vm, -2, "exit");

    lua_pushcfunction(vm, ngx_http_test_ngx_sleep);
    lua_setfield(vm, -2, "sleep");

    lua_setglobal(vm, "ngx");

    /* 将r保存到全局变量中，key为ngx_http_test_req_key */
    lua_pushlightuserdata(vm, r);
    lua_setglobal(vm, ngx_http_test_req_key);

    return NGX_OK;
}


static ngx_int_t
ngx_http_test_del_thread(ngx_http_test_ctx_t *ctx)
{
    lua_State *L = ctx->vm;
    lua_pushlightuserdata(L, &ngx_http_test_coroutines_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    luaL_unref(L, -1, ctx->ref);
    ctx->ref = LUA_NOREF;
    return NGX_OK;
}


static ngx_int_t
ngx_http_test_handler(ngx_http_request_t *r)
{
    ngx_http_test_loc_conf_t *tlcf = ngx_http_get_module_loc_conf(r, ngx_http_test_module);
    if (tlcf->script.len == 0) {
        return NGX_DECLINED;
    }

    ngx_http_test_ctx_t *ctx = ngx_http_get_module_ctx(r, ngx_http_test_module);
    if (ctx == NULL) {
        ctx = ngx_pcalloc(r->pool, sizeof(*ctx));
        ngx_http_set_ctx(r, ctx, ngx_http_test_module);
    }

    if (ctx->entered_access_phase) {
        int ret = ctx->resume_handler(r);
        if (ret == 2) return NGX_DONE;

        ngx_http_test_del_thread(ctx);
        if (ctx->status == 403)
            return NGX_HTTP_FORBIDDEN;
        return NGX_DECLINED;
    }

    ngx_http_test_new_thread(tlcf->vm, r, ctx);

    /*  加载一段Lua代码，将其编译成Lua虚拟机的字节码 */
    int ret = luaL_loadstring(ctx->vm, (const char *)tlcf->script.data);
    if (ret != 0) {
        return NGX_ERROR;
    }

    /*  调用前面加载的Lua代码 */
    ret = lua_resume(ctx->vm, 0);
    if (ret == 1) {
        return NGX_AGAIN;
    }

    ngx_http_test_del_thread(ctx);
    if (ctx->status == 403) {
        return NGX_HTTP_FORBIDDEN;
    }

    return NGX_DECLINED;
}


static int
ngx_http_test_ngx_exit(lua_State *L)
{
    int status;
    status = luaL_checkint(L, 1);

    ngx_http_request_t *r;
    lua_getglobal(L, ngx_http_test_req_key);
    r = lua_touserdata(L, -1);

    ngx_http_test_ctx_t *ctx;
    ctx = ngx_http_get_module_ctx(r, ngx_http_test_module);
    ctx->status = status;

    lua_pushboolean(L, 1);
    return 1;
}


static ngx_int_t
ngx_http_test_sleep_resume(ngx_http_request_t *r)
{
    ngx_http_test_ctx_t *ctx = ngx_http_get_module_ctx(r, ngx_http_test_module);

    int rc = lua_resume(ctx->vm, 0);
    return rc;
}


static void
ngx_http_test_sleep_handler(ngx_event_t *ev)
{
    ngx_http_request_t *r = ev->data;
    ngx_http_test_ctx_t *ctx = ngx_http_get_module_ctx(r, ngx_http_test_module);

    ctx->resume_handler = ngx_http_test_sleep_resume;
    ngx_http_core_run_phases(r);

    return;
}


static int
ngx_http_test_ngx_sleep(lua_State *L)
{
    ngx_int_t       delay = luaL_checkint(L, 1);
    ngx_http_request_t *r;

    lua_getglobal(L, ngx_http_test_req_key);
    r = lua_touserdata(L, -1);

    ngx_event_t     *sleep = ngx_pcalloc(r->pool, sizeof(ngx_event_t));
    sleep->handler = ngx_http_test_sleep_handler;
    sleep->data = r;
    sleep->log = r->connection->log;

    ngx_add_timer(sleep, (ngx_msec_t) delay * 1000);

    return lua_yield(L, 0);
}


static void *
ngx_http_test_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_test_loc_conf_t *conf = NULL;
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_test_loc_conf_t));
    if (conf == NULL) return NULL;

    ngx_str_null(&conf->script);

    /* 初始化Lua环境 */
    /* 创建一个全局的global_State结构和代表一个协程的lua_State结构，lua_State作为主协程返回 */
    lua_State   *L = luaL_newstate();
    if (!L) return NULL;

    /*  将print, math，string,table等Lua内置的函数库注册到协程中 */
    luaL_openlibs(L);

    conf->vm = L;

    lua_pushlightuserdata(L, &ngx_http_test_coroutines_key);
    lua_createtable(L, 0, 0);
    lua_rawset(L, LUA_REGISTRYINDEX);

    return conf;
}


static ngx_int_t
ngx_http_test_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt *h;
    ngx_http_core_main_conf_t *cmcf;

    /* 在ACCESS阶段挂在回调函数 */
    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
    *h = ngx_http_test_handler;

    return NGX_OK;
}
```

### nginx.conf配置

```
        location / {
            access_by_lua " ngx.sleep(4)\n ngx.exit(403)";
            root   html;
            index  index.html index.htm;
        }
```

即可实现对到来的请求，延迟4s后返回403.

### 备注

这里的代码主要目的是显示完整的流程，所以很多地方没有做错误处理。对于lua_resume调用也没有做协程栈的恢复，这些在实际编程中都是必不可少的。