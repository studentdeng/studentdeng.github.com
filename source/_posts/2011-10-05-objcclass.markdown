---
layout: post
title: "objcclass"
date: 2011-10-05 23:31
comments: true
categories: [Objective-C]
---

之前一直做C++开发，最近2个多月转 Objective-C， 入门的时候，遇到了很多的困惑。现在过节，正是解决他们的好时机。

主要参考来自http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html

 Objective-C 也是面向对象的语言，那么，首先需要知道的就是什么是class。

C++ 的class相对 Objective-C 中的class，就简单明了很多了。C++ 中class简单的说，就是一个大的struct， 绝大部分的class可以在编译时决定好class的布局（通过虚继承来的class成员变量只能动态确定）。当然，最关键的是，你不可能在运行时创建一个class，因为所有的class在运行之前已经确定下来，并保存在二进制文件中。

但是， Objective-C 确不同， Objective-C 可以在运行中创建class，修改class等等。那么，改如何定义 Objective-C 中的class呢。

在这之前，我们先看一个简单的，class的实例对象。

	@interface Object 
	{

	    //typedef struct objc_class *Class; 
	    Class isa;    /* A pointer to the instance's class structure */ 
	}

对象包含一个指向class的指针，而这也就意味着，任何包含class 的指针都可以被看做是对象（object）。

	struct objc_class {            
	    struct objc_class *isa;    //这里也有isa指针 
	    struct objc_class *super_class;    //这里还有一个指向基类的指针 
	    const char *name;        
	    long version; 
	    long info; 
	    long instance_size; 
	    struct objc_ivar_list *ivars;

	    struct objc_method_list **methodLists;

	    struct objc_cache *cache; 
	     struct objc_protocol_list *protocols; 
	};

	//新的定义
	typedef struct class_t {

	    struct class_t *isa;

	    struct class_t *superclass;

	    Cache cache;

	    IMP *vtable;

	    class_rw_t *data;

	} class_t;

显然，在 Objective-C 眼中，一切都是对象，甚至包括我们的class。而对象就是class的实例，那么，class是什么的实例呢，metaclass。

事实上，我们并没有解决问题。metaclass 事实上又是root metaclass 的实例，而root metaclass 自己又是 root metaclass 的实例，一图胜千言，不做赘述。

![alt text](/images/objc.png)