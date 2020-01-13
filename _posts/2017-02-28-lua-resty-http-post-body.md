---
layout: post
title:  "lua-resty-http上传数据"
date:   2017-02-28 21:56:00 +0800
tags: Openresty lua-resty-http
categories: Openresty
---

## lua-resty-http上传数据

lua-resty-http是一个基于Openresty/ngx_lua的HTTP客户端，支持POST方法上传数据。用法很简单。

```
            content_by_lua_block {
                local http = require "resty.http"
                local httpc = http:new()
                
                local res = httpc:request_uri("http://www.baidu.com/", {
                        method = "POST",
                        body= "Hello, Lua!",
                        headers = {
                            ["User-Agent"] = "lua-resty-http",
                        }
                    })
                if not res then
                    return
                end
                ngx.log(ngx.ERR, "status: ", res.status)
                
            }
```

request_uri这个方法用于向http://www.baidu.com/发送POST请求，包体内容为"Hello, Lua!"。

### HTTP中传输包体

HTTP协议中，有包体需要发送时，需要有方法标识包体的结束，这样对方才能判断是否接收到了完整的包体。判断方法有两个。

* 包体的长度确定时，HTTP请求头部有Content-Length字段，标记包体的长度
* 包体长度不确定时，使用分块传输。请求头部有"Transfer-Encoding:chunked"表示传输包体是chunked编码的

对于lua-resty-http， 通过request_uri发送请求时包体的内容由参数中的body指定。body可以是一个lua字符串或者lua中的table。

* body为字符串时，包体长度为字符串长度，lua-resty-http会为请求添加Content-Length字段，标记包体的长度
* body为table时，lua-resty-http不会添加Content-Length头部，也不会为进行chunked编码，而是直接将table中所有成员组合成字符串发送出去

若要通过lua-resty-http以chunked编码上传数据，必须自己为数据进行chunked编码。

### chunked编码

如果一个HTTP消息（请求消息或应答消息）的Transfer-Encoding消息头的值为chunked，那么，消息体由数量未定的块组成，并以最后一个大小为0的块为结束。

每一个非空的块都以该块包含数据的字节数（字节数以十六进制表示）开始，跟随一个CRLF （回车及换行），然后是数据本身，最后块CRLF结束。在一些实现中，块大小和CRLF之间填充有白空格（0x20）。

最后一块是单行，由块大小（0），一些可选的填充白空格，以及CRLF。最后一块不再包含任何数据，但是可以发送可选的尾部，包括消息头字段。

消息最后以CRLF结尾。

例如包体为"Hello, Lua!"，长度为10, 转换成16进制为a，则编码后的结果为"a\r\nHello,Lua!\r\n0\r\n\r\n"，最后面跟的"0\r\n\r\n"没有包含任何数据，只是表示包体结束。

### lua-resty-http 上传chunked编码数据

利用lua-resty-http以chunked编码上传数据，需要手动进行chunked编码，同时设置Transfer-Encoding头部。

下面是一个示例，为了简单直接通过string.rep生成一个长字符串作为包体，string.format("%x", #raw_data)将字符串长度转换为十六进制格式。

```
            content_by_lua_block {
                local http = require "resty.http"
                local httpc = http:new()
                local raw_data = string.rep("asdfgh", 512)
                local chunked = {
                    string.format("%x", #raw_data),
                    "\r\n",
                    raw_data,
                    "\r\n",
                    "0\r\n\r\n",
                }
                
                local res = httpc:request_uri("http://www.baidu.com/", {
                        method = "POST",
                        body= chunked,
                        headers = {
                            ["User-Agent"] = "lua-resty-http",
                            ["Transfer-Encoding"] = "chunked",
                        }
                    })
                if not res then
                    return
                end
                ngx.log(ngx.ERR, "status: ", res.status)
                
            }
```
