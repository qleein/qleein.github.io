---
layout: post
title:  "Nginx的文件分片-slice模块"
date:   2018-03-14 23:23:00 +0800
tags: Nginx 文件分片
categories: Nginx
---

Nginx的slice模块可以将一个请求分解成多个子请求，每个子请求返回响应内容的一个片段，让大文件的缓存更有效率。

## HTTP Range请求

HTTP客户端下载文件时，如果发生了网络中断，必须重新向服务器发起HTTP请求，这时客户端已经有了文件的一部分，只需要请求剩余的内容，而不需要传输整个文件，Range请求就可以用来处理这种问题。

如果HTTP请求的头部有Range字段，如下面所示

```
Range: bytes=1024-2047
```

表示客户端请求文件的第1025到第2048个字节，这时服务器只会响应文件的这部分内容，响应的状态码为206，表示返回的是响应的一部分。如果服务器不支持Range请求，仍然会返回整个文件，这时状态码仍是200。

## Nginx启用slice模块

ngx_http_slice_filter_module模块默认没有编译到Nginx程序中，需要编译时添加--with-http_slice_module选项。

编译完成后， 需要在Nginx配置文件中开启，配置如下所示

```
location / {
    slice             1m;
    proxy_cache       cache;
    proxy_cache_key   $uri$is_args$args$slice_range;
    proxy_set_header  Range $slice_range;
    proxy_cache_valid 200 206 1h;
    proxy_pass        http://localhost:8000;
}
```

slice指令设置分片的大小为1m。
这里使用了proxy_set_header指令，在取源时的HTTP请求中添加了Range头部，向源服务器请求文件的一部分，而不是全部内容。在proxy_cache_key中添加slice_range变量这样可以分片缓存。

## slice_range变量

slice_range这个变量作用非常特殊，这个变量的值是当前需要向源服务器请求的分片，如果分片的大小为1m，那么最开始变量的值为`bytes=0-1048575`，通过配置文件中的`proxy_set_header Range $slice_range;`可以知道取源时请求的Range头部为`Range:bytes=0-1048575`，源服务器如果支持Range请求，便会返回响应的前1m字节，得到这个响应后slice_range变量的值变为`bytes=1048576-2097171`
，再次取源时便会取后1m字节，依次直到取得全部响应内容。

## Nginx分片的实现

Nginx的slice模块是通过挂载filter模块来起作用的，处理流程如下所示

![输入图片说明](https://static.oschina.net/uploads/img/201803/14224948_aIjX.png "在这里输入图片标题")

1. 每次取源时都会携带Range头部，
2. 第一次取源请求前1m内容，如果响应在1m以内，或者源服务器不支持Range请求，返回状态码为200，这时会直接跳过slice模块。
3. 在body_filter中向客户端发送得到的当前的分片，然后检查是否到达文件末尾，如果没有则生成一个子请求，子请求会向源服务器请求下一个分片，依次循环。

slice模块的body_filter处理在ngx_http_slice_body_filter函数中

```C
static ngx_int_t
ngx_http_slice_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t                   rc;
    ngx_chain_t                *cl;
    ngx_http_slice_ctx_t       *ctx;
    ngx_http_slice_loc_conf_t  *slcf;

    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);

        /* 如果当前请求是子请求，直接调用下一个body_filter回调然后返回，*/
    if (ctx == NULL || r != r->main) {
        return ngx_http_next_body_filter(r, in);
    }

    for (cl = in; cl; cl = cl->next) {
        if (cl->buf->last_buf) {
            cl->buf->last_buf = 0;
            cl->buf->last_in_chain = 1;
            cl->buf->sync = 1;
            ctx->last = 1;
        }
    }

    /* 向客户端发送当前的分片 */
    rc = ngx_http_next_body_filter(r, in);

    if (rc == NGX_ERROR || !ctx->last) {
        return rc;
    }

    /* 
        如果当前的子请求还没有接受完全部的响应会直接返回，这里子请求向源请求，得到响应后由这里的主请求发送给客户端，子请求只负责取源。当前子请求接收完全部的响应(这时
        ctx->sr->done为1)后，主请求才会生成下一个子请求去取下一个分片
    */
    if (ctx->sr && !ctx->sr->done) {
        return rc;
    }

    if (!ctx->active) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "missing slice response");
        return NGX_ERROR;
    }

    /*
        如果已经到达文件末尾，则返回
    */
    if (ctx->start >= ctx->end) {
        ngx_http_set_ctx(r, NULL, ngx_http_slice_filter_module);
        ngx_http_send_special(r, NGX_HTTP_LAST);
        return rc;
    }

    if (r->buffered) {
        return rc;
    }

    /*
        生成子请求，注意这里的NGX_HTTP_SUBREQUEST_CLONE，默认生成子请求从server_rewrite阶段执行并跳过access阶段，这里NGX_HTTP_SUBREQUEST_CLONE使生成的子请求从主请求的当前阶段（即content阶段）开始执行
    */
    if (ngx_http_subrequest(r, &r->uri, &r->args, &ctx->sr, NULL,
                            NGX_HTTP_SUBREQUEST_CLONE)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    ngx_http_set_ctx(ctx->sr, ctx, ngx_http_slice_filter_module);

    slcf = ngx_http_get_module_loc_conf(r, ngx_http_slice_filter_module);

    /*
    ctx->range指向的使slice_range变量的值
    */
    ctx->range.len = ngx_sprintf(ctx->range.data, "bytes=%O-%O", ctx->start,
                                 ctx->start + (off_t) slcf->size - 1)
                     - ctx->range.data;

    ctx->active = 0;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http slice subrequest: \"%V\"", &ctx->range);

    return rc;
}

```

## Range的范围

请求中的Range范围可能会超过文件的大小，如第一次取源时，Nginx并不知道实际文件的大小，所以Nginx请求时总是按照分片的大小设置Range范围，如slice设置为1m，那么第一次取`bytes=0-1048575`，如果文件不足1m，响应状态吗为200，表示不需要分片。如果超过1m，第二次取源时Range字段为`bytes=1048576-2097171`，即使这时可以知道文件实际大小。

线上使用时就遇到过一次源服务器对Range请求支持不完善的问题，文件大小为1.5m，第一次取源状态码为206，返回1m内容，第二次取源使Range字段为`bytes=1048576-2097171`，但是文件不足2m，源服务器发现这个范围超过了文件大小，所以返回了整个文件，状态码为200，这时Nginx就不能理解了，直接报错中断了响应。

开始以为是Nginx的问题，然后查看了下RFC文档，发现有解释这种情况

>   A client can limit the number of bytes requested without knowing the
   size of the selected representation.  If the last-byte-pos value is
   absent, or if the value is greater than or equal to the current
   length of the representation data, the byte range is interpreted as
   the remainder of the representation (i.e., the server replaces the
   value of last-byte-pos with a value that is one less than the current
   length of the selected representation).

大致意思是说，如果请求的分片的后一个偏移超过了文件的实际大小，服务器应该返回剩余的部分内容。这个问题应该是源服务器的实现并没有按照RFC文档的要求。

## 参考链接

* [Module ngx_http_slice_module](http://nginx.org/en/docs/http/ngx_http_slice_module.html)
* [Hypertext Transfer Protocol (HTTP/1.1): Range Requests](https://tools.ietf.org/html/rfc7233)