---
layout: post
title:  "欢迎来到JIT的世界: The Joy of Simple JITs"
date:   2017-07-22 20:22:00 +0800
---

这个例子展示了简单的JIT(即时编译器)可以多么简单和有趣。JIT这个词让人联想到高深的魔法，只有顶尖的编译器团队才会想到使用。你可能会想到JVM或者.NET这样有数十万行代码的庞大的运行时库。你看不到像"Hello, World!"那样的JIT, 通过简短的代码做些有趣的事情。这篇文章尝试改变这个现状。

一个JIT和一个调用`printf`的程序没有本质的区别，只是JIT产生的是机器代码，而不是像"Hello, World!"这样的消息。确实，像JVM这样的JIT是及其复杂的怪兽，但这是因为他们实施了一个复杂的平台并做了积极的优化。如果做的事情很简单，我们的程序同样可以很简单。

实现一个简单的JIT最困难的部分是编写你的目标CPU可以理解的指令。例如在x86-64平台，`push rbp`这个指令被编码成`0x55`。这样的编码是令人厌烦的，还需要阅读很多CPU手册，所以我们将跳过这个部分。我们将使用Mile Pall开发的一个工具`DynASM`来完成这个工作。DynASM采用了一个新颖的方式， 让你可以在JIT中混合使用汇编代码和C代码，从而可以用一个非常自然和可读的方式实现JIT。它支持很多CPU架构（如x86, x86-64, PowerPC, MIPS和ARM），所以你不会因为它对硬件的支持而受到限制。DynASM也格外小巧， 其整个运行时库都包含在500行的头文件中。

我应该简要地澄清一下我的术语。我将任何在运行时生成机器代码并执行这些机器代码的程序成为"JIT"。一些作者会在特定的地方使用这个词，认为只有根据需要生成小段机器码的解释器/编译器才叫做JIT。这些人会更宽泛的将运行时生成代码的技术成为动态编译。但是"JIT"是更常见和接受的术语，通常用于不符合 "JIT"最严格定义的地方，如Berkeley Packet Filter JIT。

# Hello, JIT World!

不用多说，让我们来实现我们的第一个JIT。这个和所有其他的程序都在我的Github仓库[jitdemo](https://github.com/haberman/jitdemo)。代码是Unix风格的，因为我们使用`mmap()`，也需要生成x86-64平台的代码，所以你需要一个支持该平台的处理器和操作系统。我已经测试过它可以在Ubuntu Linux和Mac OS X上使用。

在第一个例子中，我们甚至不需要使用DynASM以保证它足够简单。这个程序在文件jit1.c中。

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>

int main(int argc, char *argv[]) {
  // Machine code for:
  //   mov eax, 0
  //   ret
  unsigned char code[] = {0xb8, 0x00, 0x00, 0x00, 0x00, 0xc3};

  if (argc < 2) {
    fprintf(stderr, "Usage: jit1 <integer>\n");
    return 1;
  }

  // Overwrite immediate value "0" in the instruction
  // with the user's value.  This will make our code:
  //   mov eax, <user's value>
  //   ret
  int num = atoi(argv[1]);
  memcpy(&code[1], &num, 4);

  // Allocate writable/executable memory.
  // Note: real programs should not map memory both writable
  // and executable because it is a security risk.
  void *mem = mmap(NULL, sizeof(code), PROT_WRITE | PROT_EXEC,
                   MAP_ANON | MAP_PRIVATE, -1, 0);
  memcpy(mem, code, sizeof(code));

  // The function will return the user's value.
  int (*func)() = mem;
  return func();
}
```

或许难以置信，这个33行的程序确实是一个JIT。它动态地生成了一个返回运行时指定的整数值的函数，然后运行它。你可以验证它能够工作。

```
$ ./jit1 42 ; echo $?
42
```

你应该会注意到我使用`mmap()`来分配内存，而不像通常的做法使用`malloc()`从堆上获取。这是必须的，因为我需要让获得的内存可以被执行，这样我就可以跳转到这里而不引起程序的崩溃。在大部分系统上栈和堆被配置成不可以执行，因为跳转到栈或堆上意味着有严重的错误发生。更糟糕的是，可执行的栈让hacker更容易利用缓冲区溢出漏洞。因此我们通过需要避免映射即可写又可执行的内存，你自己的程序也最好遵守这个习惯。这里为了让我们的第一个程序保持简单，我打破了上面的规则。

我也没有释放我分配的内存，我们将尽快解决这个问题。`mmap()`有一个相应的函数`munmap()`，我们可以使用它释放内存到操作系统。

你或许会疑惑为什么不调用一个函数更改你通过`malloc()`获得的内存的权限。通过完全不同的方式获得可执行的内存听起来想是
累赘。其实有一个函数可以更改你获得的内存的权限，叫做`mprotect()`。但是内存的权限只以内存页为单位生效，而`malloc()`分配的内存只是一个完整内存页的一部分。如果你更改了内存页的权限会影响这个内存页上的其他代码。

# Hello, DynASM World!

DynASM是LuaJIT项目中的一部分，但完全独立于LuaJIT代码，可以单独使用。它由两部分组成：一个预处理器将混合的C /汇编文件（* .dasc）转换为C代码，和一个运行时库来执行必须在运行时执行的工作。

![输入图片说明](https://static.oschina.net/uploads/img/201707/20211325_kOOC.png)

这个设计很棒，解析汇编语言和编码机器指令的复杂代码可以用高级的带有垃圾回收机制的语言(Lua)来编写，而且只在编译时需要，运行时不依赖Lua。大多数DynASM可以用Lua编写，而运行时又不需要依赖于Lua。

作为我们的第一个DynASM的例子，我写了一个程序生成和上一个例子中同样功能的函数。我们可以比较两种方式的差异，理解`DynASM`为我们带来了什么。

```C
// DynASM directives.
|.arch x64
|.actionlist actions

// This define affects "|" DynASM lines.  "Dst" must
// resolve to a dasm_State** that points to a dasm_State*.
#define Dst &state

int main(int argc, char *argv[]) {
  if (argc < 2) {
    fprintf(stderr, "Usage: jit1 <integer>\n");
    return 1;
  }

  int num = atoi(argv[1]);
  dasm_State *state;
  initjit(&state, actions);

  // Generate the code.  Each line appends to a buffer in
  // "state", but the code in this buffer is not fully linked
  // yet because labels can be referenced before they are
  // defined.
  //
  // The run-time value of C variable "num" is substituted
  // into the immediate value of the instruction.
  |  mov eax, num
  |  ret

  // Link the code and write it to executable memory.
  int (*fptr)() = jitcode(&state);

  // Call the JIT-ted function.
  int ret = fptr();
  assert(num == ret);

  // Free the machine code.
  free_jitcode(fptr);

  return ret;
}
```

这个不是程序的全部内容，在`dynasm-driver.c`中定义了初始化DynASM和分配/释放可执行内存的一些辅助功能。这些公共的辅助代码在我们所有的例子中都是相同的，所以我们这里省略它。在仓库中它们很直观，也很容易理解。

需要注意的最主要的区别是我们生成指令的方式。像汇编语言的`.S`文件类似，我们的`.dasc`文件包含了汇编语言。以(`|`)开头的地方由DynASM来翻译，可以包含汇编指令。相比我们第一个例子这是一个巨大的进步。特别要注意的是，`mov`指令的一个参数引用的是C中的一个变量，DynASM在生成指令时知道如何将参数替换成这个变量。

为了弄清这是如何实现的，我们看下预处理器生成的`jit2.h`(从jit2.dasc生成)。我摘录了有趣的部分，文件的其余部分没有修改。

```C
//|.arch x64
//|.actionlist actions
static const unsigned char actions[4] = {
  184,237,195,255
};

// [...]

//|  mov eax, num
//|  ret
dasm_put(Dst, 0, num);
```



这里我们看到我们在`.dasc`文件（现在已注释掉）中写入的源代码行以及由它们生成的行。 “action list”是由DynASM预处理器生成的数据缓冲区，它是由DynASM运行时解释的字节码，其中掺杂了你的汇编语言指令的编码和DynASM运行时链接代码、插入运行时参数的方法。 在这种情况下，我们的actions中的四个字节被解释为：

* 184 – x86平台上执行`mov eax [immediate]`指令的第一个字节
* 237 – DynASM的指令`DASM_IMM_D`, 表示`dasm_put`的下一个参数将作为上面mov指令的第二个参数([immediate])的值，补全`mov`指令。
* 195 – x86平台`ret`指令
* 255 – DynASM的字节码指令`DASM_STOP`，表示编码中止。


然后`actions`会被实际生成汇编指令的代码引用。以`|`开头的指令行会被`dasm_pus()`函数替换，`dasm_put()`提供了在`actions`数组中的偏移和运行时数据到输出中。`dasm_put()`会把这些指令(和运行时数据如num)追加到`Dst &state`的缓冲区中。如这里的`dasm_put(Dst, 0, num)`，`Dst`表示`state`地址，`0`表示在`actions`中的偏移，`num`被作为`mov eax [immediate]`的第二个参数。

我们最终得到了与第一个示例完全相同的效果，但这次我们使用了一种方法，可以让我们利用符号来编写汇编语言。这是一种更好的编程JIT的方法。

# A Simple JIT for Brainf*ck

我们的目标是一个最简单的图灵完备的语言，命名为Branf*ck(简称BF)。BF仅用8个指令实现图灵完备(甚至包括I/O)。这些指令可以被认为是另一种格式的字节码。

没有比我们最后一个例子更复杂的了，我们将有一个不到100行代码实现的全功能的JIT(不包括公共的dynasm-driers.c的不到70行)。

```C
#include <stdint.h>

|.arch x64
|.actionlist actions
|
|// Use rbx as our cell pointer.
|// Since rbx is a callee-save register, it will be preserved
|// across our calls to getchar and putchar.
|.define PTR, rbx
|
|// Macro for calling a function.
|// In cases where our target is <=2**32 away we can use
|//   | call &addr
|// But since we don't know if it will be, we use this safe
|// sequence instead.
|.macro callp, addr
|  mov64  rax, (uintptr_t)addr
|  call   rax
|.endmacro

#define Dst &state
#define MAX_NESTING 256

void err(const char *msg) {
  fprintf(stderr, "%s\n", msg);
  exit(1);
}

int main(int argc, char *argv[]) {
  if (argc < 2) err("Usage: jit3 <bf program>");
  dasm_State *state;
  initjit(&state, actions);

  unsigned int maxpc = 0;
  int pcstack[MAX_NESTING];
  int *top = pcstack, *limit = pcstack + MAX_NESTING;

  // Function prologue.
  |  push PTR
  |  mov  PTR, rdi

  for (char *p = argv[1]; *p; p++) {
    switch (*p) {
      case '>':
        |  inc  PTR
        break;
      case '<':
        |  dec  PTR
        break;
      case '+':
        |  inc  byte [PTR]
        break;
      case '-':
        |  dec  byte [PTR]
        break;
      case '.':
        |  movzx edi, byte [PTR]
        |  callp putchar
        break;
      case ',':
        |  callp getchar
        |  mov   byte [PTR], al
        break;
      case '[':
        if (top == limit) err("Nesting too deep.");
        // Each loop gets two pclabels: at the beginning and end.
        // We store pclabel offsets in a stack to link the loop
        // begin and end together.
        maxpc += 2;
        *top++ = maxpc;
        dasm_growpc(&state, maxpc);
        |  cmp  byte [PTR], 0
        |  je   =>(maxpc-2)
        |=>(maxpc-1):
        break;
      case ']':
        if (top == pcstack) err("Unmatched ']'");
        top--;
        |  cmp  byte [PTR], 0
        |  jne  =>(*top-1)
        |=>(*top-2):
        break;
    }
  }

  // Function epilogue.
  |  pop  PTR
  |  ret

  void (*fptr)(char*) = jitcode(&state);
  char *mem = calloc(30000, 1);
  fptr(mem);
  free(mem);
  free_jitcode(fptr);
  return 0;
}
```


在这个程序中我们确实看到dynasm令人眼前一亮的做法。我们可以混合使用C和汇编实现一个漂亮的、可读的代码生成器。

比较一下前面提到的Berkeley Packet Filter JIT的代码，它的代码生成有类似的结构(一个巨大的`switch()`语句，case中使用字节码)，但是没有DynASM，代码必须手动去编码指令。包含的符号化的指令只是用作注释，读者只能假定它是正确的。在Linux内核中的arch/x86/net/bpf_jit_comp.c。

```C
    switch (filter[i].code) {
    case BPF_S_ALU_ADD_X: /* A += X; */
            seen |= SEEN_XREG;
            EMIT2(0x01, 0xd8);              /* add %ebx,%eax */
            break;
    case BPF_S_ALU_ADD_K: /* A += K; */
            if (!K)
                    break;
            if (is_imm8(K))
                    EMIT3(0x83, 0xc0, K);   /* add imm8,%eax */
            else
                    EMIT1_off32(0x05, K);   /* add imm32,%eax */
            break;
    case BPF_S_ALU_SUB_X: /* A -= X; */
            seen |= SEEN_XREG;
            EMIT2(0x29, 0xd8);              /* sub    %ebx,%eax */
            break;
```

这个JIT看起来通过使用DynASM受益很多，但这也有额外的影响。如构建时对Lua的依赖，这对于LInux来说是无法接受的。如果预处理后的DynASM文件提交到Linux的git仓库中，将可以避免对Lua的依赖，除非JIT被修改了，但这也许还是超过了Linux的构建系统的标准。

关于我们的BF JIT有些事情需要解释下，因为相比之前的例子使用了DynASM更多的特性。首先，你会注意到我们使用了一个`.define`指令为`rbx`寄存器起了一个别名。这点让我们可以先指定寄存器的分配，然后再通过符号来使用相应的寄存器。这里需要小心一点: 使用`PTR`和`rbx`的代码掩盖了他们是同一个寄存器的事实！在我使用的JIT中至少遇到了一次这样棘手的bug。

其次，我使用`.macro`定义一个DynASM的宏，一个宏代表DynASM中的一行或多行，使用这个宏的地方会被相应的代码替换。

这里使用的最后一个新特性是`pclabels`，DynASM支持三种不同的标记可以用来作为分支目标。pclabel最灵活，我们可以在运行时修改它。每个pclabel用一个无符号整数标记，用来定义标记和跳转到这里。每个label必须在[0, maxpc)范围内，但是我们可以调用`dasm_groupc()`来增大maxpc。DynASM将pclabels存储在动态数组中，我们不必担心增加太频繁，因为它的大小是以指数方式扩充的。DynASM中的`pclables`通过`=>labelnum`的方式来定义和引用，labelnum可以是任意的C语言表达式。

# 总结

我本希望再提供一个示例: ICFP 2016的JIT，它描述了一个叫做`Universal Machine`的虚拟机规范，是由程序员虚构的称为“The Cult of the Bound Variable”的社会使用。这个问题引起了我对虚拟机方面的兴趣，是一个非常有趣的问题，我非常希望有一天能为它编写一个JIT。

不幸的是，我在这篇文章上面花费了太多的时间，并且遇到了一些障碍。这也将是一个非常复杂的挑战，因为这个虚拟机允许自我修改。BF很容易，因为代码和数据是分开的，不允许执行时修改程序。如果允许自我修改的代码，你需要在有变更时重新生成JIT代码，将新的代码插入到现有的代码序列中将特别困难。确实有办法做到这一点，但这更加复杂，需要另一篇单独的博客。

所以今天我不会给你一个`Universal Machine`的JIT，你已经可以查看使用DynASM的实现。它是用于32位的x86平台上(而不是x86-64)，README里也介绍了一些额外的限制，但它可以告诉你自我修改的代码的问题和难处 。

还有更多的DynASM的特性我没有介绍。其中之一就是`typemaps`，它可以让你使用符号来计算结构体成员的实际地址(如你在寄存器中有一个结构体`timeval`的指针，你可以通过`TIMEVAL->tv->usec`来计算成员`tv_usec`的有效地址)。这让你在汇编中操作C语言的结构体更加简单。

DynASM是一个美丽的作品，但是没有太多的文档` – `你必须足智多谋，通过例子来学习。我希望这篇文章可以降低学习曲线，同时表明JIT也可以有`Hello World`这样的程序，通过少量的代码完成有趣和有用的事情。对于合适的人，他们也可以写很多有趣的东西

英文原文地址：[这里](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)