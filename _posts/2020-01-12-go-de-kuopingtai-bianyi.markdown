---
layout: post
title:  "Go实现的库的跨平台编译调用"
date:   2020-01-12 14:13:00 +0800
tags: Go
categories: 日常记录
---

通过cgo可以将go的程序编译成库，在其他程序，如C程序中调用。cgo本身就提供了多平台的支持。不过对于每个平台还需要有相应的C编译工具链的支持，对不同平台的支持程度也不一致，需要针对每个平台单独处理。


新建文件 `lib.go`，通过import C启用cgo，export指定需要导出的方法。cgo编译后会生成相应的头文件，在C程序中包含这个头文件，链接时链接生成的库即可使用。

```go
package main

import "C"

//export add
func add(a, b C.int) C.int {
    return a + b
}

func main() {
}
```

编译Linux平台的库
================

Linux平台的编译最为顺利，有其实就在Linux上操作，只要安装gcc即可。

1. 安装C编译器

因为本身就64位的系统，默认安装gcc即可，如果需要编译32位的库，则需要单独安装针对32位版本的gcc。

```
$ sudo apt install gcc
```

2. 编译静态库

```
$ CGO_ENABLED=1 GOOS=linux GOARCH=amd64 CC=gcc go build -buildmode=c-archive -o libcgotest.a lib.go
```

3. 编译动态库

```
$ CGO_ENABLED=1 GOOS=linux GOARCH=amd64 CC=gcc go build -buildmode=c-shared -o libcgotest.so lib.go
```

Linux下无论静态库还是动态库，都可以直接使用。


编译windows平台的库
================

生成windows的库，主要参考了这个[博客](https://blog.csdn.net/qq_30549833/article/details/86157744)提出的方法。

1. 安装编译器

cgo不支持使用微软的vc++编译，所以可以通过MinGW或者Mingw-w64来安装针对windows平台的gcc编译链。因为MinGW已经不再维护，推荐使用mingw-w64。

如果使用Ubuntu，直接通过apt安装即可，执行`sudo apt install mingw-w64`即可安装。
如果使用windows，访问[这里](http://mingw-w64.org/doku.php/download/mingw-builds)下载mingw-builds安装。

2. 编译静态库

```
$ CGO_ENABLED=1 GOOS=windows GOARCH=386 CC=i686-w64-mingw32-gcc go build -buildmode=c-archive -o libcgotest.a lib.go
```

3. 导出dll库

直接编译成的静态库，无法在vc++中使用，如果需要使用vc++或者在visual studio中使用，需要编译成dll。

新建导出库文件libcgotest.def
```
EXPORT
	add
```

通过下面的命令生成dll在visual studio使用即可。
```
$ gcc libcgotest.def libcgotest.a -shared -winmm -o libcgotest.dll -Wl,--out-implib,libcgotest.lib
```

* 关于调试符号

windows的 vc++和gcc的调试符号格式不兼容，所以在visual stuiod调试的话，是看不到cgo编译的库的信息的。

* visual stduio在release模式下运行报错

visual stduio在release模式下可能在运行时会报错 `无法定位程序输入点CoInitialize于动态链接库XXXX上`，对此可以在项目属性页-配置属性-链接器-优化中，把引用选项设置为否，即`/OPT:NOREF`。

* 关于64位架构

针对windows编译64位的库，按照上面的步骤可以编译成功，但是运行时会不断报出运行时错误，无论是Debug模式还是Release模式，亦或是用MinGW还是Mingw-w64编译均是如此。只能怀疑是生成64位的库时本身就有兼容性问题。

* 关于直接编译dll库

可以直接通过buildmode=c-shared编译dll库，不过需要再从dll库生成lib导入库，因为编译链接的时候是需要.lib导入库的。

编译android平台的库
================


针对android平台，cgo直接编译出的库不能直接在Java中调用，需要自己再封装一层针对Java的JNI的方法，封装方法网上已有很多资料，这里不再详述。

* 编译动态库

1. 下载安装NDK

首先需要安装ndk，可以直接在[android官方网站](https://developer.android.com/ndk/downloads?hl=zh-cn)下载。这里用的r16b的版本。

2. 构建编译环境

下载完成后，构建编译环境。下面的命令创建arheabi的编译环境，使用的api版本是api16。arm-linux-android-api16为生成目录名，亦可随意指定。
```
$ export NDK_DIR=/path/to/ndk
$ $NDK_DIR/build/tools/make_standalone_toolchain.py --arch arm --api 16 --install-dir arm-linux-android-api16
```

3. 编译动态库。

和其他平台一样，需要通过CC、GOOS和GOARCH指定编译器和目标平台。额外的一个是要通过snname指定so名。否则在高版本的android上运行时会报找不到so库的错误。

```
$ CGO_ENABLED=1 GOOS=android GOARCH=arm CC=arm-linux-android-api16/bin/clang CGO_LDFLAGS='-Wl,--soname,libcgotest.so' go build -buildmode=c-shared  -o libcgotest.so lib.go
```

编译完成后，只要在原本的JNI的编译脚本中加入这个动态库即可。其他目标架构编译的方式相同，唯一的区别是构建编译环境时，根据项目需要指定不同的架构和支持的api版本。

* 运行时报找不到库

先确认编译时通过CGO_LDFLAGS指定了snname，然后确认打包的apk中包含生成的so文件

* 关于使用静态库

先说结论，行不通.....

开始想通过cgo来编译静态库，这样现有的项目在Andoird.mk中直接添加静态库即可。然而，编译的时候发现行不通。

在android下cgo不支持编译静态库，即设置buildmode=c-archive会报不支持。很奇怪go的编译平台是支持android的，却不支持静态库。在github上有人在[这里](https://github.com/golang/go/issues/33806)提了同样的问题，项目的成员回复可以先更改go的源码里的cmd/go/build.go和cmd/dist/test.go，尝试编译看能不能工作。 花了半天时间发现，这个是在go工具包的cmd/link命令判断的，更改后编译静态库成功。但是在这个静态库基础上再编译so时，又在报`text relocation`的错误，可以通过编译选项忽略这个错误，但是实际运行的时候，android api25以下的版本会有警告，更新的版本程序直接crash。

