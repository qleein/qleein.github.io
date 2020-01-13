---
layout: post
title:  "ngx_lua中的协程调度(六)之ngx_http_lua_run_posted_thread"
date:   2017-03-07 23:20:00 +0800
---

## ngx_http_lua_run_posted_thread

这个函数主要是为了ngx.thread.spawn的处理，ngx.thread.spawn生成新的"light thread"，这个"light thread"运行优先级比它的父协程高，会优先运行，父协程被迫暂停。"light thread"运行结束或者yield后，再由ngx_http_lua_run_posted_threads去运行父协程。

ngx.thread.spawn中创建"light thread"后， 调用ngx_http_lua_post_thread。

```c
    if (ngx_http_lua_post_thread(r, ctx, ctx->cur_co_ctx) != NGX_OK) {
        return luaL_error(L, "no memory");
    }
```

ngx_http_lua_post_thread函数将父协程放在了ctx->posted_threads指向的链表中。

```c
ngx_int_t
ngx_http_lua_post_thread(ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx,
    ngx_http_lua_co_ctx_t *coctx)
{
    ngx_http_lua_posted_thread_t  **p;
    ngx_http_lua_posted_thread_t   *pt;

    pt = ngx_palloc(r->pool, sizeof(ngx_http_lua_posted_thread_t));
    if (pt == NULL) {
        return NGX_ERROR;
    }

    pt->co_ctx = coctx;
    pt->next = NULL;

    for (p = &ctx->posted_threads; *p; p = &(*p)->next) { /* void */ }

    *p = pt;

    return NGX_OK;
}
```

ngx_http_lua_run_posted_threads从ctx->posted_threads指向的链表中依次取出每个元素，调用ngx_http_lua_run_thread运行。

```c
/* this is for callers other than the content handler */
ngx_int_t
ngx_http_lua_run_posted_threads(ngx_connection_t *c, lua_State *L,
    ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx)
{
    ngx_int_t                        rc;
    ngx_http_lua_posted_thread_t    *pt;

    for ( ;; ) {
        if (c->destroyed) {
            return NGX_DONE;
        }

        pt = ctx->posted_threads;
        if (pt == NULL) {
            return NGX_DONE;
        }

        ctx->posted_threads = pt->next;

        ngx_http_lua_probe_run_posted_thread(r, pt->co_ctx->co,
                                             (int) pt->co_ctx->co_status);

        if (pt->co_ctx->co_status != NGX_HTTP_LUA_CO_RUNNING) {
            continue;
        }

        ctx->cur_co_ctx = pt->co_ctx;

        rc = ngx_http_lua_run_thread(L, r, ctx, 0);

        if (rc == NGX_AGAIN) {
            continue;
        }

        if (rc == NGX_DONE) {
            ngx_http_lua_finalize_request(r, NGX_DONE);
            continue;
        }

        /* rc == NGX_ERROR || rc >= NGX_OK */

        if (ctx->entered_content_phase) {
            ngx_http_lua_finalize_request(r, rc);
        }

        return rc;
    }

    /* impossible to reach here */
}

```

## ngx_http_lua_run_thread使用方式

以lua-nginx-module的Access阶段的处理为例，实际的执行工作由ngx_http_lua_access_by_chunk函数中实现。
如下面的代码，调用ngx_http_lua_run_thread后根据返回值继续处理。

```c
static ngx_int_t
ngx_http_lua_access_by_chunk(lua_State *L, ngx_http_request_t *r)
{
    /* 此处省去了创建协程的部分，只关注协程的运行  */


    rc = ngx_http_lua_run_thread(L, r, ctx, 0);

    dd("returned %d", (int) rc);

    if (rc == NGX_ERROR || rc > NGX_OK) {
        return rc;
    }

    c = r->connection;

    if (rc == NGX_AGAIN) {
        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }

    } else if (rc == NGX_DONE) {
        ngx_http_lua_finalize_request(r, NGX_DONE);

        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }
    }

#if 1
    if (rc == NGX_OK) {
        if (r->header_sent) {
            dd("header already sent");

            /* response header was already generated in access_by_lua*,
             * so it is no longer safe to proceed to later phases
             * which may generate responses again */

            if (!ctx->eof) {
                dd("eof not yet sent");

                rc = ngx_http_lua_send_chain_link(r, ctx, NULL
                                                  /* indicate last_buf */);
                if (rc == NGX_ERROR || rc > NGX_OK) {
                    return rc;
                }
            }

            return NGX_HTTP_OK;
        }

        return NGX_OK;
    }
#endif

    return NGX_DECLINED;
}
```

## ngx_http_lua_run_thread的返回值

函数ngx_http_lua_run_thread的返回值可分为下面几种 

*  NGX_OK
*  NGX_AGAIN
*  NGX_DONE
*  NGX_ERROR:  执行出错
*  大于200:  响应的HTTP状态码

按照Nginx的处理规则，返回NGX_ERROR或大于200的HTTP状态码时，将会无条件结束当前请求的处理。
返回NGX_OK表明当前阶段处理完成，此时只需要调用ngx_http_lua_send_chain_link发送响应即可。重点关注的是NGX_AGAIN和NGX_DONE这两个。返回这两个值时都要调用ngx_http_lua_run_posted_thread来处理。


### NGX_AGAIN

ngx_http_lua_run_thread什么时候会返回NGX_AGAIN?
* 1. ngx.sleep或ngx.socket等导致协程的yield
* 2. 调用ngx.thread导致当前请求对应的一个父协程和一个或多个"light thread"没有全部退出。
由于情况2的存在，需要调用ngx_http_lua_run_posted_thread进行处理。

### NGX_DONE

在Nginx中，NGX_DONE表示对当前请求的处理已经告一段落了，但是请求还没有处理完成，之后的工作会有其他的模块进行。主要出现在三个地方
* 调用ngx_http_read_client_request_body读取请求包体时，由于从socket读取数据是异步的，会返回NGX_DONE。读取数据后对请求的处理由设置的回调函数执行
* 调用ngx_http_internal_redirect执行了内部跳转
* 创建子请求后，等待子请求完成后继续处理

在ngx_http_request_t中有一个作为引用计数的成员count。每次调用ngx_http_finalize_requet(r, NGX_DONE)时会将r的引用计数减一，减为0时才会真正结束当前请求。与此对应的模块返回NGX_DONE时都会有r->count++的操作。

在函数ngx_http_lua_access_by_chunk中当ngx_http_lua_run_thread返回NGX_DONE时(相比于返回值为NGX_AGAIN的情况)增加了一次ngx_http_finalize_request(r, NGX_DONE)的操作，就是为了将r的引用计数减一。

如果这里调用ngx_http_finalize_request(r, NGX_DONE)导致r的引用计数为0,将请求结束了，此时c->destory为true，再调用ngx_http_lua_run_posted_thread会直接返回。