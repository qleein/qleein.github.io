---
layout: post
title:  "LuaJIT的变量实现-TValue"
date:   2017-09-25 22:37:00 +0800
tags: LuaJIT
categories: Openresty
---

Lua是动态类型的编程语言，变量的值可以是数值、字符串、table等所有支持的数据类型。在Lua虚拟机中每个变量都是用一个TValue结构体表示。LuaJIT出于效率的考虑重新组织了TValue结构体。


# lua-5.1中的TValue结构

lua-5.1中TValue的结构定义在lobject.h中，如下所示

```
/*
** Union of all Lua values
*/
typedef union {
  GCObject *gc;
  void *p;
  lua_Number n;
  int b;
} Value;

#define TValuefields	Value value; int tt

typedef struct lua_TValue {
  TValuefields;
} TValue;
```

TValue结构体包含了两个部分，int类型的成员tt表示类型，Value成员是一个union结构，依据类型，有不同的含义。

* 当类型位nil时，nil本身不再需要其他标识，Value成员没有意义
* 当类型为boolean时，成员b为0或1表示false或true
* 当类型为number时，成员n表示，为double类型
* 当类型为lightuserdata时，成员p，表示指针
* 当类型为function/string/userdata/table/thread等需要GC管理的类型时，成员gc表示相应GC对象的指针。

这样一个变量只要对应一个TValue结构便可以表示Lua支持的所有类型。

lua中所有的数值都是用double类型的浮点数来表示，需要占用64位的空间。再加上额外的int类型成员tt来表示类型，一个TValue结构至少需要64+32=96位的空间，如果按照8字节对其的话就需要占用128位的空间。而LuaJIT中通过Nan-boxing技术，重组了TValue，只需要占用64位的空间。

#  Nan-boxing

ieee754是使用最广泛的浮点数编码格式，它将浮点数编码成三个部分，符号、指数和尾数。如下所示

![输入图片说明](https://static.oschina.net/uploads/img/201709/24202936_go5e.jpg "在这里输入图片标题")

双精度类型即double类型，最高位为符号位，后面的11位表示指数，最低52位为尾数。三个组合表示浮点数的值。

浮点数有些特殊的值，其中之一就是NaN(Not a Number)。有些浮点数运算如0/0得到的结果就是NaN。IEEE 754标准中，如果指数部分全为1，且尾数部分不全为0时，表示值为 NaN。double类型的浮点数尾数部分有52位，NaN只要求这52位不全为0即可，只要其中一个是1剩余的51位就可以编码表示其他的含义。

实际使用的浮点数运算单元也只会产生一种NaN表示，即0xfff8_0000_0000_0000，只用了最高的13位，剩余的的51位便可以表示Lua中其他的字符串、table等。

# 内存地址的处理

TValue结构中有些成员是指针，64位系统中，指针的长度为64位，那么如何在剩下的51位中表示指针类型呢？为此LuaJIT对不同类型有两种处理方式。

## 1. 用47位地址表示指针

对于64位系统，理论上每个进程都有64位的线性地址空间，共有16,777,216TB。然而可预见的将来，操作系统和应用并不需要这么多的内存，支持如此大的地址会增加地址转换的复杂性和成本， 因此现在的实现并不允许使用全部的地址空间。

以率先实现64位架构的AMD为例，在进行内存地址转换时，只会使用地址的低48位，并且要求从第48到63的这16位需要与第47位相同。即地址必须在0到00007FFF'FFFFFFFF 和 FFFF8000'00000000 到 FFFFFFFF'FFFFFFFF这两个范围内，共有256TB的虚拟地址空间。

操作系统本身也会对内存使用进行限制，以Linux为例，将高128TB的空间划归内核使用，这样用户态进程只能低128TB的地址，如下图所示。地址的高17位皆为0，因此使用47位即可表示所有能够使用的地址。

![输入图片说明](https://static.oschina.net/uploads/img/201709/24213148_VX8B.png "在这里输入图片标题")

## 2. 只使用最低的2G的地址空间

以Linux为例，LuaJIT默认通过mmap系统调用来分配内存，对于x86_64平台的64位程序，mmap有一个MAP_32BIT标记选项，表示只分配虚拟地址空间的低2GB的空间，这样分配的内存地址，高33位皆为0, 相应的指针只需要32位的空间即可。

## 3. LuaJIT的处理与GC64模式

对于lightuserdata类型的值，LuaJIT用47位表示，对于GC类型的对象，都是通过mmap加MAP_32BIT标记分配的，用32位表示，这限制了LuaJIT只能使用不超过2GB的内存。为了摆脱这个限制，LuaJIT增加了GC64模式，开启后，所有的指针类型，包括lightuserdata都使用47位指针来表示。


# LuaJIT的TValue结构

## 默认模式下TValue布局

LuaJIT中的TValue布局如下所示

![输入图片说明](https://static.oschina.net/uploads/img/201709/24230910_NpFp.png "在这里输入图片标题")

可以通过下面的步骤判断值的类型

1. 如果最高的16位（即48到63位）不全为1，表示这是一个double类型的浮点数，
2. 最高的16位全为1，如果第47位为0时表示一个lightuserdata类型，
3. 其余情况下，高32位表示类型，低32位表示实际值。

## GC64模式下TValue布局

GC64模式开启后如下所示

![输入图片说明](https://static.oschina.net/uploads/img/201709/24231154_L05M.png "在这里输入图片标题")

这里比较特殊的是最高的13位全位1表double类型，itype表示类型，占4位，指针占用47位，总共仍是64位。

## TValue结构

LuaJIT的TValue定义在lj_obj.h中，如下面所示，因为Nan-boxing的缘故，这里的TValue结构并没有直接反映实际的内存布局。

```
/* Tagged value. */
typedef LJ_ALIGN(8) union TValue {
  uint64_t u64;		/* 64 bit pattern overlaps number. */
  lua_Number n;		/* Number object overlaps split tag/value object. */
#if LJ_GC64
  GCRef gcr;		/* GCobj reference with tag. */
  int64_t it64;
  struct {
    LJ_ENDIAN_LOHI(
      int32_t i;	/* Integer value. */
    , uint32_t it;	/* Internal object tag. Must overlap MSW of number. */
    )
  };
#else
  struct {
    LJ_ENDIAN_LOHI(
      union {
	GCRef gcr;	/* GCobj reference (if any). */
	int32_t i;	/* Integer value. */
      };
    , uint32_t it;	/* Internal object tag. Must overlap MSW of number. */
    )
  };
#endif

 ..........

} TValue;
```

## 类型的定义

所有的数据类型定义中最高的几位均为1，方便与浮点数的区分

```
#define LJ_TNIL			(~0u)
#define LJ_TFALSE		(~1u)
#define LJ_TTRUE		(~2u)
#define LJ_TLIGHTUD		(~3u)
#define LJ_TSTR			(~4u)
#define LJ_TUPVAL		(~5u)
#define LJ_TTHREAD		(~6u)
#define LJ_TPROTO		(~7u)
#define LJ_TFUNC		(~8u)
#define LJ_TTRACE		(~9u)
#define LJ_TCDATA		(~10u)
#define LJ_TTAB			(~11u)
#define LJ_TUDATA		(~12u)
/* This is just the canonical number type used in some places. */
#define LJ_TNUMX		(~13u)

/* Integers have itype == LJ_TISNUM doubles have itype < LJ_TISNUM */
#if LJ_64 && !LJ_GC64
#define LJ_TISNUM		0xfffeffffu
#else
#define LJ_TISNUM		LJ_TNUMX
#endif
```

## 判断类型的宏定义

LuaJIT定义了一系列宏用来判断值的类型。

```
#if LJ_GC64
#define itype(o)	((uint32_t)((o)->it64 >> 47))
#define tvisnil(o)	((o)->it64 == -1)
#else
#define itype(o)	((o)->it)
#define tvisnil(o)	(itype(o) == LJ_TNIL)
#endif
#define tvisfalse(o)	(itype(o) == LJ_TFALSE)
#define tvistrue(o)	(itype(o) == LJ_TTRUE)
#define tvisbool(o)	(tvisfalse(o) || tvistrue(o))
#if LJ_64 && !LJ_GC64
#define tvislightud(o)	(((int32_t)itype(o) >> 15) == -2)
#else
#define tvislightud(o)	(itype(o) == LJ_TLIGHTUD)
#endif
#define tvisstr(o)	(itype(o) == LJ_TSTR)
#define tvisfunc(o)	(itype(o) == LJ_TFUNC)
#define tvisthread(o)	(itype(o) == LJ_TTHREAD)
#define tvisproto(o)	(itype(o) == LJ_TPROTO)
#define tviscdata(o)	(itype(o) == LJ_TCDATA)
#define tvistab(o)	(itype(o) == LJ_TTAB)
#define tvisudata(o)	(itype(o) == LJ_TUDATA)
#define tvisnumber(o)	(itype(o) <= LJ_TISNUM)
#define tvisint(o)	(LJ_DUALNUM && itype(o) == LJ_TISNUM)
#define tvisnum(o)	(itype(o) < LJ_TISNUM)
```

## LJ_DUALNUM

Lua中数值都是double类型的浮点数，而实际使用时经常会用到整数，而位操作等都需要将double类型转换成整数进行。为此LuaJIT提供了LJ_DUALNUM的选项，一些数值可以直接通过int类型存储，方便使用，相当于为Lua增加了整数这个数据类型。

LJ_DUALNUM的定义可以参考lj_arch.h，不过对常用的x86_64架构，默认并没有启用LJ_DUALNUM。

# 参考

* [IEEE Standard for Floating-Point Arithmetic (IEEE 754)](https://en.wikipedia.org/wiki/IEEE_754)
* [X86_64 Virtual_address_space_details](https://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details)