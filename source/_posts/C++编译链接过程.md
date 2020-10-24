---
title: C++编译链接过程
date: 2013-03-12 19:41:13
tags:
- C++
- 编译
categories:
- C++
---

C++生成可执行文件其实分为了几个阶段，每个阶段都非常重要。了解每个阶段所做的事情对我们编写C++代码时会有极大的帮助。浅尝辄止的来了解一下其中的步骤！

<!-- more -->

## 示例

先准备一个示例，用来展示整个过程。程序结构如下所示：

```bash
├── main.cpp
└── math
    ├── Math.cpp
    └── Math.h
```

三个文件的代码分别如下：

其中`main.cpp`如下所示：
```c++
// main.cpp
#include "math/Math.h"

#include <stdio.h>

int main(int argc, char const *argv[])
{
    printf("a=%f, b=%f, a+b=%f\n", 1.0f, 2.0f, add(1.0f, 2.0f));
    printf("a=%d, b=%d, a-b=%d\n", 5, 4, sub(5, 4));

    return 0;
}
```

`Math.h`头文件如下所示：
```c++
// math/Math.h

#pragma once

template<typename T>
T add(T a, T b)
{
    return a + b;
}

int sub(int a, int b);

```

`Math.cpp`实现如下所示：
```c++
// math/Math.cpp
#include "math/Math.h"

int sub(int a, int b)
{
    return a - b;
}
```

## 预处理

预处理用于将所有的#include头文件以及宏定义替换成其真正的内容,预处理之后得到的仍然是文本文件,但文件体积会大很多。主要规则如下：
- 将所有的"#define"删除,并且展开所有的宏定义。
- 处理所有条件预编译指令,比如"#if", "#ifdef", "#elif", "#else", "#endif"。
- 处理"#include"预编译指令,将被包含的文件插入到该预编译指令的位置。注意这个过程是递归进行的,也就是说被包含的文件可能还包含其他文件。
- 删除所有的注释"//"和"/**/"。
- 添加行号和文件名标识，比如#2"a.c"2，以便于编译时编译器产生调试用的行号信息及用于编译时产生编译错误或警告时能够显示行号。
- 保留所有的#pragma编译器指令，因为编译器需要使用它们。
- 
通过如下命令对main.cpp进行预处理：
```bash
gcc -E -I ./math main.cpp -o main.i
```
或者`cpp`命令
```bash
cpp -I ./math main.cpp -o main.i
```

## 编译

编译过程就是把预处理完的文件进行一系列词法分析、语法分析、语义分析、代码优化及优化后生成相应的汇编代码文件。
通过如下命令对`main.i`进行编译
```bash
gcc -S -I ./math main.cpp -o main.s
```
上述命令中-S让编译器在编译之后停止,不进行后续过程。编译过程完成后将生成程序的汇编代码main.s。内容如下:
```asm
	.file	"main.cpp"
	.text
	.section	.rodata
.LC4:
	.string	"a=%f, b=%f, a+b=%f\n"
.LC5:
	.string	"a=%d, b=%d, a-b=%d\n"
	.text
	.globl	main
	.type	main, @function
main:
.LFB1:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$16, %rsp
	movl	%edi, -4(%rbp)
	movq	%rsi, -16(%rbp)
	movss	.LC0(%rip), %xmm1
	movss	.LC1(%rip), %xmm0
	call	_Z3addIfET_S0_S0_
	cvtss2sd	%xmm0, %xmm2
	movsd	.LC2(%rip), %xmm1
	movsd	.LC3(%rip), %xmm0
	leaq	.LC4(%rip), %rdi
	movl	$3, %eax
	call	printf@PLT
	movl	$4, %esi
	movl	$5, %edi
	call	_Z3subii@PLT
	movl	%eax, %ecx
	movl	$4, %edx
	movl	$5, %esi
	leaq	.LC5(%rip), %rdi
	movl	$0, %eax
	call	printf@PLT
	movl	$0, %eax
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE1:
	.size	main, .-main
	.section	.text._Z3addIfET_S0_S0_,"axG",@progbits,_Z3addIfET_S0_S0_,comdat
	.weak	_Z3addIfET_S0_S0_
	.type	_Z3addIfET_S0_S0_, @function
_Z3addIfET_S0_S0_:
.LFB2:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movss	%xmm0, -4(%rbp)
	movss	%xmm1, -8(%rbp)
	movss	-4(%rbp), %xmm0
	addss	-8(%rbp), %xmm0
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE2:
	.size	_Z3addIfET_S0_S0_, .-_Z3addIfET_S0_S0_
	.section	.rodata
	.align 4
.LC0:
	.long	1073741824
	.align 4
.LC1:
	.long	1065353216
	.align 8
.LC2:
	.long	0
	.long	1073741824
	.align 8
.LC3:
	.long	0
	.long	1072693248
	.ident	"GCC: (Ubuntu 8.3.0-6ubuntu1~18.10.1) 8.3.0"
	.section	.note.GNU-stack,"",@progbits
```

## 汇编(Assemble)

汇编过程将上一步的汇编代码转换成机器码,这一步产生的文件叫做目标文件,是二进制格式。gcc汇编过程通过as命令完成:

```bash
as main.s -o main.o
```
或者
```bash
gcc -c main.s -o main.o
```

这一步会为每一个源文件产生一个目标文件。

## 链接

链接过程将多个目标文以及所需的库文件(.so等)链接成最终的可执行文件(executable file)。
命令如下：
```bash
gcc Math.o main.o -o main.out
```
然后运行main.out程序，输出结果如下所示：
```
a=1.000000, b=2.000000, a+b=3.000000
a=5, b=4, a-b=1
```

## 静态库

可以将上述里面的Math.o目标文件打包成一个静态库。代码如下所示：
```bash
ar cr libmath.a Math.o
```
然后在链接的时候指定该静态链接库路径：
```bash
gcc main.o -L. -lmath -o main.out
```
注意，`-L`紧跟的是路径'.';`-l`紧跟的是静态库名词`math`，-l会自动添加lib前缀以及.a后缀。

## 动态链接库

可以将上述代码里面的`Math`生成一个动态链接库，然后链接到可执行文件。生成动态链接库命令如下：
```bash
gcc -fPIC -shared -I ./ ./math/Math.cpp -o libmath.so
```
链接到可执行文件：
```bash
gcc main.cpp -L. -lmath -o main.out
```
然后执行`main.out`可执行文件，但是在执行之前需要指定一个动态链接库的路径。不然可执行文件会报错提示找不到动态库的位置。
```bash
# 未设置环境变量之前
./main.out 
./main.out: error while loading shared libraries: libmath.so: cannot open shared object file: No such file or directory
# 设置环境变量之后
export LD_LIBRARY_PATH=./
./main.out
a=1.000000, b=2.000000, a+b=3.000000
a=5, b=4, a-b=1
```
然后我们可以更改一下Math.cpp的逻辑，重新生成一个动态库，然后运行一下`main.out`.
```c++
#include "math/Math.h"

int sub(int a, int b)
{
    return a - b + 5;
}
```

```bash
gcc -fPIC -shared -I ./ ./math/Math.cpp -o libmath.so
./main.out 
a=1.000000, b=2.000000, a+b=3.000000
a=5, b=4, a-b=6
```
注意，我们并没有修改`main.cpp`文件，但是运行结果发生了变化。

### 动态加载动态库

修改`Math.h`如下：
```c++
#pragma once

template<typename T>
T add(T a, T b)
{
    return a + b;
}

extern "C"
{
    int sub(int a, int b);
}
```

然后生存动态链接库：
```c++
gcc -fPIC -shared -I ./ ./math/Math.cpp -o libmath.so
```

修改`main.cpp`如下：
```c++
#include <stdio.h>
#include <dlfcn.h> 

int main(int argc, char const *argv[])
{
	void* handle = dlopen("./libmath.so", RTLD_LAZY);

	if (!handle) 
    {
		printf("%s\n", dlerror());
		return 1;
	}
    
    // int sub(int a, int b);
    int (*sub)(int, int);
	sub = (int(*)(int, int))dlsym(handle, "sub");
    
    char* error = NULL;
	if ((error = dlerror()) != NULL)  
    {  
		printf("%s\n", error);
        return 1;
	}
    
	printf ("a = %d, b=%d, a-b=%d\n", 5, 4, (*sub)(5, 4));  
	dlclose(handle);  

    return 0;
}

```

然后生成可执行文件：
```
gcc main.cpp -o main.out -ldl
```

最后运行`main.out`,结果如下所示：
```
a = 5, b=4, a-b=6
```