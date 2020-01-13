---
layout: post
title:  "程序的调试信息"
date:   2018-07-22 16:14:00 +0800
tags: 调试 DWARF
categories: 日常记录
---

调试二进制程序时，经常要借助GDB工具，跟踪程序的执行流程，获取程序执行时变量的值，以发现问题所在。GDB能得到这些信息，是因为编译程序时，编译器保存了相应的信息。Linux下的可执行程序和链接库一般为ELF格式(Executable and Linking Format)，调试信息以DWARF格式保存。


## 查看ELF文件信息

新建文件main.c

内容如下

```c
#include <stdio.h>
#include <stdlib.h>

int add(int a, int b) {
	int c;
	c = a + b
	return c;
}

int main(void) {
    int a = 3;
    int b = 4;
    int c = add(a, b);
    printf("a + b = %d.\n", c);
    return 0;
}
```

通过gcc编译, `-g`表示添加调试信息

```
$ gcc -O0 -g main.c -o main
```

通过readelf命令ELF文件的Section headers

```
$ readelf -S main
There are 34 section headers, starting at offset 0x2240:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000000238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000000254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000000274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000000298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000000002b8  000002b8
       00000000000000a8  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000000360  00000360
       0000000000000084  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           00000000000003e4  000003e4
       000000000000000e  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          00000000000003f8  000003f8
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000000418  00000418
       00000000000000c0  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000000004d8  000004d8
       0000000000000018  0000000000000018  AI       5    22     8
  [11] .init             PROGBITS         00000000000004f0  000004f0
       0000000000000017  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000000510  00000510
       0000000000000020  0000000000000010  AX       0     0     16
  [13] .plt.got          PROGBITS         0000000000000530  00000530
       0000000000000008  0000000000000008  AX       0     0     8
  [14] .text             PROGBITS         0000000000000540  00000540
       00000000000001e2  0000000000000000  AX       0     0     16
  [15] .fini             PROGBITS         0000000000000724  00000724
       0000000000000009  0000000000000000  AX       0     0     4
  [16] .rodata           PROGBITS         0000000000000730  00000730
       0000000000000011  0000000000000000   A       0     0     4
  [17] .eh_frame_hdr     PROGBITS         0000000000000744  00000744
       0000000000000044  0000000000000000   A       0     0     4
  [18] .eh_frame         PROGBITS         0000000000000788  00000788
       0000000000000128  0000000000000000   A       0     0     8
  [19] .init_array       INIT_ARRAY       0000000000200db8  00000db8
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000200dc0  00000dc0
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000200dc8  00000dc8
       00000000000001f0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000200fb8  00000fb8
       0000000000000048  0000000000000008  WA       0     0     8
  [23] .data             PROGBITS         0000000000201000  00001000
       0000000000000010  0000000000000000  WA       0     0     8
  [24] .bss              NOBITS           0000000000201010  00001010
       0000000000000008  0000000000000000  WA       0     0     1
  [25] .comment          PROGBITS         0000000000000000  00001010
       0000000000000024  0000000000000001  MS       0     0     1
  [26] .debug_aranges    PROGBITS         0000000000000000  00001034
       0000000000000030  0000000000000000           0     0     1
  [27] .debug_info       PROGBITS         0000000000000000  00001064
       0000000000000396  0000000000000000           0     0     1
  [28] .debug_abbrev     PROGBITS         0000000000000000  000013fa
       000000000000011c  0000000000000000           0     0     1
  [29] .debug_line       PROGBITS         0000000000000000  00001516
       00000000000000da  0000000000000000           0     0     1
  [30] .debug_str        PROGBITS         0000000000000000  000015f0
       0000000000000289  0000000000000001  MS       0     0     1
  [31] .symtab           SYMTAB           0000000000000000  00001880
       0000000000000678  0000000000000018          32    48     8
  [32] .strtab           STRTAB           0000000000000000  00001ef8
       0000000000000208  0000000000000000           0     0     1
  [33] .shstrtab         STRTAB           0000000000000000  00002100
       000000000000013e  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

```

编译生成的可执行文件中有34个Section, 其中的`debug_*`几个ELF头部代表程序的DWARF格式的调试信息


## 查看DWARF信息

### 查看debug_line Section

`debug_line` Section中记录了二进制程序的指令地址对应源代码的位置。可以通过`readelf -wl main`查看

```
$ readelf -wl main
Raw dump of debug contents of section .debug_line:

  Offset:                      0x0
  Length:                      214
  DWARF Version:               2
  Prologue Length:             179
  Minimum Instruction Length:  1
  Initial value of 'is_stmt':  1
  Line Base:                   -5
  Line Range:                  14
  Opcode Base:                 13

 Opcodes:
  Opcode 1 has 0 args
  Opcode 2 has 1 arg
  Opcode 3 has 1 arg
  Opcode 4 has 1 arg
  Opcode 5 has 1 arg
  Opcode 6 has 0 args
  Opcode 7 has 0 args
  Opcode 8 has 0 args
  Opcode 9 has 1 arg
  Opcode 10 has 0 args
  Opcode 11 has 0 args
  Opcode 12 has 1 arg

 The Directory Table (offset 0x1b):
  1	/usr/lib/gcc/x86_64-linux-gnu/7/include
  2	/usr/include/x86_64-linux-gnu/bits
  3	/usr/include

 The File Name Table (offset 0x74):
  Entry	Dir	Time	Size	Name
  1	0	0	0	main.c
  2	1	0	0	stddef.h
  3	2	0	0	types.h
  4	2	0	0	libio.h
  5	3	0	0	stdio.h
  6	2	0	0	sys_errlist.h

 Line Number Statements:
  [0x000000bd]  Extended opcode 2: set Address to 0x64a
  [0x000000c8]  Special opcode 8: advance Address by 0 to 0x64a and Line by 3 to 4
  [0x000000c9]  Special opcode 146: advance Address by 10 to 0x654 and Line by 1 to 5
  [0x000000ca]  Special opcode 160: advance Address by 11 to 0x65f and Line by 1 to 6
  [0x000000cb]  Special opcode 48: advance Address by 3 to 0x662 and Line by 1 to 7
  [0x000000cc]  Special opcode 35: advance Address by 2 to 0x664 and Line by 2 to 9
  [0x000000cd]  Special opcode 118: advance Address by 8 to 0x66c and Line by 1 to 10
  [0x000000ce]  Special opcode 104: advance Address by 7 to 0x673 and Line by 1 to 11
  [0x000000cf]  Special opcode 104: advance Address by 7 to 0x67a and Line by 1 to 12
  [0x000000d0]  Advance PC by constant 17 to 0x68b
  [0x000000d1]  Special opcode 20: advance Address by 1 to 0x68c and Line by 1 to 13
  [0x000000d2]  Advance PC by constant 17 to 0x69d
  [0x000000d3]  Special opcode 76: advance Address by 5 to 0x6a2 and Line by 1 to 14
  [0x000000d4]  Special opcode 76: advance Address by 5 to 0x6a7 and Line by 1 to 15
  [0x000000d5]  Advance PC by 2 to 0x6a9
  [0x000000d7]  Extended opcode 1: End of Sequence


```

注意这里面的`Line Number Statements`，里面的每一行都标明了一条指定的地址和对应的源代码再文件中的位置。

`[0x000000c8]  Special opcode 8: advance Address by 0 to 0x64a and Line by 3 to 4`
这一行表示指定地址为`0x64a`的指令对应源代码在文件中的第4行


对`debug_line` section处理后，可以得到指令地址到源代码位置的转换表，可以通过`readelf -wL main`查看处理后的结果(指令地址和源代码的对应关系)。

```
$ readelf -wL main
Contents of the .debug_line section:

CU: ./main.c:
File name                            Line number    Starting address    View
main.c                                         4               0x64a
main.c                                         5               0x654
main.c                                         6               0x65f
main.c                                         7               0x662
main.c                                         9               0x664
main.c                                        10               0x66c
main.c                                        11               0x673
main.c                                        12               0x67a
main.c                                        13               0x68c
main.c                                        14               0x6a2
main.c                                        15               0x6a7
main.c                                        15               0x6a9

```

其中每行都指明了文件名、第X行、指令地址。


### 查看`debug_info section`

通过`readelf -wi main`可以读取`debug_info` section的内容。debug_info section是DWARF的核心内容，其他一些如`debug_str`等都是为了加快查找/压缩空间而使用的。

```
$ readelf -wi main
Contents of the .debug_info section:

  Compilation Unit @ offset 0x0:
   Length:        0x392 (32-bit)
   Version:       4
   Abbrev Offset: 0x0
   Pointer Size:  8
 <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
    <c>   DW_AT_producer    : (indirect string, offset: 0x21a): GNU C11 7.3.0 -mtune=generic -march=x86-64 -g -O0 -fstack-protector-strong
    <10>   DW_AT_language    : 12	(ANSI C99)
    <11>   DW_AT_name        : (indirect string, offset: 0x118): main.c
    <15>   DW_AT_comp_dir    : (indirect string, offset: 0xdc): /home/kenan
    <19>   DW_AT_low_pc      : 0x64a
    <21>   DW_AT_high_pc     : 0x5f
    <29>   DW_AT_stmt_list   : 0x0


...................
...................

 <1><353>: Abbrev Number: 20 (DW_TAG_subprogram)
    <354>   DW_AT_external    : 1
    <354>   DW_AT_name        : add
    <358>   DW_AT_decl_file   : 1
    <359>   DW_AT_decl_line   : 4
    <35a>   DW_AT_prototyped  : 1
    <35a>   DW_AT_type        : <0x62>
    <35e>   DW_AT_low_pc      : 0x64a
    <366>   DW_AT_high_pc     : 0x1a
    <36e>   DW_AT_frame_base  : 1 byte block: 9c 	(DW_OP_call_frame_cfa)
    <370>   DW_AT_GNU_all_call_sites: 1
 <2><370>: Abbrev Number: 21 (DW_TAG_formal_parameter)
    <371>   DW_AT_name        : a
    <373>   DW_AT_decl_file   : 1
    <374>   DW_AT_decl_line   : 4
    <375>   DW_AT_type        : <0x62>
    <379>   DW_AT_location    : 2 byte block: 91 5c 	(DW_OP_fbreg: -36)
 <2><37c>: Abbrev Number: 21 (DW_TAG_formal_parameter)
    <37d>   DW_AT_name        : b
    <37f>   DW_AT_decl_file   : 1
    <380>   DW_AT_decl_line   : 4
    <381>   DW_AT_type        : <0x62>
    <385>   DW_AT_location    : 2 byte block: 91 58 	(DW_OP_fbreg: -40)
 <2><388>: Abbrev Number: 19 (DW_TAG_variable)
    <389>   DW_AT_name        : c
    <38b>   DW_AT_decl_file   : 1
    <38c>   DW_AT_decl_line   : 5
    <38d>   DW_AT_type        : <0x62>
    <391>   DW_AT_location    : 2 byte block: 91 6c 	(DW_OP_fbreg: -20)
 <2><394>: Abbrev Number: 0

****
```

这里是由一个个的称为DIE(Debuging Information Entity)的单元来表示的，其中TAG表示DIE的类型，如DW_TAG_compile_unit中包含了编译时的参数，源文件，目录等。这里只关注其中DW_TAG_subsystem，即函数信息。
* `DW_AT_name: add`: 函数名为add
* `DW_AT_low_pc: 0x64a`： 函数对应的初始PC地址
* `DW_AT_high_pc: 0x1a`: 函数结束时PC地址为0x64a + 0x1a
* `DW_AT_frame_base`: 表达函数的栈帧基址(frame base)，函数参数和局部变量的存储位置会以相对栈帧基址的偏移给出。

在DW_TAG_subprogram后面是函数中变量的信息 `DW_TAB_formal_parameter`表示这个DIE代表的时函数的参数
* `DW_AT_name: a`: 参数名为a
* `DW_AT_type： 0x62`: 参数类型
* `DW_AT_location    : 2 byte block: 91 5c 	(DW_OP_fbreg: -36)`: 表示存储在函数栈帧基址偏移`-24`的地方，

有了这些信息，GDB就可以根据当前执行的指令地址得到对应的源代码文件位置、当前函数名以及当前函数中的参数/局部变量/全局变量的信息。

## 调试符号单独保存

GDB允许将调试信息保存在单独的文件中。因为调试信息占用的空间会很大，甚至远超过程序本身。很多系统会将调试信息剥离到单独的文件中，需要调试时再安装，以节约存储空间。

### 剥离出调试信息

通过objcopy可以将ELF文件的调试信息提取到单独的文件中

```
$ objcopy --only-keep-debug main main.debug
```

然后通过strip去除文件中的调试信息

```
$ strip -g main
```

### 寻找带有调试信息的文件路径

GDB支持使用下面两种方式来寻找调试信息所在的文件

#### 通过`build-id`指定

```
$ readelf -n main

Displaying notes found in: .note.ABI-tag
  Owner                 Data size	Description
  GNU                  0x00000010	NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0

Displaying notes found in: .note.gnu.build-id
  Owner                 Data size	Description
  GNU                  0x00000014	NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 536cc8d42fa3ed672abc427d4a683313fb902b6b
```

在`.note.gnu.build-id`中记录了build-id为`536cc8d42fa3ed672abc427d4a683313fb902b6b`，GDB会尝试从
`/usr/lib/debug/.build-id/53/6cc8d42fa3ed672abc427d4a683313fb902b6b.debug`文件中读取调试信息。

在`/usr/lib/debug`目录中，以build-id的头两位为子目录名，后面的几位+`.debug`为文件名。

#### 通过`debug_link`指定

通过`debug_link`指定文件名。如指定为main.debug，则GDB会在下面三个路径下寻找debug文件

- /usr/bin/main.debug
- /usr/bin/.debug/main.debug
- /usr/lib/debug/usr/bin/main.debug

可通过`objcopy`命令将debug_link添加到程序中

```
$ objcopy --add-gnu-debuglink=main.debug main
```


然后可以看到ELF头部已有`gnu_debuglink`

```
$ readelf -S main
.........
  [26] .gnu_debuglink    PROGBITS         0000000000000000  00001034
       0000000000000010  0000000000000000           0     0     4
........
```

## 链接

* [DWARF4标准](http://www.dwarfstd.org/doc/DWARF4.pdf)
* [DWARF, 说不定你也需要它哦](https://www.jianshu.com/p/20dfe4fe1b3f)
* [Debugging Information in Separate Files](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)
* [readelf命令](http://man7.org/linux/man-pages/man1/readelf.1.html)