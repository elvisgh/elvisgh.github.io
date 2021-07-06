---
layout: post
title: C/C++中sizeof与strlen异同
tags: c++ 数据结构
categories: 数据结构
---

*转载地址：[sizeof_百度百科 (baidu.com)](https://baike.baidu.com/item/sizeof/6349467?fr=aladdin) [strlen（C/C++语言函数）_百度百科 (baidu.com)](https://baike.baidu.com/item/strlen/2737?fr=aladdin)*

C/C++中计算数据或者字符串的方法有哪些？

**sizeof**

sizeof是C/C++中的一个操作符（operator），简单的说其作用就是返回一个对象或者类型所占的**内存字节数**。

The sizeof keyword gives the amount of storage, in bytes, associated with a variable or a type(including aggregate types). This keyword returns a value of type size_t.

sizeof有两种语法形式，如下：

```c++
sizeof (type_name);//sizeof(类型);
sizeof object;//sizeof对象;
头文件：stddef.h
```

sizeof的计算发生在编译时刻，所以它可以被当作[常量](https://baike.baidu.com/item/常量)[表达式](https://baike.baidu.com/item/表达式)使用，如：

```c++
char ary[sizeof(int)*10];//ok
```

**strlen**

strlen是C/C++中的一个函数，它从内存的某个位置（可以是字符串开头，中间某个位置，甚至是某个不确定的内存区域）开始扫描，直到碰到第一个字符串结束符'\0'为止，然后返回**计数器值**(长度不包含'\0')。

```c++
extern unsigned int strlen(char *s);//strlen (字符数组名)
头文件：string.h
```

**sizeof/strlen区别**

```c++
char aa[10];cout<<strlen(aa)<<endl; //结果是不定的，因为未初始化，'\0'在内存中的位置不确定
char aa[10]={'\0'}; cout<<strlen(aa)<<endl; //结果为0
char aa[10]="jun"; cout<<strlen(aa)<<endl; //结果为3
而sizeof()函数返回的是变量声明后所占的内存数，不是实际长度。
sizeof(aa) 返回10 int a[10]; sizeof(a) 返回40
    
在C++里参数传递数组永远都是传递指向数组首元素的指针，编译器不知道数组的大小，如果想在函数内知道数组的大小， 需要这样做：
进入函数后用memcpy拷贝出来，长度由另一个形参传进去
fun(unsiged char*p1,int len)　
{
unsigned char*buf=new unsigned char[len+1];
memcpy(buf,p1,len);
}

常在用到 sizeof 和 strlen 的时候，通常是计算字符串数组的长度，如果是对指针，结果则会不一样的：
char* str = "abacd";
sizeof(str) //结果 4 --->str是指向字符串常量的字符指针，sizeof 获得的是一个指针所占的空间,应该是长整型的，所以是4；
sizeof(*str) //结果 1 --->*str是第一个字符，其实就是字符串的第一位'a' 所占的内存空间，是char类型的，占了 1 位；
strlen(str)= 5 //--->若要获得这个字符串的长度，则一定要使用 strlen

看了上面的详细解释，发现两者的使用还是有区别的，从这个例子可以看得很清楚：
char str[20]="0123456789";
int a=strlen(str); //a=10; >>>> strlen 计算字符串的长度，以结束符 0x00 为字符串结束。
int b=sizeof(str); //而b=20; >>>> sizeof 计算的则是分配的数组str[20] 所占的内存空间的大小，不受里面存储的内容改变。

上面是对静态数组处理的结果，如果是对指针，结果就不一样了
char* ss = "0123456789";
sizeof(ss) 结果 4 ===》ss是指向字符串常量的字符指针，sizeof 获得的是一个指针的之所占的空间,应该是
长整型的，所以是4
sizeof(*ss) 结果 1 ===》*ss是第一个字符其实就是获得了字符串的第一位'0' 所占的内存空间，是char类
型的，占了 1 个字节
strlen(ss)= 10 >>>> 如果要获得这个字符串的长度，则一定要使用 strlen
sizeof返回对象所占用的字节大小. //正确
strlen返回字符个数. //正确

在使用sizeof时，有一个很特别的情况，就是数组名到指针蜕变，
char Array[3]={'0'};
sizeof(Array)==3;
char*p=Array;
strlen(p)==1;//sizeof(p)结果为4
在传递一个数组名到一个函数中时，它会完全退化为一个指针
大部分编译程序 在编译的时候就把sizeof计算过了 是类型或是变量的长度
这就是sizeof(x)可以用来定义数组维数的原因

char str[20]="0123456789";
int a=strlen(str);//a=10;
int b=sizeof(str);//而b=20;
char ss[]="0123456789";
sizeof(ss)结果11===》ss是数组，计算到\0位置，因此是10+1
sizeof(*ss)结果1===》*ss是第一个字符
char ss[100]="0123456789";
sizeof(ss)结果是100===》ss表示在内存中的大小100×1
strlen(ss)结果是10===》strlen是个函数内部实现是用一个循环计算到\0为止之前
int ss[100]="0123456789";
sizeof(ss)结果400===》ss表示在内存中的大小100×4
strlen(ss)错误===》strlen的参数只能是char*且必须是以'\0'结尾的
char q[]="abc";
char p[]="a\n";
sizeof(q),sizeof(p),strlen(q),strlen(p);
结果是 4 3 3 2
```