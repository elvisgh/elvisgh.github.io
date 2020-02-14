---
layout: post
title: 从硬件到语言，详解C++的内存对齐（memory alignment）
tags: c++ 数据结构
categories: 数据结构
---

**原博：**[**https://www.cnblogs.com/zhao-zongsheng/p/9099603.html**](https://www.cnblogs.com/zhao-zongsheng/p/9099603.html)

# 什么是内存对齐（memory alignment）

首先，什么是内存对齐（memory alignment）？这个是从硬件层面出现的概念。大家都知道，可执行程序是由一系列CPU指令构成的。CPU指令中有一些指令是需要访问内存的。最常见的就是“从内存读到寄存器”，以及“从寄存器写到内存”。在老的架构中（包括x86），也有一些运算的指令是可以直接以内存为操作数，那么这些指令也隐含了内存的读取。在很多CPU架构下，这些指令都要求操作的内存地址（更准确的说，操作内存的起始地址）能够被操作的内存大小整除，满足这个要求的内存访问叫做访问对齐的内存（aligned memory access），否则就是访问未对齐的内存（unaligned memory access）。举例来说，ARM的LDRH指令从内存中读取2个byte到寄存器中。如果指定的内存的地址是0x2587c20，因为0x2587c20这个数能够被2整除，所以这2个byte是对齐的。而如果指定的内存的地址是0x2587c33，因为不能被2整除，所以是未对齐的。

那如果访问未对齐的内存会出现什么结果呢？这个要看CPU。

- 有些CPU架构可以访问未对齐的内存，但是会有性能上的影响。典型的就是x86架构CPU
- 有些CPU会抛出异常
- 还有些CPU不会抛出任何异常，会静默地访问错误的地址
- 近几年也有些CPU的一部分指令可以正常访问未对齐的内存，同时不会有性能影响

因为每个CPU对未对齐内存的访问的处理方式都不一样，所以访问未对齐的内存是要尽量避免的。所以就出现了C/C++的内存对齐机制。

# C++的内存对齐机制

在C++中每个类型都有两个属性，一个是大小（size），还有一个就是对齐要求（alignment requirement），或称之为对齐量（alignment）。C++标准并没有规定每个类型的对齐量，但是一般都会有这样的规律。

1. 所有基础类型的对齐量等于这个类型的大小。
2. struct, class, union类型的对齐量等于他的非静态成员变量中最大的对齐量。

另外，标准规定所有的对齐量必须是2的幂。

编译器在给一个变量分配内存时，都要算出并满足这个类型的对齐要求。struct和class类型的非静态成员变量的字节数偏移（offset）也要满足各自类型的对齐要求。

举例来说，

```
class MyObject
{
    char c;
    int i;
    short s;
};
```

c是char类型，对齐要求是1，i是int类型，对齐要求是4，s是short类型，对齐要求是2。那么MyObject取最大的，也就是4作为他的对齐要求。如果在某个函数中声明了MyObject类型的变量，那么分配给这个变量的内存的起始地址是能够被4整除的。

我们再看MyObject的成员变量。c是MyObject的第一个成员变量，所以他的字节数偏移是0，也就是说变量c占据MyObject的第一个byte。i的对齐要求是4，所以字节数偏移必须是4的倍数，又因为变量i必须在变量c的后面，于是i的字节数偏移就是4，也就是说变量i占据MyObject的第5到第8个byte，而第2到第4个byte则是空白填充（padding）。s的对齐要求是2，又因为s必须在i的后面，所以s的字节数偏移是8，也就是说，变量s占据MyObject的第9个和第10个byte。另外，因为struct、class、union类型的数组的每个元素都要内存对齐，所以一般来说struct、class、union的大小都是这个类型的对齐量的整数倍，所以MyObject的大小是12，也就是说，变量s后面会有2个byte的空白填充。

因为C++中所有内存访问都是通过变量的读写来访问的，这个机制确保了所有变量都满足了内存对齐，也就确保了程序中所有内存访问都是对齐的。

当然，C++不会阻止我们去访问未对齐的内存。例如，以下的代码就很可能会访问未对齐的内存：

```
char buf[10];
int* ptr = (int*)(buf + 1);
++*ptr;
```

这类代码是我们在实际工作中也是能遇到的。事实上这种写法是比较危险的，因为他很可能会去访问未对齐的内存。这也是为什么写c++大家都不推荐用c风格的类型转换写法，而是要用static_cast, dynamic_cast, const_cast与reinterpret_cast。这样的话，上面的代码就必须要使用reinterpret_cast，大家都知道reinterpret_cast是很危险的，也许就会想办法避免这样的逻辑。

# 常见CPU的未对齐内存访问

根据Intel最新的Intel 64及IA-32架构说明书，Intel 64及IA-32架构都支持未对齐内存的访问，但是会有性能上的额外开销（详见http://www.intel.com/products/processor/manuals）。但是实际上最近的Core系列CPU已经可以无额外开销访问未对齐的内存。

而手机上最常见的ARMv8架构，如果是普通的、不做多核同步的未对齐的内存访问，那么CPU可能会产生对齐错误（alignment fault）或者执行未对齐内存操作。换句话说，到底会报错还是正常执行，是要看具体CPU的实现的。即使是执行正常操作，也会有一些限制。例如，不能保证读写的原子性（操作一个byte的除外），很可能产生额外的开销等（详见https://developer.arm.com/docs/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile）。ARMv8中的Cortex-A系列是手机上常见的CPU家族，他们就可以正常处理未对齐内存访问，但是一般会有额外的开销（详见http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka15414.html）。

我们也可以写一个简单的程序测试一下自己的CPU对未对齐内存访问的支持，以下是代码：

```
#include <iostream>
#include <chrono>

using namespace std;
using namespace std::chrono;

milliseconds test_duration(volatile int * ptr)  // 使用volatile指针防止编译器的优化
{
    auto start = steady_clock::now();
    for (unsigned i = 0; i < 100'000'000; ++i)
    {
        ++(*ptr);
    }
    auto end = steady_clock::now();
    return duration_cast<milliseconds>(end - start);
}

int main()
{
    int raw[2] = {0, 0};
    {
        int* ptr = raw;
        cout << "address of aligned pointer: " << (void*)ptr << endl;
        cout << "aligned access: " << test_duration(ptr).count() << "ms" << endl;
        *ptr = 0;
    }
    {
        int* ptr = (int*)(((char*)raw) + 1);
        cout << "address of unaligned pointer: " << (void*)ptr << endl;
        cout << "unaligned access: " << test_duration(ptr).count() << "ms" << endl;
        *ptr = 0;
    }
    cin.get();
    return 0;
}
```

我测试使用的电脑的CPU是Intel Core i7 2630QM，是intel 2代酷睿CPU，测试结果为：

```
address of aligned pointer: 000000668DEFFA78
aligned access: 282ms
address of unaligned pointer: 000000668DEFFA79
unaligned access: 285ms
```

可以看出对齐与未对齐的内存访问没有性能上的差别。

# 在C++中修改对齐要求

一般情况下，我们不需要自定义对齐要求，但也会有很特殊的情况下需要做调整。C++中，我们可以使用alignas关键字修改一个类型、或者一个变量的对齐要求。例如：

```
class MyObject
{
    char c;
    alignas(8) int i;
    short s;
};
```

这样的话，变量i的对齐要求由原本的4变成了8，结果就是，i的字节数偏移由4变成了8，s的字节数偏移由8变成了12，MyObject的对齐要求也变成了8，大小变成了16。

我们也可以对MyObject的定义使用alignas：

```
class alignas(16) MyObject
{
    char c;
    int i;
    short s;
};
```

还可以在alignas里面写某个类型。也可以使用多个alignas，结果就是使用最大的对齐要求。例如以下MyObject的对齐要求就是16：

```
class alignas(int) alignas(16) MyObject
{
    char c;
    int i;
    short s;
};
```

alignas有一个限制，那就是不能用alignas改小对齐要求。例如以下的代码会报错：

```
alignas(1) int i;
```

另外，C++中，有一个特殊的类型：max_align_t，所有不大于他的对齐量叫做基础对齐量（fundamental alignment），比这个对齐量大的叫做扩展对齐量（extended alignment ）。C++标准规定，所有平台必须要支持基础对齐量，而对于扩展对齐量的支持要看各个平台。一般来说max_align_t的对齐量等于long double的对齐量。

C++关于内存对齐的支持还有很多功能，例如查询对齐量的alignof关键字，可以创建任意大小任意对齐要求的类型的aligned_storage模板，还有方便模板编程的alignment_of等等，在此就不细述了。