---
layout: post
title:  "ngx_lua的代码缓存"
date:   2018-03-15 22:48:00 +0800
tags: Openresty
categories: Openresty
---

Lua代码的执行一般要先将代码变成成字节码，然后再Lua虚拟机中执行字节码。lua-nginx-module将编译后的结果保存了下来，这样只需要编译一次，之后便可以直接使用，省去了编译的消耗。

## Lua代码的加载

以access_by_lua为例，在Access阶段会执行指定的一段Lua代码，这是会调用ngx_http_lua_cache_loadbuffer来加载Lua代码，函数的实现如下所示

```C

ngx_int_t
ngx_http_lua_cache_loadbuffer(ngx_log_t *log, lua_State *L,
    const u_char *src, size_t src_len, const u_char *cache_key,
    const char *name)
{
    int          n;
    ngx_int_t    rc;
    const char  *err = NULL;

    n = lua_gettop(L);

    dd("XXX cache key: [%s]", cache_key);

    rc = ngx_http_lua_cache_load_code(log, L, (char *) cache_key);
    if (rc == NGX_OK) {
        /*  code chunk loaded from cache, sp++ */
        dd("Code cache hit! cache key='%s', stack top=%d, script='%.*s'",
           cache_key, lua_gettop(L), (int) src_len, src);
        return NGX_OK;
    }

    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }

    /* rc == NGX_DECLINED */

    dd("Code cache missed! cache key='%s', stack top=%d, script='%.*s'",
       cache_key, lua_gettop(L), (int) src_len, src);

    /* load closure factory of inline script to the top of lua stack, sp++ */
    rc = ngx_http_lua_clfactory_loadbuffer(L, (char *) src, src_len, name);

    if (rc != 0) {
        /*  Oops! error occurred when loading Lua script */
        if (rc == LUA_ERRMEM) {
            err = "memory allocation error";

        } else {
            if (lua_isstring(L, -1)) {
                err = lua_tostring(L, -1);

            } else {
                err = "unknown error";
            }
        }

        goto error;
    }

    /*  store closure factory and gen new closure at the top of lua stack to
     *  code cache */
    rc = ngx_http_lua_cache_store_code(L, (char *) cache_key);
    if (rc != NGX_OK) {
        err = "fail to generate new closure from the closure factory";
        goto error;
    }

    return NGX_OK;

error:

    ngx_log_error(NGX_LOG_ERR, log, 0,
                  "failed to load inlined Lua code: %s", err);
    lua_settop(L, n);
    return NGX_ERROR;
}


```

函数中首先调用ngx_http_lua_cache_load_code获取缓存，没有的话调用ngx_http_lua_clfactory_loadbuffer来加载Lua代码，完成后通过ngx_http_lua_cache_store_code将加载的结果缓存起来。

## Closure factory

在ngx_http_lua_clfactory_loadbuffer加载Lua代码时，在文件的头和尾分别加了一段代码，如下面的代码，

```Lua
local a = 1
local b = 4
ngx.log(ngx.ERR, a+b)
```

实际加载时相当与

```Lua
return function()
    local a = 1
    local b = 4
    ngx.log(ngx.ERR, a+b)
end
```

原本通过lua_load加载一段代码，生成函数A(以A标识)，执行函数A即可。
这里通过ngx_http_lua_clfactory_loadfile中的封装，得到一个Closure Factory(实际上也是一个Lua函数)。每次调用这个closure factory就会返回一个函数（即函数A）

ngx_http_lua_clfactory_loadbuffer代码如下所示

```C
ngx_int_t
ngx_http_lua_clfactory_loadbuffer(lua_State *L, const char *buff,
    size_t size, const char *name)
{
    ngx_http_lua_clfactory_buffer_ctx_t     ls;

    ls.s = buff;
    ls.size = size;
    ls.sent_begin = 0;
    ls.sent_end = 0;

    return lua_load(L, ngx_http_lua_clfactory_getS, &ls, name);
}
```

实际调用的时lua_load，参数中的ngx_http_lua_clfactory_getS表示通过这个函数读取Lua代码。

```C
#define CLFACTORY_BEGIN_CODE "return function() "
#define CLFACTORY_BEGIN_SIZE (sizeof(CLFACTORY_BEGIN_CODE) - 1)

#define CLFACTORY_END_CODE "\nend"
#define CLFACTORY_END_SIZE (sizeof(CLFACTORY_END_CODE) - 1)

static const char *
ngx_http_lua_clfactory_getS(lua_State *L, void *ud, size_t *size)
{
    ngx_http_lua_clfactory_buffer_ctx_t      *ls = ud;

    if (ls->sent_begin == 0) {
        ls->sent_begin = 1;
        *size = CLFACTORY_BEGIN_SIZE;

        return CLFACTORY_BEGIN_CODE;
    }

    if (ls->size == 0) {
        if (ls->sent_end == 0) {
            ls->sent_end = 1;
            *size = CLFACTORY_END_SIZE;
            return CLFACTORY_END_CODE;
        }

        return NULL;
    }

    *size = ls->size;
    ls->size = 0;

    return ls->s;
}
```

在ngx_http_lua_clfactory_getS中在Lua代码的首尾添加的响应的代码。

```C
static ngx_int_t
ngx_http_lua_cache_store_code(lua_State *L, const char *key)
{
    int rc;

    /*  get code cache table */
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    dd("Code cache table to store: %p", lua_topointer(L, -1));

    if (!lua_istable(L, -1)) {
        dd("Error: code cache table to load did not exist!!");
        return NGX_ERROR;
    }

    lua_pushvalue(L, -2); /* closure cache closure */
    lua_setfield(L, -2, key); /* closure cache */

    /*  remove cache table, leave closure factory at top of stack */
    lua_pop(L, 1); /* closure */

    /*  call closure factory to generate new closure */
    rc = lua_pcall(L, 0, 1, 0);
    if (rc != 0) {
        dd("Error: failed to call closure factory!!");
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

先将加载Lua代码生成的函数保存在table中，此table通过全局的注册表索引，以免被GC回收掉。 然后调用lua_pcall执行函数工厂，生成一个新的函数，之后Access阶段执行这个新的函数，完成需要的工作。

类似在ngx_http_lua_cache_load_code中拿到保存的closure factory后，也需要调用lua_pcall返回新的函数。

## Closure factory的作用

这个closure factory存在的意思是什么呢？从表面上看的话，这种做法除了每次使用时增加了一次lua_pcall的调用外，没有什么作用。确实，即使不这样封装也可以照常运行，实际上最初的lua-nginx-module也没有使用closure factory。后面改用closure factory是为了让处理不同请求的Lua协程完全分离，互不影响。

这个涉及到Lua中的环境的概念，每个Lua函数都关联了一个称为环境的table，在Lua代码中的全局变量实际是在关联的环境中，如果没有closure factory，所有的协程在Access阶段执行的是同一个函数，使用的就会使同一个环境。一个协程声明了或者修改全局变量，其他的协程都能看到，都会收到影响。而使用了closure factory后，每个协程操作的是独立的函数，而且会为函数设置单独的环境，这样协程之间不会由任何的相互影响。

关于环境的介绍，可以参考另一篇文章：[Lua的全局变量与环境](https://qlee.in/luazhong-de-quan-ju-bian-liang-yu-huan-jing/)

## lua_code_cache指令

lua-nginx-module提供了lua_code_cache指令可以开启/关闭代码的缓存，如果设置为OFF，每次执行都会重新加载一遍，比较方便开发时调试。

实际上这个指令为OFF是不只是重新加载响应的代码，而是每个请求会创建一个全新的Lua运行环境。

正常的流程中，Nginx启动时会创建一个Lua运行环境，之后对于每个的请求，会基于这个Lua运行环境创建新的协程去处理，如果设置了`lua_code_cache off;`，会为每个请求创建完全独立的Lua运行环境。执行Lua代码前都会通过ngx_http_lua_create_ctx创建lua-nginx-module模块的上下文结构体，创建过程如下所示

```C
static ngx_inline ngx_http_lua_ctx_t *
ngx_http_lua_create_ctx(ngx_http_request_t *r)
{
    lua_State                   *L;
    ngx_http_lua_ctx_t          *ctx;
    ngx_pool_cleanup_t          *cln;
    ngx_http_lua_loc_conf_t     *llcf;
    ngx_http_lua_main_conf_t    *lmcf;

    ctx = ngx_palloc(r->pool, sizeof(ngx_http_lua_ctx_t));
    if (ctx == NULL) {
        return NULL;
    }

    ngx_http_lua_init_ctx(r, ctx);
    ngx_http_set_ctx(r, ctx, ngx_http_lua_module);

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    if (!llcf->enable_code_cache && r->connection->fd != (ngx_socket_t) -1) {
        lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);

        dd("lmcf: %p", lmcf);

        L = ngx_http_lua_init_vm(lmcf->lua, lmcf->cycle, r->pool, lmcf,
                                 r->connection->log, &cln);
        if (L == NULL) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "failed to initialize Lua VM");
            return NULL;
        }

        if (lmcf->init_handler) {
            if (lmcf->init_handler(r->connection->log, lmcf, L) != NGX_OK) {
                /* an error happened */
                return NULL;
            }
        }

        ctx->vm_state = cln->data;

    } else {
        ctx->vm_state = NULL;
    }

    return ctx;
}
```

可以看到如果`llcf->enable_code_cache`为0（即设置了`lua_code_cache off`），会通过ngx_http_lua_init_vm创建新的Lua运行环境，保存在ctx->vm_state中。

所以关闭代码缓存后，增加的性能消耗远不止编译一段Lua代码那么简单，这个只能用于开发调试。