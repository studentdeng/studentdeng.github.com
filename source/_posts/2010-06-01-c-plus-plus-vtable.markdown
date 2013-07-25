---
layout: post
title: "C++虚继承初探"
date: 2010-06-01 20:30
comments: true
categories: [c++]
---
昨天和同学对c++虚继承这部分 产生了一些争论，发觉自己对技术越来越浮躁了。不得不痛下决心。一看c++虚继承的内部实现（很浅很浅的看看）。

以下内容来自自己实验，希望各位大哥指点。当然要想获得权威的解释，看《Inside C++ Object Model》

让我们从最简单的开始。以下测试代码。

	class Base
	{
	public:
	    Base()
	    {
	        printf("Base construct!\n");
	    }
	    //virtual void Test()=0;
	    virtual void f()
	    {
	        printf("Base\n");
	    }
	    virtual void f2()
	    {
	        printf("Base2\n");
	    }
	    virtual void f3()
	    {
	        printf("Base3\n");
	    }
	    void f4()
	    {
	        printf("Base4\n");
	    }
	};
	class Derived: public Base
	{
	public:
	    Derived()
	    {
	        printf("Derived construct!\n");
	    }
	    virtual void f()
	    {
	        printf("Derived\n");
	    }
	    virtual void f2()
	    {
	        printf("Derived2\n");
	    }
	    virtual void f3()
	    {
	        printf("Derived3\n");
	    }
	    void f4()
	    {
	        printf("Derived4\n");
	    }
	    /*virtual void Test()
	    {
	        printf("test\n");
	    }*/
	};
	int main()
	{
	    Base *p=new Base;
	    p->f();
	    p->f2();
	    p->f3();
	    p->f4();
	    /*Base *p = new Derived;*/
	    p = new Derived;
	    p->f();
	    p->f2();
	    p->f3();
	    p->f4();
	    //p->Test();
	    delete p;
	    return 0;
	}



 
以下是在我的环境下反汇编的部分代码。我的环境是vs2008 默认的Release。 

	.text:00401060 ; int __cdecl main(int argc, const char **argv, const char **envp)
	.text:00401060 _main           proc near               ; CODE XREF: __tmainCRTStartup+10Ap
	.text:00401060
	.text:00401060 argc            = dword ptr  4
	.text:00401060 argv            = dword ptr  8
	.text:00401060 envp            = dword ptr  0Ch
	.text:00401060
	.text:00401060                 push    esi
	.text:00401061                 push    edi
	.text:00401062                 push    4               ; unsigned int
	.text:00401064                 call    ??2@YAPAXI@Z_0  ; operator new(uint)
	.text:00401069                 mov     edi, ds:__imp__printf
	.text:0040106F                 mov     esi, eax
	.text:00401071                 add     esp, 4
	.text:00401074                 test    esi, esi
	.text:00401076                 jz      short loc_40108A
	.text:00401078                 push    offset aBaseConstruct ; "Base construct!\n"
	.text:0040107D                 mov     dword ptr [esi], offset ??_7Base@@6B@ ; const Base::`vftable'
	.text:00401083                 call    edi ; __imp__printf
	.text:00401085                 add     esp, 4
	.text:00401088                 jmp     short loc_40108C

 
	.text:0040107D mov dword ptr [esi], offset ??_7Base@@6B@ ; const Base::`vftable' 
是关键，根据上面分析，将指向Base类
的虚表的指针保存到了向堆中分配的空间中，也就是 esi=**base_vtbl

	.text:0040108C                 mov     eax, [esi]
	.text:0040108E                 mov     edx, [eax]   ;这里就好理解了，eax=*base_vtbl，edx=base_vtbl
	.text:00401090                 mov     ecx, esi
	.text:00401092                 call    edx          ;调用虚表中的第一个函数以下类推
	.text:00401094                 mov     eax, [esi]
	.text:00401096                 mov     edx, [eax+4]
	.text:00401099                 mov     ecx, esi
	.text:0040109B                 call    edx
	.text:0040109D                 mov     eax, [esi]
	.text:0040109F                 mov     edx, [eax+8]
	.text:004010A2                 mov     ecx, esi
	.text:004010A4                 call    edx
	.text:004010A6                 push    offset aBase4   ; "Base4\n" ;这里看出了非虚函数的优势，效率高，直接调用函数
	.text:004010AB                 call    edi ; __imp__printf


这是Base虚表内容

	.rdata:0040216C ; const Base::`vftable'
	.rdata:0040216C ??_7Base@@6B@   dd offset ?f@Base@@UAEXXZ ; DATA XREF: _main+1Do  ;这里每个标号都指向相应函数
	.rdata:0040216C                                         ; _main+62o
	.rdata:0040216C                                         ; Base::f(void)
	.rdata:00402170                 dd offset ?f2@Base@@UAEXXZ ; Base::f2(void)
	.rdata:00402174                 dd offset ?f3@Base@@UAEXXZ ; Base::f3(void)
	.rdata:00402178                 dd offset ??_R4Derived@@6B@ ; const Derived::`RTTI Complete Object Locator' ;这个不懂


Base 还是比较简单的，让我们看Derived

	.text:004010BD                 push    offset aBaseConstruct ; "Base construct!\n"
	.text:004010C2                 mov     dword ptr [esi], offset ??_7Base@@6B@ ; const Base::`vftable'
	.text:004010C8                 call    edi ; __imp__printf
	.text:004010CA                 push    offset aDerivedConstru ; "Derived construct!\n"
	.text:004010CF                 mov     dword ptr [esi], offset ??_7Derived@@6B@ ; const Derived::`vftable'
	.text:004010D5                 call    edi ; __imp__printf
	.text:004010D7                 add     esp, 8
 

可见在构造函数中和我们想象的完全一样，从基类开始，不过需要注意一点，最后esi=**Derived_vtbl
 以后的代码完全和在基类中调用函数一致。看来在VS2008中，c++的虚表其实就是数组（原来居然还以为是链表，不过似乎也有的编译器
 是用链表实现的）。这个例子的确不复杂，但是事实上却没有这么简单。看下一个稍微复杂一点的。


	class A
	{
	public:
	    A()
	    {
	        printf("A construct\n");
	    }
	    virtual void f(){printf("A_F\n");}
	};
	class B
	{
	public:
	    B()
	    {
	        printf("B construct\n");
	    }
	    virtual void f(){printf("B_F\n");}
	    virtual void g(){printf("B_G\n");}
	};
	class C: public A,public B
	{
	public:
	    C()
	    {
	        printf("C construct\n");
	    }
	    void f(){printf("C_f\n");}
	};

	int _tmain(int argc, _TCHAR* argv[])
	{
	    A *a=new A;
	    B *b=new B;
	    C *c=new C;
	    a->f();
	    b->f();
	    b->g();
	    c->f();
	    return 0;
	}
 


先不看结果，花几分钟思考一下，class C 的虚表结构是什么？

首先看代码，发现在class C中首先有一点不同，这个是之前的在class A,classB,classC中都是默认构造函数的代码

	.text:00401077                 push    8               ; unsigned int      ;以前class只放一个指针，现在2个了。
	.text:00401079                 call    ??2@YAPAXI@Z_0  ; operator new(uint)
	.text:0040107E                 add     esp, 4
	.text:00401081                 test    eax, eax
	.text:00401083                 jz      short loc_40109D
	.text:00401085                 mov     dword ptr [eax+4], offset ??_7B@@6B@ ; const B::`vftable'
	.text:0040108C                 mov     dword ptr [eax], offset ??_7C@@6BA@@@ ; const C::`vftable'{for `A'}
	.text:00401092                 mov     dword ptr [eax+4], offset ??_7C@@6BB@@@ ; const C::`vftable'{for `B'}
	.text:00401099                 mov     edi, eax
	.text:0040109B                 jmp     short loc_40109F


这个是上面代码真正的反汇编代码，对比下，就可能对上面代码为什么有一个这么冗余的代码，似乎有些感觉了。
 
	.text:004010A6                 push    offset aAConstruct ; "A construct\n"
	.text:004010AB                 mov     dword ptr [esi], offset ??_7A@@6B@ ; const A::`vftable'
	.text:004010B1                 call    edi ; __imp__printf
	.text:004010B3                 push    offset aBConstruct ; "B construct\n"
	.text:004010B8                 mov     dword ptr [esi+4], offset ??_7B@@6B@ ; const B::`vftable'
	.text:004010BF                 call    edi ; __imp__printf
	.text:004010C1                 push    offset aCConstruct ; "C construct\n"
	.text:004010C6                 mov     dword ptr [esi], offset ??_7C@@6BA@@@ ; const C::`vftable'{for `A'}
	.text:004010CC                 mov     dword ptr [esi+4], offset ??_7C@@6BB@@@ ; const C::`vftable'{for `B'}
	.text:004010D3                 call    edi ; __imp__printf
	.text:004010D5                 add     esp, 0Ch

 


下面的大部分容易理解，关键的是在class B的虚表中的f()。

	; [thunk]:public: virtual void __thiscall C::f`adjustor{4}' (void)
	?f@C@@W3AEXXZ proc near               ;这时ecx 也就是this是指向class B的
	sub     ecx, 4                        ;这里很明显将原来的指向B:f()，指向了class C的虚表的开始部分。ecx放的是this指针
	jmp     ?f@C@@UAEXXZ    ; C::f(void)  ;这里顺理成章的变成了C::f()，this也在上部改变了
	?f@C@@W3AEXXZ endp
 


这里似乎就是传说中的“形式转换程序”，这个的确减少了虚表的体积。
再看后面的代码，函数调用的时候和之前完全一致，也就是在class C中定义的f()，虽然没有被显示的声明为virtual，但vs2008已经
把他默认当成虚函数调用了。至此，和同学的争论就此结束。