---
layout: post
title: 子类的构造与析构原理–深入阐述虚析构函数的重要性
tags: c++ 数据结构
categories: 数据结构
---

*转载地址：https://blog.csdn.net/bobbypollo/article/details/79888526*


通过反汇编（objdump -Dsx）分析得知，编译器在编译子类时，会自动把调用基类构造（析构）函数的代码嵌入到子类的构造函（析构）函数体中，也就是说，子类的构造（或析构）函数会自动调用基类的构造（或析够）函数。

当子类的构造函数中没有显式调用基类构造函数时，会默认调用基类的无参构造函数，否则会调用基类的指定参数的某构造函数。

从上述原理来看，子类指针指向子类对象的情况下，delete子类指针必然会调用到基类的析构函数。如下：
~~~c++
class Base
{
public:
Base(){};
~Base(){};
};
class Sub : public Base
{
public:
Sub(){};
~Sub(){};
};
Sub* p = new Sub;
delete p; //此处由于p的类型是Sub，因此执行的是Sub::~Sub()，因此必然也会执行到Base的析构。

特殊情况：基类指针指向子类对象时，delete基类指针会导致子类的析构函数不被调用。如下，
Base* p = new Sub;
delete p; //此处由于p的类型是Base，因此执行Base::~Base()，而Base的析构函数中是不会调用Sub的析构的，因此产生了内存泄漏（子类没有被析构）。
~~~

如何解决上述问题？我们想到了虚函数。应利用虚函数的调用原理来解决此问题。
我们知道如果某基类中含有虚函数，那么编译器将为此类生成一个虚表，虚表中的每个元素对应的是每个虚函数的地址。当子类继承基类后，会继承这个虚表，并在基类虚表基础上扩展更新，最终生成子类自己的虚表，例如，如果子类重载了基类的某个虚函数，那么子类的虚表中对应的该函数指针会被重置成子类重新实现的该函数（不管是不是virtual）的地址。
特定对象的内存中，有一个指针__vptr专门指向了该类的虚表地址（也就是说，同一个类的不同对象，实际上共用同一张虚表）。

因此，我们将基类Base的析构函数声明为virtual，如下：
~~~c++
class Base
{
public:
Base(){};
virtual ~Base(){};
};
class Sub : public Base
{
public:
Sub(){};
~Sub(){};
};
main()
{
Base* p = new Sub;
delete p;
}
~~~
解析：虽然p是Base类型，但它指向了Sub类型的对象内存，而Sub的对象内存中有一个指向Sub虚表的指针，Sub的虚表内含有函数Sub::~Sub()的指针。因此 delete p 实际上执行的是Sub虚表内的函数指针Sub::~Sub()，而Sub::~Sub()中自动调用了Base::~Base()，因此解决了“指向子类对象的基类指针在销毁时的内存泄漏问题“。

这也就完美解释了为何基类的析构函数一定要声明为 virtual 的原因。