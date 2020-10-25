---
title: C++左值右值引用
date: 2013-04-14 22:13:21
tags:
- C++
- 编译
categories:
- C++
---

C++ 11中引入了右值引用和移动语义，可以避免无谓的复制，提高程序性能。

<!-- more -->

## 左值、右值
C++中所有的值都必然属于左值、右值二者之一。左值是指表达式结束后依然存在的持久化对象，右值是指表达式结束时就不再存在的临时对象。所有的具名变量或者对象都是左值，而右值不具名。很难得到左值和右值的真正定义，但是有一个可以区分左值和右值的便捷方法:看能不能对表达式取地址，如果能，则为左值，否则为右值。

## 左值引用、右值引用
左值引用就相当于给变量取了一个别名;右值引用则关联到右值，右值被存储到了特定的位置，虽然右值无法无法获取地址，但是右值引用可以获取地址，该地址表示临时对象的存错位置。

传统的c++引用被称为左值引用，用法如下：
```c++
int i = 10;
int & ii = I;
```

C++11中新增了右值引用:
```c++
int && iii = 10;
```

现在来看一下左值引用的汇编代码：
```c++
int i = 1;
int & ii = i;
```
```asm
0x080483f3  movl   $0x1,-0x10(%ebp)
0x080483fa  lea    -0x10(%ebp),%eax
0x080483fd  mov    %eax,-0x8(%ebp)
```
第一句是将1赋值给i，第二句将i的地址放入eax中，第三句将eax中的值传给ii。可见引用就是从一个变量处取得变量的地址，然后赋值给引用变量。

然后再看一下右值引用的汇编代码：
```c++
int && iii = 10;
```
```asm
0x08048400  mov    $0xa,%eax
0x08048405  mov    %eax,-0xc(%ebp)
0x08048408  lea    -0xc(%ebp),%eax
0x0804840b  mov    %eax,-0x4(%ebp)
```
第一句将10赋值给eax，第二句将eax放入-0xc(%ebp)处，前面说到“临时变量会引用关联到右值时，右值被存储到特定位置”，在这段程序中，-0xc(%ebp)便是该临时变量的地址，后两句通过eax将该地址存到iii处。

通过上述代码，我们还可以发现，在上述的程序中-0x4(%ebp)存放着右值引用iii，-0x8(%ebp)存放着左值引用，-0xc(%ebp)存放着10，而-0x10(%ebp)存放着1，左值引用和右值引用同int一样是四个字节（因为都是地址）

同时，我们可以深入理解下临时变量，在本程序中，有名字的1（名字为i）和没有名字的10（临时变量）的值实际是按同一方式处理的，也就是说，临时变量根本上来说就是一个没有名字的变量而已。它的生命周期和函数栈帧是一致的。也可以说临时变量和它的引用具有相同的生命周期。

## 示例

先准备一个示例：

```c++
#include <stdio.h>

static int g_ID = 0;

class ABC
{
public:
	int a;
	int b;

	ABC()
	{
		a = g_ID++;
		printf("New ABC:a=%d\n"， a);
	}

	ABC(const ABC& abc)
	{
		a = g_ID++;
		printf("New ABC(const ABC& abc):a=%d\n"， a);
	}

	~ABC()
	{
		printf("Delete ABC:a=%d\n"， a);
	}
};

ABC GetABC()
{
	return ABC();
}

int main(int argc， char const *argv[])
{
	ABC abc = GetABC();
	printf("-----:a=%d\n"， abc.a);
    return 0;
}

```

我们在编译的时候加上`-fno-elide-constructors`选项来关闭返回值的优化。编译命令如下：
```bash
gcc main.cpp -o main.out -lstdc++ -fno-elide-constructors
```

然后运行可执行程序，输出结果如下：
```bash
New ABC:a=0
New ABC(const ABC& abc):a=1
Delete ABC:a=0
New ABC(const ABC& abc):a=2
Delete ABC:a=1
-----:a=2
Delete ABC:a=2
```

注意输出日志，我们会发现仅一行代码`ABC abc = GetABC();`就创建了三个对象出来。如果未开启任何优化，那么这对于我们的程序来讲性能存在很大的问题。现在我们来分析一下产生这个的原因。

首先分析函数：
```c++
ABC GetABC()
{
	return ABC();
}
```
我们可以发现`return ABC();`中的`ABC()`创建了一个临时对象，然后`return`之后临时对象通过拷贝构造函数`ABC(const ABC& abc)`又创建了一个新的临时对象。这个临时对象用于`ABC abc = GetABC();`这行语句，用于拷贝赋值给对象`abc`。这也就解释了为何会有以下的一个输出结果：
```bash
New ABC:a=0 # return ABC()语句中创建对象
New ABC(const ABC& abc):a=1 # return拷贝赋值给ABC GetABC()中的临时对象
Delete ABC:a=0 # return ABC()语句中创建对象离开了函数作用域被销毁
New ABC(const ABC& abc):a=2 # ABC abc = GetABC();临时对象拷贝构造给了abc
Delete ABC:a=1 # ABC GetABC()中的临时对象离开了作用域自动销毁
-----:a=2 # print语句
Delete ABC:a=2 # 离开作用域被销毁
```

如果我们使用右值引用会是怎样一种情况呢？修改代码如下所示：
```c++
#include <stdio.h>

static int g_ID = 0;

class ABC
{
public:
	int a;
	int b;

	ABC()
	{
		a = g_ID++;
		printf("New ABC:a=%d\n", a);
	}

	ABC(const ABC& abc)
	{
		a = g_ID++;
		printf("New ABC(const ABC& abc):a=%d\n", a);
	}

	~ABC()
	{
		printf("Delete ABC:a=%d\n", a);
	}
};

ABC GetABC()
{
	return ABC();
}

int main(int argc, char const *argv[])
{
	ABC&& abc = GetABC();
	printf("-----:a=%d\n", abc.a);
    return 0;
}

```

然后我们看看输出结果：
```bash
New ABC:a=0
New ABC(const ABC& abc):a=1
Delete ABC:a=0
-----:a=1
Delete ABC:a=1
```

我们会发现构造的临时对象少了一个，这个是因为`ABC&& abc = GetABC();`中的`abc`绑定了一个临时对象。

## 其它示例

典型的左值:
```c++
int i = 1;   // i 是左值
int *p = &i; // i 的地址是可辨认的
i = 2; // i 在内存中的值可以改变

class A;
A a; // 用户自定义类实例化的对象也是左值
```

典型的右值:
```c++
int i = 2; // 2 是右值
int x = i + 2; // (i + 2) 是右值
int *p = &(i + 2);//错误，不能对右值取地址
i + 2 = 4; // 错误，不能对右值赋值

class A;
A a = A(); // A() 是右值

int sum(int a, int b) {return a + b;}
int i = sum(1, 2) // sum(1, 2) 是右值
```

左值引用:
```c++
int i;
int &r = i;
int &r = 5; // 错误，不能左值引用绑定右值

// const 引用是例外
// 可以认为指向一个临时左值的引用，其值为5
const int &r = 5;
```

对于函数：
```c++
int square(int& a) {return a * a;}

int i = 2;
square(i); // 正确
square(2); // 报错，左值引用绑定右值

// 变通一下
int square(const int& a) {
  return a * a;
} 
// 可以调用 square(2)
```

左值可以创造右值, 右值也可创造左值:
```c++
int i = 2;
int x = i + 2; // i + 2 是右值

int v[3];
// 指针解引用可以把右值转化为左值
*(v + 1) = 2;
```

一些注意事项
```c++
// 1. 函数也可以返回左值
int myglobal ;
int& foo() {return myglobal;}
foo() = 50;

// 一个常见的例子，[]操作符几乎总是返回左值
array[3] = 50; 

// 2. 左值并不总是可修改的
// i是左值, 但不可修改
const int i = 1;  

// 3. 右值有时是可修改的
class dog;
// bark() 可能会修改 dog() 对象
dog().bark();
```

## 右值引用和移动
右值引用的两大用途：
- 移动语意
- 完美转发

### 移动语意(std::move)
可以将左值转化为右值引用:
```c++
int a = 1; // 左值
int &b = a; // 左值引用

// 移动语意: 转换左值为右值引用
int &&c = std::move(a); 

void printInt(int& i) 
{
	printf("lval ref:%d\n", i);
}

void printInt(int&& i) 
{
	printf("rval ref:%d\n", i);
}

int main(int argc, char const *argv[])
{
	int i = 1;
  
	// 调用 printInt(int&), i是左值
	printInt(i);
	
	// 调用 printInt(int&&), 6是右值
	printInt(6);
	
	// 调用 printInt(int&&)，移动语意
	printInt(std::move(i));   

    return 0;
}
```

由于编译器调用时无法区分:
- printInt(int) 与 printInt(int&)
- printInt(int) 与 printInt(int&&)
如果再定义 printInt(int) 函数，会报错

为什么需要移动语意:
```c++
#include <stdio.h>
#include <utility>

class FloatVector 
{

private:
    int size;
    float* array;
public:
    
    // 复制构造函数
    FloatVector(const FloatVector& rhs) 
    {
        printf("Copy Construct\n");
        size = rhs.size;
        array = new float[size];
        for (int i=0; i<size; i++) 
        {
            array[i] = rhs.array[i];
        }
    }

    FloatVector(int n) 
    {
        printf("Construct(n)\n");
        size = n;
        array = new float[n];
    }
};

void Foo(FloatVector v) 
{
    /* Do something */
    
}

// 假设有一个函数，返回值是一个FloatVector
FloatVector CreateFloatVector()
{
    return FloatVector(1024);
}

int main(int argc, char const *argv[])
{
	// Case 1:
    FloatVector reusable = CreateFloatVector();

    // 这里会调用 FloatVector 的复制构造函数
    // 如果我们不希望 foo 随意修改 reusable
    // 这样做是 ok 的
    Foo(reusable); 
    /* Do something with reusable */

    // Case 2:
    // CreateFloatVector 会返回一个临时的右值
    // 传参过程中会调用拷贝构造函数
    // 多余地被复制一次
    // 虽然大部分情况下会被编译器优化掉
    Foo(CreateFloatVector());

    return 0;
}

```
输出如下：
```bash
Construct(n)
Copy Construct
Copy Construct
Copy Construct
Construct(n)
Copy Construct
Copy Construct
```
我们发现，3行代码调用了7次构造函数。显然跟我们的预期不符。

解决方法, 添加一个移动构造函数:
```c++
#include <stdio.h>
#include <utility>

class FloatVector 
{

private:
    int size;
    float* array;
public:

    // 复制构造函数
    FloatVector(const FloatVector& rhs) 
    {
        printf("Copy Construct\n");
        size = rhs.size;
        array = new float[size];
        for (int i=0; i<size; i++) 
        {
            array[i] = rhs.array[i];
        }
    }

    FloatVector(int n) 
    {
        printf("Construct(n)\n");
        size = n;
        array = new float[n];
    }

    // 移动构造函数
    FloatVector(FloatVector&& rhs) 
    {  
        printf("Move Constructor\n");
        size = rhs.size; 
        array = rhs.array;
        rhs.size = 0;
        rhs.array = nullptr;
    }
};

void Foo(FloatVector v) 
{
    /* Do something */
    
}

// 假设有一个函数，返回值是一个FloatVector
FloatVector CreateFloatVector()
{
    return FloatVector(1024);
}

int main(int argc, char const *argv[])
{
	// Case 1:
    FloatVector reusable = CreateFloatVector();

    // 这里会调用 FloatVector 的复制构造函数
    // 如果我们不希望 foo 随意修改 reusable
    // 这样做是 ok 的
    Foo(reusable); 
    /* Do something with reusable */

    // Case 2:
    // CreateFloatVector 会返回一个临时的右值
    // 传参过程中会调用拷贝构造函数
    // 多余地被复制一次
    // 虽然大部分情况下会被编译器优化掉
    Foo(CreateFloatVector());

    return 0;
}

```

那么，Foo(CreateFloatVector())就不会调用拷贝构造函数，而会调用移动构造函数 (当然更大的可能性是编译器直接把这一步优化掉)。输出如下所示：
```bash
Construct(n)
Move Constructor
Move Constructor
Copy Construct
Construct(n)
Move Constructor
Move Constructor
```

### 完美转发
完美转发是指：
- 只有在需要的时候，才调用复制构造函数
- 左值被转发为左值，右值被转发为右值

参数转发：
```c++
#include <stdio.h>
#include <utility>

class FloatVector 
{

private:
    int size;
    float* array;
public:

    // 复制构造函数
    FloatVector(const FloatVector& rhs) 
    {
        printf("Copy Construct\n");
        size = rhs.size;
        array = new float[size];
        for (int i=0; i<size; i++) 
        {
            array[i] = rhs.array[i];
        }
    }

    FloatVector(int n) 
    {
        printf("Construct(n)\n");
        size = n;
        array = new float[n];
    }

    // 移动构造函数
    FloatVector(FloatVector&& rhs) 
    {  
        printf("Move Constructor\n");
        size = rhs.size; 
        array = rhs.array;
        rhs.size = 0;
        rhs.array = nullptr;
    }
};

// 假设有一个函数，返回值是一个FloatVector
FloatVector CreateFloatVector()
{
    return FloatVector(1024);
}

void Foo(FloatVector v) 
{
    /* Do something */
}

// 参数转发
template<typename T>
void Relay(T arg) 
{
    Foo(arg);
}

int main(int argc, char const *argv[])
{
    FloatVector reusable = CreateFloatVector();
    printf("---\n");
    Relay(reusable);
    printf("+++\n");
    Relay(CreateFloatVector());
    return 0;
}
```
输出如下：
```bash
Construct(n)
Move Constructor
Move Constructor
---
Copy Construct
Copy Construct
+++
Construct(n)
Move Constructor
Move Constructor
Copy Construct
```

这个实现有个问题，假如定义了两个版本的`Foo`。
```c++
void Foo(FloatVector& v) 
{

}

void Foo(FloatVector&& v) 
{

}
```
永远只有`Foo(FloatVector& v)`会被调用(右值引用FloatVector&& v是个左值，这里的v是有名字的)，所以我们需要改写上文的`Relay`函数，借助 std::forward。代码如下所示：
```c++
#include <stdio.h>
#include <utility>

class FloatVector 
{

private:
    int size;
    float* array;
public:

    // 复制构造函数
    FloatVector(const FloatVector& rhs) 
    {
        printf("Copy Construct\n");
        size = rhs.size;
        array = new float[size];
        for (int i=0; i<size; i++) 
        {
            array[i] = rhs.array[i];
        }
    }

    FloatVector(int n) 
    {
        printf("Construct(n)\n");
        size = n;
        array = new float[n];
    }

    // 移动构造函数
    FloatVector(FloatVector&& rhs) 
    {  
        printf("Move Constructor\n");
        size = rhs.size; 
        array = rhs.array;
        rhs.size = 0;
        rhs.array = nullptr;
    }
};

void Foo(FloatVector& v) 
{
    /* Do something */
    printf("Foo(FloatVector& v)\n");
}

void Foo(FloatVector&& v) 
{
    /* Do something */
    printf("Foo(FloatVector&& v)\n");
}

// 假设有一个函数，返回值是一个FloatVector
FloatVector CreateFloatVector()
{
    return FloatVector(1024);
}

// 参数转发
template<typename T>
void Relay(T arg) 
{
    Foo(std::forward<T>(arg));
}

int main(int argc, char const *argv[])
{
    FloatVector reusable = CreateFloatVector();
    printf("---\n");
    Relay(reusable);
    printf("+++\n");
    Relay(CreateFloatVector());
    return 0;
}
```

输出结果如下所示：
```bash
Construct(n)
Move Constructor
Move Constructor
---
Copy Construct
Foo(FloatVector&& v)
+++
Construct(n)
Move Constructor
Move Constructor
Foo(FloatVector&& v)
```

于是就有
- Relay(reusable)调用Foo(FloatVector& v)
- Relay(CreateFloatVector())调用Foo(FloatVector&& v)

要解释完美转发的原理，首先引入`C++11`的引用折叠原则：
- T& & => T&
- T& && => T&
- T&& & => T&
- T&& && => T&&

所以，在`relay`函数中
- Relay(9); => T = int&& => T&& = int&& && = int&&
- Relay(x); => T = int& => T&& = int& && = int &

因此，在这种情况下，T&& 被称作`universal reference`，即满足：
- T 是模板类型
- T 是被推导出来的，适用引用折叠，即`T`是一个函数模板类型，而不是类模板类型

然后，C++11 定义了`remove_reference`，用于返回引用指向的类型:
```c++
template<class T>
struct remove_reference; 

remove_reference<int&>::type == int
remove_reference<int>::type  == int
```

于是，`std::forword`的实现如下:
```c++
template<class T>
T&& forward(typename remove_reference<T>::type& arg) 
{
  return static_cast<T&&>(arg);
}
```
等于是把右值引用(右值引用本身是个左值)转成了右值，左值保持不变。


