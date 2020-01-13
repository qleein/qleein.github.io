---
layout: post
title:  "Lua中的全局变量与环境"
date:   2017-03-14 00:02:00 +0800
tags: Lua
categories: Openresty
---

## 环境的概念

Lua中类型为thread，function和userata的对象都可以关联一个表，称之为环境。环境也是一个常规的table。可以和普通的table一样进行操作，存放与对象相关的各种变量。

* 关联的thread上的环境只能通过C代码中访问。
* 关联在userdata上的环境在 Lua 中没有意义。 这个东西只是为了在程序员想把一个表关联到一个 userdata 上时提供便利。
* 关联在function上的环境用来接管本函数内全局变量的访问。


## 全局变量


Lua中的全局变量存在放当前函数的环境中，Lua标准库中的函数如setmetable, string.find等注册在函数的环境中，在Lua脚本就可以直接访问。

### 操作函数的环境

Lua代码中可以访问和操作和函数关联的环境，Lua标准库为此提供了两个方法

* getfenv 获取当前函数的环境
* setfenv 设置当前对象函数关联的环境

### 查看所有的全局变量

通过getfenv(0)获取当前环境后遍历，可以查看所有的全局变量。

```
for k, v in pairs(getfenv(0)) do
    print ("k: ", k, ", v: ", type(v))
end
```

### 定义全局变量与局部变量

Lua中定义的变量默认为全局变量，保存在当前函数的环境中。
如下面的代码

```lua
local env = getfenv(1)
print("var: ", env["var"])
print("loval_var: ", env["loval_var"])

var = "Hello, global variable"
local local_var = "Hell, local variable"

print("after set")
print("var: ", env["var"])
print("loval_var: ", env["loval_var"])
```

变量var为全局变量，存放在当前的环境中，通过`getfenv().var`可以访问，而变量local_var为局部变量，`getfenv().local_var`值为`nil`

执行结果

```
$ lua test3.lua
var: 	nil
loval_var: 	nil
after set
var: 	Hello, global variable
loval_var: 	nil
```

### 嵌套函数的环境

在thread中通过load等方式创建的函数称为非嵌套函数，非嵌套函数的默认环境为此thread的环境。在函数中创建函数时，会将自己的环境设置为新创建的函数的默认环境。

访问全局变量时访问的是当前所在函数的环境，如下面的代码

```lua
local function f()
    print("env: ", getfenv())

    local function f1()
        print("env f1: ", getfenv())
    end

    local function f2()
        print("env f2: ", getfenv())
    end

    f1()
    f2()
end

f()
```

执行结果 f，f1，f2三个函数的环境是同一个table

```
env: 	table: 0x25a16b0
env f1: 	table: 0x25a16b0
env f2: 	table: 0x25a16b0

```

### 改变函数的环境

因为函数f1和f2都是在函数f中创建的，他们的环境的初始值就是函数f的环境。
下面通过setfenv改变函数的环境

```
local function f()
    print("env: ", getfenv())

    local function f1()
        print("env f1: ", getfenv())
    end

    local function f2()
        print("env f2: ", getfenv())
    end
    
    setfenv(f2, {})

    f1()
    f2()
end

f()
```

执行结果如下，getfenv本身就在环境中存放，通过`setfenv(f2, {})`将函数f2的环境设置成一个空的table，在函数f2中调用getfenv时由于getfenv是nil，导致脚本报错。

```
env: 	table: 0x17a66b0
env f1: 	table: 0x17a66b0
lua: test4.lua:9: attempt to call global 'getfenv' (a nil value)
stack traceback:
	test4.lua:9: in function 'f2'
	test4.lua:15: in function 'f'
	test4.lua:18: in main chunk
	[C]: ?

```

将函数f的环境通过闭包保存下来，就可以在函数f2中调用其中的方法。如下所示

```
local function f()
    print("env: ", getfenv())

    local function f1()
        print("env f1: ", getfenv())
    end
    
    local env = getfenv()
    local function f2()
        local e = env
        e.print("env f2: ", e.getfenv())
    end
    
    setfenv(f2, {})

    f1()
    f2()
end

f()
```

此时可以看到函数f2的环境与函数f和f1不同

```
$ lua test4.lua 
env: 	table: 0x1ebf6b0
env f1: 	table: 0x1ebf6b0
env f2: 	table: 0x1ec7480
```

### _G

Lua中的_G是一个指向全局环境(thread的环境)的全局变量。详细一点可以这样理解

1. thread有一个环境，称为全局环境
2. 全局环境中有一个变量，变量名为_G， 值为全局环境（_G._G = _G?）
3. thread中调用load(string)或者loadstring等，解析Lua代码，生成一个函数
4. 生成的函数的环境默认为thread的环境
5. 在非嵌套函数中使用_G如`print(_G)`， 打印的是全局环境，也是此函数的环境。


输出_G和_G._G，值完全相同

```
$ cat test.lua 
print(_G)
print(_G._G)
$ lua test.lua
table: 0x1d0c6b0
table: 0x1d0c6b0
```

如果通过setfenv设置了环境，新的环境是没有_G这个变量的。

```lua
local function f()

    local function f1()
        print("f1: ", _G)
    end
    
    local env = getfenv()
    local function f2()
        local e = env
        e.print("f2: ", _G)
    end
    
    setfenv(f2, {})

    f1()
    f2()
end

f()
```

执行结果

```
f1: 	table: 0xf916b0
f2: 	nil
```


## GETGLOBAL 和 lua_getglobal的迷惑

调用print输出字符串`print("Hello")`，可以通过luac查看编译后的lua虚拟机的指令

```
$ luac -l -l test.lua

main <test.lua:0,0> (4 instructions, 16 bytes at 0xd92530)
0+ params, 2 slots, 0 upvalues, 0 locals, 2 constants, 0 functions
	1	[1]	GETGLOBAL	0 -1	; print
	2	[1]	LOADK    	1 -2	; "Hello"
	3	[1]	CALL     	0 2 1
	4	[1]	RETURN   	0 1
constants (2) for 0xd92530:
	1	"print"
	2	"Hello"
locals (0) for 0xd92530:
upvalues (0) for 0xd92530:
```


这里只关注GETGLOBAL这条指令，print是全局变量，需要通过GETGLOBAL取到这个变量的值

对于GETGLOBAL，Lua虚拟机的执行如下(以lua5.1为例)，在函数luaV_execute中

```c
      case OP_GETGLOBAL: {
        TValue g;
        TValue *rb = KBx(i);
        sethvalue(L, &g, cl->env);
        lua_assert(ttisstring(rb));
        Protect(luaV_gettable(L, &g, rb, ra));
        continue;
      }
```

注意`sethvalue(L, &g, cl->env);`这句，cl代表当前的函数，cl->env指向当前函数的环境。即GETGLOBAL指令取得的是从当前函数的环境中取得变量的值的。

而lua_getglobal这个函数确是直接访问thread的全局环境，取得全局环境中key为`name`的值。

```
void lua_getglobal (lua_State *L, const char *name);
```

同样都是"getglobal"，却是两个不同的意思，直接让我迷惑了很久，差点分不清什么是全局变量了。