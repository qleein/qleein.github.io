---
layout: post
title:  "gcc链接选项 链接静态库和动态库"
date:   2017-03-09 00:01:00 +0800
---

## 链接库

链接指定的库有两种方式

* -llibrary
* -l library

如链接LuaJIT库，可以用-lluajit-5.1，此时gcc会在库路径中查找libluajit-5.1.so或者libluajit-5.1.a。 也可以用-l llibluajit-5.1.a，第二种只能用在POXIS上，推荐使用第一种方式。而且第二种方式只会在特定的目录进行搜索，会发生找不到库的情况。

通过-llibrary链接库时，可能既有静态库如libluajit-5.1.a，也有动态库如libluajit-5.1.so，这是链接器会优先链接动态库。

## 指定库路径

如果需要的库不在系统的库搜索路径下，就需要通过-L指定库的搜索路径。
如lua解释器安装时会把libluajit-5.1.a，和libluajit-5.1.so等放在/usr/local/lib下，而gcc进行链接时默认不会在这个路径下搜索库，导致链接失败。通过-L/usr/local/lib指定库路径就可以解决这个问题。
```
~$ gcc main.c -lluajit-5.1             
/usr/bin/ld: 找不到 -lluajit-5.1
collect2: error: ld returned 1 exit status

~$ gcc main.c -L/usr/local/lib -lluajit-5.1
```

## 指定运行时库搜索路径

有时通过-L指定库路径可以正常链接，运行程序时却因为找不到库报错
```
~ $ ./a.out 
./a.out: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
~ $ ldd ./a.out 
	linux-vdso.so.1 =>  (0x00007ffc1638e000)
	libluajit-5.1.so.2 => not found
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4c23d48000)
	/lib64/ld-linux-x86-64.so.2 (0x000055713bf16000)

```
使用ldd命令可以看到运行时找不到libluajit-5.1.so.2，无法运行，对此可以通过-Wl,-rpath指定运行时库的搜索路径

```
kenan@kenan-desktop ~ $ gcc test.c -I/usr/local/include/luajit-2.1 -Wl,-rpath=/usr/local/lib/ -lluajit-5.1
kenan@kenan-desktop ~ $ ./a.out 
Hello, Lua!
~ $ 
~ $ ldd ./a.out 
	linux-vdso.so.1 =>  (0x00007ffd6c5d7000)
	libluajit-5.1.so.2 => /usr/local/lib/libluajit-5.1.so.2 (0x00007f5ea238c000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f5ea1fa5000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f5ea1c9b000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f5ea1a97000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f5ea1881000)
	/lib64/ld-linux-x86-64.so.2 (0x0000565491551000)

```
再使用ldd命令查看，libluajit-5.1.so.2在/sur/local/lib下。

## 使用静态链接

对于运行时找不到动态库的问题，除了通过-Wl,-rpath指定路径外，还可以通过静态链接库的方式来解决。实际生产环境中通常只在一台机器上编译二进制程序，再将二进制程序分发到线上运行环境。程序需要的库可能实际线上环境中根本没有，也需要将库静态链接。

静态链接库有下面几种方式。

### 只保留静态库
如安装luajit后，在/usr/local/lib下既有静态库，也有动态库。链接时优先链接动态库。如果将动态库删除，只保留静态库，这时链接器就会链接静态库.

### 通过-Wl,-Bstatic对指定的库使用静态链接
gcc会对-Wl,-Bstatic 后面的库使用静态链接。对-Wl,-Bdynamic后面跟的库使用动态链接。
如果需要对指定的库使用静态链接，其他的库使用默认的动态链接，可以这样用
```
gcc test  -Wl,-Bstatic -lluajit-5.1 -Wl,-Bdynamic -lxxx -lxxx
```
这样gcc会链接libluajit-5.1.a，而对其他库使用动态链接。

### 使用 -static 避免动态链接

gcc编译时使用-static参数，会阻止使用动态链接的方式。与之前两个方法不同，-static参数会导致gcc对所有的库使用静态链接，一般不推荐使用这种方式。
```
~ $ gcc test.c -I/usr/local/include/luajit-2.1 -Wl,-rpath=/usr/local/lib/ -L/usr/local/lib -lluajit-5.1 -ldl -lm -static
~ $ ldd ./a.out 
	不是动态可执行文件
```
对程序使用ldd命令提示不是动态可执行文件

没有使用-static时ldd可以看到程序需要的动态库，以及库的路径。
```
~ $ gcc test.c -I/usr/local/include/luajit-2.1 -Wl,-rpath=/usr/local/lib/ -L/usr/local/lib -lluajit-5.1 -ldl -lm     
~ $ ldd ./a.out 
	linux-vdso.so.1 =>  (0x00007ffd057f1000)
	libluajit-5.1.so.2 => /usr/local/lib/libluajit-5.1.so.2 (0x00007f1e72a48000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1e7267f000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f1e72375000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f1e72171000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f1e71f5b000)
	/lib64/ld-linux-x86-64.so.2 (0x000055875c7a0000)
```

### 使用 -static-libxxx对某些系统库进行静态链接

gcc本身提供了参数可以只对libgcc，libstdc++等库进行静态链接
主要有下面这些
```
-static-libgcc

-static-libasan

-static-libtsan

-static-liblsan

-static-libubsan

-static-libmpx

-static-libmpxwrappers

-static-libstdc++
```

更详细的介绍可以参见[官方文档](https://gcc.gnu.org/onlinedocs/gcc-5.4.0/gcc/Link-Options.html#Link-Options)