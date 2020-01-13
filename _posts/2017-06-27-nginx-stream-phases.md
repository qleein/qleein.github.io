---
layout: post
title:  "Nginx stream模块的执行阶段"
date:   2017-06-27 23:32:00 +0800
---

        
Nginx的stream模块提供了TCP负载均衡的功能，最初的stream模块比较简单，在nginx-1.11.4后也开始采用类似HTTP模块中分阶段处理请求的方式。


### stream模块的处理阶段

在ngx_stream.h中定义了stream模块的7个阶段。如下面所示

```c
typedef enum {
    NGX_STREAM_POST_ACCEPT_PHASE = 0,
    NGX_STREAM_PREACCESS_PHASE,
    NGX_STREAM_ACCESS_PHASE,
    NGX_STREAM_SSL_PHASE,
    NGX_STREAM_PREREAD_PHASE,
    NGX_STREAM_CONTENT_PHASE,
    NGX_STREAM_LOG_PHASE
} ngx_stream_phases;
```
与HTTP模块相同，每个阶段有相应的checker检查方法和handler回调方法，每个阶段都有零个或多个ngx_stream_phase_handler_t结构体。

```c
typedef struct ngx_stream_phase_handler_s  ngx_stream_phase_handler_t;

typedef ngx_int_t (*ngx_stream_phase_handler_pt)(ngx_stream_session_t *s,
    ngx_stream_phase_handler_t *ph);
typedef ngx_int_t (*ngx_stream_handler_pt)(ngx_stream_session_t *s);
typedef void (*ngx_stream_content_handler_pt)(ngx_stream_session_t *s);


struct ngx_stream_phase_handler_s {
    ngx_stream_phase_handler_pt    checker;
    ngx_stream_handler_pt          handler;
    ngx_uint_t                     next;
};
```

stream模块接到请求后，初始化连接后调用ngx_stream_core_run_phases依次执行各个阶段的处理函数。

```c
void
ngx_stream_core_run_phases(ngx_stream_session_t *s)
{
    ngx_int_t                     rc;
    ngx_stream_phase_handler_t   *ph;
    ngx_stream_core_main_conf_t  *cmcf;

    cmcf = ngx_stream_get_module_main_conf(s, ngx_stream_core_module);

    ph = cmcf->phase_engine.handlers;

    while (ph[s->phase_handler].checker) {

        rc = ph[s->phase_handler].checker(s, &ph[s->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}
```

### POST_ACCEPT、PREACCESS、ACCESS阶段

POST_ACCEPT、PREACCESS、ACCESS阶段的checker检查方法都是ngx_stream_core_generic_phase，这三个阶段主要进行访问控制的一些工作。因为stream模块处理TCP请求，阶段之间关系比较简单，将模块挂载在哪个阶段只会影响执行的顺序。

下面是ngx_stream_core_generic_phase函数

```c

ngx_int_t
ngx_stream_core_generic_phase(ngx_stream_session_t *s,
    ngx_stream_phase_handler_t *ph)
{
    ngx_int_t  rc;

    /*
     * generic phase checker,
     * used by all phases, except for preread and content
     */

    ngx_log_debug1(NGX_LOG_DEBUG_STREAM, s->connection->log, 0,
                   "generic phase: %ui", s->phase_handler);

    rc = ph->handler(s);

    if (rc == NGX_OK) {
        s->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    if (rc == NGX_DECLINED) {
        s->phase_handler++;
        return NGX_AGAIN;
    }

    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        return NGX_OK;
    }

    if (rc == NGX_ERROR) {
        rc = NGX_STREAM_INTERNAL_SERVER_ERROR;
    }

    ngx_stream_finalize_session(s, rc);

    return NGX_OK;
}
```

与HTTP模块类似，根据`rc = ph->handler(s)`结果进行处理。

* rc为NGX_OK表示这一阶段的工作完成，进入下一个阶段
* rc为NGX_DECLINED表示进入下一个模块处理
* rc为NGX_AGAIN表示当前请求暂时无法完成，返回NGX_OK
* rc为NGX_DONE表示当前请求告一段落，会被再次调用，返回NGX_OK
* rc为NGX_ERROR表示出现错误，结束请求
* rc为其他值时表示处理完成，此时rc为状态码

注意，TCP请求中没有HTTP请求那样的状态码，这里的状态码只是表示连接处理的信息，如源超时状态码为502。

### SSL阶段

SSL阶段的checker检查方法也是ngx_stream_core_generic_phase，这里挂载了ngx_stream_ssl_module，开启SSL时这里进行SSL握手的处理。


### PREREAD阶段

这个阶段的特点是会读取下游的请求包体。读取后调用handler回调函数处理。读取的数据会保存在c->buffer中。如CONTENT阶段中ngx_stream_proxy_module在处理时会将c->buffer中的内容发送给上游源服务器。如果你在CONTENT阶段之前读取了下游的数据又想将这些数据通过proxy发送给上游，就可以加数据放到c->buffer中。

这里官方挂载了ngx_stream_ssl_preread_module模块，当TCP连接采用SSL通信时用这个模块可以解析client hello握手包，从extensions字段中得到server_name赋值给变量$ssl_preread_server_name中。这个模块只能在监听地址没有开启SSL，但是上下游却是通过SSL进行通信的情况下使用。

PREREAD阶段checker检查方法是ngx_stream_core_preread_phase。如下所示

```c

ngx_int_t
ngx_stream_core_preread_phase(ngx_stream_session_t *s,
    ngx_stream_phase_handler_t *ph)
{
    size_t                       size;
    ssize_t                      n;
    ngx_int_t                    rc;
    ngx_connection_t            *c;
    ngx_stream_core_srv_conf_t  *cscf;

    c = s->connection;

    c->log->action = "prereading client data";

    cscf = ngx_stream_get_module_srv_conf(s, ngx_stream_core_module);

    if (c->read->timedout) {
        rc = NGX_STREAM_OK;

    } else if (c->read->timer_set) {
        rc = NGX_AGAIN;

    } else {
        rc = ph->handler(s);
    }

    while (rc == NGX_AGAIN) {

        if (c->buffer == NULL) {
            c->buffer = ngx_create_temp_buf(c->pool, cscf->preread_buffer_size);
            if (c->buffer == NULL) {
                rc = NGX_ERROR;
                break;
            }
        }

        size = c->buffer->end - c->buffer->last;

        if (size == 0) {
            ngx_log_error(NGX_LOG_ERR, c->log, 0, "preread buffer full");
            rc = NGX_STREAM_BAD_REQUEST;
            break;
        }

        if (c->read->eof) {
            rc = NGX_STREAM_OK;
            break;
        }

        if (!c->read->ready) {
            if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                rc = NGX_ERROR;
                break;
            }

            if (!c->read->timer_set) {
                ngx_add_timer(c->read, cscf->preread_timeout);
            }

            c->read->handler = ngx_stream_session_handler;

            return NGX_OK;
        }

        n = c->recv(c, c->buffer->last, size);

        if (n == NGX_ERROR) {
            rc = NGX_STREAM_OK;
            break;
        }

        if (n > 0) {
            c->buffer->last += n;
        }

        rc = ph->handler(s);
    }

    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (rc == NGX_OK) {
        s->phase_handler = ph->next;
        return NGX_AGAIN;
    }

    if (rc == NGX_DECLINED) {
        s->phase_handler++;
        return NGX_AGAIN;
    }

    if (rc == NGX_DONE) {
        return NGX_OK;
    }

    if (rc == NGX_ERROR) {
        rc = NGX_STREAM_INTERNAL_SERVER_ERROR;
    }

    ngx_stream_finalize_session(s, rc);

    return NGX_OK;
}
```

### CONTENT阶段

CONTENT阶段的检查方法是ngx_stream_core_content_phase

```c
ngx_int_t
ngx_stream_core_content_phase(ngx_stream_session_t *s,
    ngx_stream_phase_handler_t *ph)
{
    ngx_connection_t            *c;
    ngx_stream_core_srv_conf_t  *cscf;

    c = s->connection;

    c->log->action = NULL;

    cscf = ngx_stream_get_module_srv_conf(s, ngx_stream_core_module);

    if (c->type == SOCK_STREAM
        && cscf->tcp_nodelay
        && ngx_tcp_nodelay(c) != NGX_OK)
    {
        ngx_stream_finalize_session(s, NGX_STREAM_INTERNAL_SERVER_ERROR);
        return NGX_OK;
    }

    cscf->handler(s);

    return NGX_OK;
}
```

很显然这一阶段只能有一个处理方法，如进行TCP代理时需要ngx_stream_proxy_module，这里挂载的就是proxy模块的handler函数

### LOG阶段

LOG阶段虽然是一个独立的阶段，却是在连接结束时调用的。在CONTENT阶段结束时就会调用ngx_stream_finalize_session结束请求。这部分在ngx_stream_handler.c中

```c
void
ngx_stream_finalize_session(ngx_stream_session_t *s, ngx_uint_t rc)
{
    ngx_log_debug1(NGX_LOG_DEBUG_STREAM, s->connection->log, 0,
                   "finalize stream session: %i", rc);

    s->status = rc;

    ngx_stream_log_session(s);

    ngx_stream_close_connection(s->connection);
}


static void
ngx_stream_log_session(ngx_stream_session_t *s)
{
    ngx_uint_t                    i, n;
    ngx_stream_handler_pt        *log_handler;
    ngx_stream_core_main_conf_t  *cmcf;

    cmcf = ngx_stream_get_module_main_conf(s, ngx_stream_core_module);

    log_handler = cmcf->phases[NGX_STREAM_LOG_PHASE].handlers.elts;
    n = cmcf->phases[NGX_STREAM_LOG_PHASE].handlers.nelts;

    for (i = 0; i < n; i++) {
        log_handler[i](s);
    }
}
```

