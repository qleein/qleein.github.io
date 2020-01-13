---
layout: post
title:  "LuaJIT虚拟机-函数与原型"
date:   2018-03-10 22:59:00 +0800
tags: LuaJIT
categories: Openresty
---

Lua中的函数(或Function)，其实应该是闭包(Closure)，闭包可以认为是函数+外部变量，这里为了简单没有作区分，函数原型(或Proto)可以认为是函数的静态表示。

函数与函数原型的关系有点类似系统中进程与程序，一个程序被多次启动会创建多个进程。一个Proto可以被此加载创建多个函数。Proto是静态的，Function是动态的，在Lua中调用一个函数，总是先依照Proto创建一个函数对象，再执行这个函数对象。

## 函数原型 Proto

函数原型代表了一个函数定义，是解释器编译lua代码得到的，包括一系列lua指令、常量等的集合。

GCProto结构体代表了一个函数的原型，结构体的组成如下所示。

```c
typedef struct GCproto {
  GCHeader;
  uint8_t numparams;	/* Number of parameters. */
  uint8_t framesize;	/* Fixed frame size. */
  MSize sizebc;		/* Number of bytecode instructions. */
#if LJ_GC64
  uint32_t unused_gc64;
#endif
  GCRef gclist;
  MRef k;		/* Split constant array (points to the middle). */
  MRef uv;		/* Upvalue list. local slot|0x8000 or parent uv idx. */
  MSize sizekgc;	/* Number of collectable constants. */
  MSize sizekn;		/* Number of lua_Number constants. */
  MSize sizept;		/* Total size including colocated arrays. */
  uint8_t sizeuv;	/* Number of upvalues. */
  uint8_t flags;	/* Miscellaneous flags (see below). */
  uint16_t trace;	/* Anchor for chain of root traces. */
  /* ------ The following fields are for debugging/tracebacks only ------ */
  GCRef chunkname;	/* Name of the chunk this function was defined in. */
  BCLine firstline;	/* First line of the function definition. */
  BCLine numline;	/* Number of lines for the function definition. */
  MRef lineinfo;	/* Compressed map from bytecode ins. to source line. */
  MRef uvinfo;		/* Upvalue names. */
  MRef varinfo;		/* Names and compressed extents of local variables. */
} GCproto;
```

GCProto结构体主要成员

- GCHeader: GC类型对象的通用头部
- k: 指向常量数组中，GC类型的常量和数值类型常量的分割点，GC型常量索引位负，数值型常量索引位正。
- sizekgc: GC类型常量的数量
- sizekn: 数值型常量的数量
- uv: 指向UpValue列表
- sizeuv: UpValue的数量
- sizebc: 字节码指令的数量。指令存放在GCProto结构体后面

函数原型的内存布局如下图所示

![Proto结构示意图](https://static.oschina.net/uploads/img/201803/10225108_QvcP.png "在这里输入图片标题")

## 函数 Function

LuaJIT中函数(Function)的定义如下

```C
/* -- Function object (closures) ------------------------------------------ */

/* Common header for functions. env should be at same offset in GCudata. */
#define GCfuncHeader \
  GCHeader; uint8_t ffid; uint8_t nupvalues; \
  GCRef env; GCRef gclist; MRef pc

typedef struct GCfuncC {
  GCfuncHeader;
  lua_CFunction f;	/* C function to be called. */
  TValue upvalue[1];	/* Array of upvalues (TValue). */
} GCfuncC;

typedef struct GCfuncL {
  GCfuncHeader;
  GCRef uvptr[1];	/* Array of _pointers_ to upvalue objects (GCupval). */
} GCfuncL;

typedef union GCfunc {
  GCfuncC c;
  GCfuncL l;
} GCfunc;
```

GCfunc是一个union，封装了Lua函数和C函数，这里只关注Lua函数GCfuncL，GCfuncHeader是通用的函数头部，其中

* uint8_t nupvalues表示upvalue的个数
* Mref pc 是一个指针，指向相应的GCProto中字节码指令部分

GCfuncL还有另一个成员，`GCfunc upptr[1]`指向upvalue对象的数组。

## 函数的操作数

函数中操作的数据由三种

* 常量，保存在函数对应的原型中，通过KSTR/KNIL等指令获取
* 全局变量，保存在函数的环境中，通过GSET/GGET等指令获取/更新，可参考[Lua中的全局变量与环境](https://my.oschina.net/u/2539854/blog/857968)
* upvalue，代表函数中用到的外部变量。
* 函数参数，调用时传入

以下面的代码为例
```Lua
local code = 2
local function output()
    local a = “code:" .. tostring(code) 
    print(a)
end
```
对于函数output来说，

* `code`是upvalue，因为是在函数外层定义的
* `"a"`是局部变量
* `2`和`"code:"`是常量，会保存在Proto中
* `print`是全局变量(如果引用了一个变量，而这两变量不是局部变量，如a，也不是在函数外层定义的，如code，编译器便会把他当作全局变量)，


## 创建函数对象

依照Proto创建一个函数对象有两种情况

### 加载一个Lua脚本时创建函数

加载一个Lua脚本时，首先会调用lj_parse解析脚本生成一个Proto结构体，然后调用lj_func_newL_empty创建一个Lua函数然后执行，通过解析Lua脚本生成的函数是非嵌套函数，是没有upvalue的。

```C
/* Create a new Lua function with empty upvalues. */
GCfunc *lj_func_newL_empty(lua_State *L, GCproto *pt, GCtab *env)
{
  GCfunc *fn = func_newL(L, pt, env);
  MSize i, nuv = pt->sizeuv;
  /* NOBARRIER: The GCfunc is new (marked white). */
  for (i = 0; i < nuv; i++) {
    GCupval *uv = func_emptyuv(L);
    int32_t v = proto_uv(pt)[i];
    uv->immutable = ((v / PROTO_UV_IMMUTABLE) & 1);
    uv->dhash = (uint32_t)(uintptr_t)pt ^ (v << 24);
    setgcref(fn->l.uvptr[i], obj2gco(uv));
  }
  fn->l.nupvalues = (uint8_t)nuv;
  return fn;
}
```

### `FNEW`字节码指令创建
对于在Lua脚本中定义的函数，加载时只会生成相应的Proto，创建函数对象的操作隐藏在字节码指令中，Lua解释器会在函数调用之前插入一个`FNEW`的指令，这条指令会依照Proto创建相应的函数对象。

相应的逻辑在lj_func_new_gc中，与前一种方式最大的不同在于需要加载Upvalue，也需要继承父函数的环境。

```C
/* Do a GC check and create a new Lua function with inherited upvalues. */
GCfunc *lj_func_newL_gc(lua_State *L, GCproto *pt, GCfuncL *parent)
{
  GCfunc *fn;
  GCRef *puv;
  MSize i, nuv;
  TValue *base;
  lj_gc_check_fixtop(L);
  fn = func_newL(L, pt, tabref(parent->env));
  /* NOBARRIER: The GCfunc is new (marked white). */
  puv = parent->uvptr;
  nuv = pt->sizeuv;
  base = L->base;
  for (i = 0; i < nuv; i++) {
    uint32_t v = proto_uv(pt)[i];
    GCupval *uv;
    if ((v & PROTO_UV_LOCAL)) {
      uv = func_finduv(L, base + (v & 0xff));
      uv->immutable = ((v / PROTO_UV_IMMUTABLE) & 1);
      uv->dhash = (uint32_t)(uintptr_t)mref(parent->pc, char) ^ (v << 24);
    } else {
      uv = &gcref(puv[v])->uv;
    }
    setgcref(fn->l.uvptr[i], obj2gco(uv));
  }
  fn->l.nupvalues = (uint8_t)nuv;
  return fn;
}
```