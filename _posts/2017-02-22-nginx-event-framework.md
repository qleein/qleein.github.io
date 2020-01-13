---
layout: post
title:  "Nginx事件驱动框架"
date:   2017-02-22 22:21:00 +0800
---

最初的Web服务器如Apache，采用的是fork and run的模式，对每个到来的连接，fork一个进程去处理，处理完成后进程退出。优点是编程实现简单，缺点是并发处理能力不足。为应对高并发的处理，以Nginx为代表的异步处理方式应运而生。

Nginx以事件驱动工作，事件的来源有两个，网络IO和定时器。

对于高并发的网络IO处理，不同的操作系统提供了不同的解决方案，如linux的epoll， freebsd的kqueue。这里以epoll为例。将需要监听的socket加入到epoll中后，便可以通过epoll\_wait获取已发生的事件，避免对众多的socket进行轮寻。

Nginx的定时器采用红黑数实现，每个定时器事件以超时时间为key插入到红黑树中，每次取红黑树中key最小的结点与当前的系统时间比较即可知道是否超时。

Nginx工作进程处理任务的核心在ngx\_process\_events\_and\_timers函数中。

```c
    for ( ;; ) {

        ...................

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

        ngx_process_events_and_timers(cycle);

        ...................
    }
```

ngx\_process\_events\_and\_timers主要逻辑如下所示。

```c
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{


    (void) ngx_process_events(cycle, timer, flags);

    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    
    ngx_event_expire_timers();

    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

函数中先调用ngx\_process\_events处理epoll\_wait得到的网络IO事件，再调用ngx\_event\_expire\_timers处理所有的超时事件。

```
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;


    events = epoll_wait(ep, event_list, (int) nevents, timer);

    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;
        rev = c->read;

        revents = event_list[i].events;


        if ((revents & EPOLLIN) && rev->active) {
             rev->handler(rev);
        }

        wev = c->write;

        if ((revents & EPOLLOUT) && wev->active) {

             wev->handler(wev);
        }
    }

    return NGX_OK;
}
```

利用epoll模式时ngx\_process\_events即为ngx\_epoll\_process\_events，上面的代码中省略了无关的部分，当revents & EPOLLIN为true时socket有数据可以读取，revents & EPOLLOUT为true时socket可写，然后调用相应的handler回调函数进行处理。

ngx\_event\_expire\_timers处理定时器事件逻辑更简单一些，主要就是调用ngx\_rbtree\_min获取超时时间最近的事件，如果已超时即调用ev->handler进行处理。

```c
void
ngx_event_expire_timers(void)
{
    for ( ;; ) {
        root = ngx_event_timer_rbtree.root;

        if (root == sentinel) {
            return;
        }

        node = ngx_rbtree_min(root, sentinel);

        /* node->key > ngx_current_time */
        if ((ngx_msec_int_t) (node->key - ngx_current_msec) > 0) {
            return;
        }

        ev->timer_set = 0;

        ev->timedout = 1;

        ev->handler(ev);
    }
}
```



对于网络IO主要操作就是读和写，现实网络环境非常复杂，连接会因为各种原因中断，无法传输数据。同时服务器端的每个网络连接都要消耗服务器资源，为了避免无效的连接一直占用系用的资源，需要对读写操作设置超时机制，为此Nginx做了专门的处理。

对每个到来的链接，Nginx分配一个ngx\_connection\_t结构体存储相应的信息，每个ngx\_connection\_t结构体都有两个成员read和write，对应两个ngx\_event\_t事件。当需要从socket读取数据时，将socket加入到epoll监听事件中，同时将对应的read事件加入到定时器中，如果定时器超时后仍然没有数据可读，便认为读取数据超时。

事件超时时，Nginx会将ev->timedout置为1，再调用ev->handler。相应的Nginx中对每个事件的handler回调函数大多会有如下的逻辑

```c
    if (ev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        ngx_http_close_connection(c);
        return;
    }
```

handler回调函数的开头判断事件是否已超时，如果ev->timedout不为0便认为已超时。可能会关闭连接，也可能会执行其他操作。