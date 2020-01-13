---
layout: post
title:  "ngx_lua中的协程调度（一）在Nginx中嵌入Lua环境"
date:   2017-02-26 18:44:00 +0800
---

## 命令行中执行lua

在命令行中调用lua执行一条输出语句， 如下所示。

```
$ luajit -e "print('Hello, Lua')"
Hello, Lua
```


## C程序中内嵌Lua运行环境

在C语言中创建Lua运行环境，执行同样的Lua语句也相当简单。

```c
#include <stdio.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

const char script[] = "print('Hello, Lua!')";

int main(void) {
    /* 创建一个全局的global_State结构和代表一个协程的lua_State结构，lua_State作为主协程返回 */
    lua_State   *L = luaL_newstate();
    if (!L) return -1;

    /*  将print, math，string,table等Lua内置的函数库注册到协程中 */
    luaL_openlibs(L);

    /*  加载一段Lua代码，将其编译成Lua虚拟机的字节码 */
    int ret = luaL_loadstring(L, script);
    if (ret != 0) {
        return -1;
    }

    /*  在Lua虚拟机中执行前面加载的Lua代码 */
    //ret = lua_pcall(L, 0, LUA_MULTRET, 0);
    ret = lua_resume(L, 0);
    if (ret != 0) {
        return -1;
    }

    lua_close(L);

    return 0;
}
```


## Nginx C模块嵌入Lua脚本

### 主要工作

编写一个简单的Nginx模块，在ACCESS阶段执行配置中指定的Lua脚本， 主要工作有

* 解析配置时创建Lua运行环境
* 在ACCESS阶段挂在回调函数，执行配置中设置的Lua脚本
* 在Nginx配置中的location中增加 access\_by\_lua "print('Hello, Lua!')"
* 模块名为ngx\_http\_test\_module，源文件为ngx\_http\_test\_module.c， 文件内容如下。

ngx\_http\_test\_module.c文件内容

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


static ngx_int_t
ngx_http_test_handler(ngx_http_request_t *r);
static ngx_int_t
ngx_http_test_init(ngx_conf_t *cf);
static void *
ngx_http_test_create_loc_conf(ngx_conf_t *cf);


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


static ngx_int_t
ngx_http_test_handler(ngx_http_request_t *r)
{
    ngx_http_test_loc_conf_t *tlcf = ngx_http_get_module_loc_conf(r, ngx_http_test_module);
    if (tlcf->script.len == 0) {
        return NGX_DECLINED;
    }

    /*  加载一段Lua代码，将其编译成Lua虚拟机的字节码 */
    int ret = luaL_loadstring(tlcf->vm, (const char *)tlcf->script.data);
    if (ret != 0) {
        return -1;
    }

    /*  调用前面加载的Lua代码 */
    ret = lua_resume(tlcf->vm, 0);
    if (ret != 0) {
        return -1;
    }

    return NGX_DECLINED;
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

### 模块config文件

```shell
ngx_addon_name=ngx_http_test_module
HTTP_MODULES="$HTTP_MODULES ngx_http_test_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_test_module.c"
```

### 编译脚本内容

主要是连接Lua库和头文件，这里用的是LuaJIT-2.1.0, 如果用的是其他版本或者lua5.1需要根据需要更改。

```
export LUAJIT_INC=/usr/local/include/luajit-2.1
export LUAJIT_LIB=/usr/local/lib


./configure --with-debug \
    --with-cc-opt='-O0 -I /usr/local/include/luajit-2.1'  \
    --with-ld-opt='-Wl,-rpath,/usr/local/lib -lluajit-5.1' \
    --add-module=$HOME/ngx_http_test_module 
```

### nginx.conf

在location中增加access_by_lua指定执行的Lua代码。

```
daemon off;

events {
    worker_connections  1024;
}

http {
    server {
        listen       80;
        server_name  localhost;
        location / {
	        access_by_lua "print('Hello, Lua!')";
            root   html;
            index  index.html index.htm;
        }

    }
}
```

### 运行

编译成功后启动Nginx，用curl或浏览器访问， Nginx会在终端输出

```
$ ./sbin/nginx
Hello, Lua!
```

如果以daemon方式运行Nginx，可能无法输出内容。

## Nginx与Lua交互

Nginx的ACCESS阶段用来控制是否允许访问，这里为Lua增加一个功能，返回403禁止访问。

### 增加模块上下文结构

为模块增加一个ngx\_http\_test\_ctx\_t结构，保存执行过程中需要的一些信息。

```c
typedef struct {
    int         status;
} ngx_http_test_ctx_t;
```

有一个statu的成员，执行Lua脚本后检查status的值， 如果是403的话就返回NGX\_HTTP\_FORBIDDEN结束请求。

### 增加ngx.exit API

这里仿照lua-nginx-module的做法，增加一个方法ngx.exit, 在Lua中调用ngx.exit(403)时将status值设置为403。

那么如何在Lua中修改status的值呢？Nginx中的ngx\_http\_request\_t结构体保存了请求的所有信息，包括各个模块的上下文，将这个结构体的指针以lightuserdata的方式保存到lua\_State的全局变量中获取即可。

ngx\_http\_test\_handler中保存ngx\_http\_request\_t指针

```c
    /* 将r保存到全局变量中，key为ngx_http_test_req_key */
    lua_pushlightuserdata(tlcf->vm, r);
    lua_setglobal(tlcf->vm, ngx_http_test_req_key);
```

### ngx.exit API的实现

```c
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
```


在函数ngx\_http\_test\_create\_loc\_conf中，创建全局变量ngx， 类型为table，将ngx["exit"]值设置为函数ngx\_http\_test\_ngx\_exit。完成ngx.exit的注册，在Lua脚本中就可以通过ngx.exit()的方式调用。

```c
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

    /* 注册ngx API */
    lua_createtable(L, 0, 0);
    lua_pushcfunction(L, ngx_http_test_ngx_exit);
    lua_setfield(L, -2, "exit");

    lua_setglobal(L, "ngx");

    conf->vm = L;

    return conf;
}
```


最终的ngx\_http\_test\_module.c

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
    int         status;
} ngx_http_test_ctx_t;


static ngx_int_t
ngx_http_test_handler(ngx_http_request_t *r);
static ngx_int_t
ngx_http_test_init(ngx_conf_t *cf);
static void *
ngx_http_test_create_loc_conf(ngx_conf_t *cf);


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

    /* 将r保存到全局变量中，key为ngx_http_test_req_key */
    lua_pushlightuserdata(tlcf->vm, r);
    lua_setglobal(tlcf->vm, ngx_http_test_req_key);

    /*  加载一段Lua代码，将其编译成Lua虚拟机的字节码 */
    int ret = luaL_loadstring(tlcf->vm, (const char *)tlcf->script.data);
    if (ret != 0) {
        return NGX_ERROR;
    }

    /*  调用前面加载的Lua代码 */
    ret = lua_resume(tlcf->vm, 0);
    if (ret != 0) {
        return NGX_ERROR;
    }

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

    /* 注册ngx API */
    lua_createtable(L, 0, 0);
    lua_pushcfunction(L, ngx_http_test_ngx_exit);
    lua_setfield(L, -2, "exit");

    lua_setglobal(L, "ngx");

    conf->vm = L;

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

### 更改nginx.conf配置

```
        location / {
	    access_by_lua "ngx.exit(403)";
            root   html;
            index  index.html index.htm;
        }
```

### 编译后启动Nginx

访问时直接返回403 Forbidden.

```
$ curl 127.0.0.1
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.10.1</center>
</body>
</html>
```

同样的方法也可以用于获取请求的URL, 头部等信息。如获取请求方法的实现

```c
static int
ngx_http_test_req_get_method(lua_State *L)
{
    int status;
    status = luaL_checkint(L, 1);

    ngx_http_request_t *r;
    lua_getglobal(L, ngx_http_test_req_key);
    r = lua_touserdata(L, -1);

    lua_pushlstring(L, (char *) r->method_name.data, r->method_name.len);

    return 1;
}
```
