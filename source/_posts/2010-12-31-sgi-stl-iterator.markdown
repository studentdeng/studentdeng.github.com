---
layout: post
title: "SGI STL 学习笔记一 Iterator"
date: 2010-12-31 21:33
comments: true
categories: [STL, algorithms]
---

之前一直希望能够看STL源代码，因为他一直存放在我的硬盘深处。但是由于复杂性，我一直再绕。而且纠结的是，我一直推荐我的学弟去研读STL。由于最近的工作需要，使我不得不一看STL的究竟。当然，STL对我来说依然是太庞大了，有相当多的相关的基础知识的缺乏导致整个过程实在是太艰难了，直到我看到了《SGI STL 源码剖析》。之后我的很多例子其实就是这本书的源代码。真的，这又是一本经典的著作。这本书贯穿了我整个STL的学习。当然，如果你之前研读过《inside c++ object model》等经典C++教材。你会发现。整个知识开始网罗了。

　　按照道理来讲，学习STL，实在是不能不从总图开始。但是STL太庞大了。这个总图只是一个残缺的部分。按照常规的思路，的确是需要从总纲学，但是，真的，我不能在这里胡扯。为了更方便自己理解。我从Iterator开始。

　　从这个图中可以看出，Algorithm 通过 Iterator 访问 Container，而我们很多面向container的操作同样是有Iterator出发的。所以，我也准备从这里入手。

![alt text](/images/stl-it-1.png)

制作一个Iterator，我们首先遇到的一个问题，就是如何找到这个Iterator 指向的类型。

1、根据参数推导，我们可以找出参数类型，但是，如果是返回值，我们则无能为力。

	#include <iostream>
	using namespace std;
	 
	template <class I, class T>
	void func_impl(I iter, T t)
	{
	    T tmp = t;
	    cout<<tmp<<endl;
	}
	template <class I>
	inline
	void func(I iter)
	{
	    func_impl(iter, *iter);
	}
	 
	int main()
	{
	    int i = 9;
	    func(&i);
	}

2、通过声明内嵌类型。我们可以找到这个Iterator的类型

	#include <iostream>
	using namespace std;
	 
	template <class T>
	struct MyIter
	{
	    typedef T value_type;
	    T* ptr;
	    MyIter(T* p = 0):ptr(p){}
	    T& operator*()const {return *ptr;}
	};
	template <class I>
	typename I::value_type
	func(I ite){return *ite;}
	 
	int main()
	{
	    MyIter<int> iter(new int(8));
	    cout<<func(iter)<<endl;
	    delete iter.ptr;
	    iter.ptr = NULL;
	}

但是，我们并没有解决问题，如果这个Iterator 指向的地方，不是一个class type。原生指针不是，所以，我们必须找到一个方式这个就是 template partial specialization。

	#include <iostream>
	using namespace std;
	 
	template <class I>
	struct iterator_traits
	{
	    typedef typename I::value_type value_type;
	};
	 
	template <class T>
	struct iterator_traits<T*>
	{
	    typedef T value_type;
	};
	template <class T>
	struct iterator_traits<const T*>
	{
	    typedef T value_type;
	};
	 
	template <class I>
	typename iterator_traits<I>::value_type
	func(I iter){return *iter;}
	 
	int main()
	{
	    int i = 50;
	    cout<<func(&i)<<endl;
	}

构造一个Iterator，我们必须有的部分。

	struct iterator {
	 
	typedef _Category iterator_category;//Iterator种类
	 
	typedef _Tp value_type; //iterator 所指对象类型
	 
	typedef _Distance difference_type; //2个Iterator之间的距离
	 
	typedef _Pointer pointer; //iterator 所指对象的地址
	 
	typedef _Reference reference; //Iterator 所指对象引用类型
	 
	};
	 
	template <class _Iterator>
	 
	struct iterator_traits {
	 
	typedef typename _Iterator::iterator_category iterator_category;
	 
	typedef typename _Iterator::value_type value_type;
	 
	typedef typename _Iterator::difference_type difference_type;
	 
	typedef typename _Iterator::pointer pointer;
	 
	typedef typename _Iterator::reference reference;
	 
	};

对于原生指针，需要适应特化版本。这里从略。

![alt text](/images/stl-it-2.png)

SGI STL 增加元素之一 __type_traits

从字面上看，这里是类型萃取。的确。这里是对 trivial default constructor 和 none trivial defaultconstructor 的区别。 以及 none-trivial assignment operator 。 non-trivial-dtor。相关的知识可以在《inside c++ object model》中找到。在面对拥有"无用"的构造,拷贝，复制等类时，通过萃取机制，可以在编译时完成函数绑定。会直接采用最有效的策略。采用更为高效的memcpy等。 为了构造能够在编译时完成函数绑定，我们只能利用重载机制，那么，我们也就必须构造类型，作为函数参数

	struct __true_type {
	};
	 
	struct __false_type {
	};
	 
	 
	template <class _Tp>
	struct __type_traits { 
	   typedef __true_type     this_dummy_member_must_be_first;
	                   /* Do not remove this member. It informs a compiler which
	                      automatically specializes __type_traits that this
	                      __type_traits template is special. It just makes sure that
	                      things work if an implementation is using a template
	                      called __type_traits for something unrelated. */
	 
	   /* The following restrictions should be observed for the sake of
	      compilers which automatically produce type specific specializations 
	      of this class:
	          - You may reorder the members below if you wish
	          - You may remove any of the members below if you wish
	          - You must not rename members without making the corresponding
	            name change in the compiler
	          - Members you add will be treated like regular members unless
	            you add the appropriate support in the compiler. */
	  
	 
	   typedef __false_type    has_trivial_default_constructor;
	   typedef __false_type    has_trivial_copy_constructor;
	   typedef __false_type    has_trivial_assignment_operator;
	   typedef __false_type    has_trivial_destructor;
	   typedef __false_type    is_POD_type;
	 
	};

SGI 为每一个内嵌类型都定义为默认__false_type。这样来保证最底线的正确。因为如果判断错误则会有致命的错误。

然后为每一个标准类型设计特化版本。从而里引用偏特化机制来保证整个机制运行。比如

	__STL_TEMPLATE_NULL struct __type_traits<char> {
	   typedef __true_type    has_trivial_default_constructor;
	   typedef __true_type    has_trivial_copy_constructor;
	   typedef __true_type    has_trivial_assignment_operator;
	   typedef __true_type    has_trivial_destructor;
	   typedef __true_type    is_POD_type; // plain old data
	};

	template <class _ForwardIter, class _Size, class _Tp>
	inline _ForwardIter 
	uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x)
	{
	  return __uninitialized_fill_n(__first, __n, __x, __VALUE_TYPE(__first));
	}
	 
	 
	template <class _ForwardIter, class _Size, class _Tp, class _Tp1>
	inline _ForwardIter 
	__uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x, _Tp1*)
	{
	    //这里根据传入的类型_Tp1，得到了is_POD_type 类型。并定义了一个类型_Is_POD
	    //通过_Is_POD()构造一个临时对象，并传入函数参数。
	  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;
	  return __uninitialized_fill_n_aux(__first, __n, __x, _Is_POD());
	}
	 
	//_Is_POD 类型 == __true_type 执行
	//这里其实，并没有调用构造函数。
	template <class _ForwardIter, class _Size, class _Tp>
	inline _ForwardIter
	__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
	                           const _Tp& __x, __true_type)
	{
	  return fill_n(__first, __n, __x);
	}
	 
	//_Is_POD 类型 == __false_type 执行
	//这里，我们可以看出，在构造多个函数的时候，这里采用了c++的异常处理，保证如果有异常出现，
	//构造过的对象能够被析构掉。当然，那个被构造了一半的对象是不会被析构的，也可能会造成memory leak，所以，切忌不要
	//在构造函数中抛出异常。
	template <class _ForwardIter, class _Size, class _Tp>
	_ForwardIter
	__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
	                           const _Tp& __x, __false_type)
	{
	  _ForwardIter __cur = __first;
	  __STL_TRY {
	    for ( ; __n > 0; --__n, ++__cur)
	      construct(&*__cur, __x);
	    return __cur;
	  }
	  __STL_UNWIND(destroy(__first, __cur));
	}
	//类似的这样的，还有许多特化后的版本。这里从略。

类似这样的设计，充斥在SGI STL中，在这里，任何一个小小的开销都被认为是无法接受的。的确。这里给人一种真实的理想的世界。如果你对code 有洁癖，SGI STL，是不能错过的。


