---
layout: post
title: "SGI STL 学习笔记四 内存管理"
date: 2011-01-17 22:46
comments: true
categories: [STL, algorithms]
---

SGI STL 在g++中默认的编译选项是构造2个分配器。

# 第一级分配器__malloc_alloc_template
这个一级分配器设计比较简单。由于SGI STL中分配内存没有使用C++推荐的 operator new/delete 而是使用malloc/delete。所以，并没有set_new_handler()。当面对内存不足的情况，这里模仿了c++的做法。

	template <int __inst>
	void (* __malloc_alloc_template<__inst>::__malloc_alloc_oom_handler)() = 0;
	 
	static void (* __set_malloc_handler(void (*__f)()))()
	{
	    void (* __old)() = __malloc_alloc_oom_handler;
	    __malloc_alloc_oom_handler = __f;
	    return(__old);
	}
	 
	template <int __inst>
	void*
	__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
	{
	    void (* __my_malloc_handler)();
	    void* __result;
	 
	    for (;;) {
	       //这里不断的调用处理分配不足的情况代码，如果有可能解决问题，那么OK，如果还是不行，那么
	         //只能抛出异常，当然，如果没有指定，默认为NULL，则会直接抛出异常。
	        __my_malloc_handler = __malloc_alloc_oom_handler;
	        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }
	        (*__my_malloc_handler)();
	        __result = malloc(__n);
	        if (__result) return(__result);
	    }
	}
	 
	//这里，如果分配不足，则会调用_S_oom_malloc来救急。oom，就是out of memory的意思。
	static void* allocate(size_t __n)
	{
	    void* __result = malloc(__n);
	    if (0 == __result) __result = _S_oom_malloc(__n);
	    return __result;
	 
	}

# 二级分配器 __default_alloc_template
二级配置器多了很多机制，在分配小的内存上做了优化。

粗略的分配策略。

分配大小超过 _MAX_BYTES = 128bytes，使用一级分配器处理。当分配器大小小于128bytes时，则通过内存池管理。
调整分配大小到8的倍数，从freeList中，分配内存。

	template <bool threads, int inst>
	class __default_alloc_template {
	 
	private:
	  // Really we should use static const int x = N
	  // instead of enum { x = N }, but few compilers accept the former.
	# ifndef __SUNPRO_CC
	    enum {_ALIGN = 8}; 
	    enum {_MAX_BYTES = 128};
	    enum {_NFREELISTS = _MAX_BYTES/_ALIGN}; //free_list 个数
	# endif
	  static size_t
	  _S_round_up(size_t __bytes)
	    { return (((__bytes) + _ALIGN-1) & ~(_ALIGN - 1)); }
	 
	__PRIVATE:
	  union _Obj {
	        union _Obj* _M_free_list_link;
	        char _M_client_data[1];    /* The client sees this.        */
	  };
	//根据大小，找到对应的index，从1开始。
	static  size_t _S_freelist_index(size_t __bytes) {
	        return (((__bytes) + _ALIGN-1)/_ALIGN - 1);
	}

可以看出，_Obj就是一个简单的单向链表，只是这个链表和我们之前学习的不一样。之前的链表数据只是链表节点的一部分，而这里当分配给client的时候是一整块的。

# 分配空间

	static void* allocate(size_t __n)
	  {
	    _Obj* __VOLATILE* __my_free_list;
	    _Obj* __RESTRICT __result;
	 
	    if (__n > (size_t) _MAX_BYTES) {
	        return(malloc_alloc::allocate(__n));
	    }
	    __my_free_list = _S_free_list + _S_freelist_index(__n);
	    // Acquire the lock here with a constructor call.
	    // This ensures that it is released in exit or during stack
	    // unwinding.
	#ifndef _NOTHREADS
	        /*REFERENCED*/
	        _Lock __lock_instance;
	#endif
	    __result = *__my_free_list;
	    if (__result == 0) { //没有找到freeList
	        void* __r = _S_refill(_S_round_up(__n)); //这里重新填充 freeList
	        return __r;
	    }
	    *__my_free_list = __result -> _M_free_list_link;
	    return (__result);
	  };

如果能够获得freeList，情况很简单，将__my_free_list 的值，指向已经分配空间的下一块空间。

	/* Returns an object of size __n, and optionally adds to size __n free list.*/
	/* We assume that __n is properly aligned.                                */
	/* We hold the allocation lock.                                         */
	template <bool __threads, int __inst>
	void*
	__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
	{
	    int __nobjs = 20;
	     //由他来分配空间，第二个参数为引用， 所以有可能出现分配不足，也就是__nobjs < 20的情况。
	    char* __chunk = _S_chunk_alloc(__n, __nobjs); 
	    _Obj* __VOLATILE* __my_free_list;
	    _Obj* __result;
	    _Obj* __current_obj;
	    _Obj* __next_obj;
	    int __i;
	 
	    if (1 == __nobjs) return(__chunk); //如果只能分配一个大小，那么我们不需要调整freeList了。
	    __my_free_list = _S_free_list + _S_freelist_index(__n);
	 
	    /* Build free list in chunk */ //构造freeList
	      __result = (_Obj*)__chunk;
	      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);
	      for (__i = 1; ; __i++) { //从第二个开始构造freeList，因为第一个需要传给上层函数。
	        __current_obj = __next_obj;
	        __next_obj = (_Obj*)((char*)__next_obj + __n);
	        if (__nobjs - 1 == __i) {
	            __current_obj -> _M_free_list_link = 0;
	            break;
	        } else {
	            __current_obj -> _M_free_list_link = __next_obj;
	        }
	      }
	     //这里，我们发现，这个freeList，其实就是一个简单的单向链表，
	    //__my_free_list = _S_free_list + _S_freelist_index(__n); 这个就是获得这个链表的表头。
	    return(__result);
	}

# 释放空间

	/* __p may not be 0 */
	  static void deallocate(void* __p, size_t __n)
	  {
	    _Obj* __q = (_Obj*)__p;
	    _Obj* __VOLATILE* __my_free_list;
	 
	    if (__n > (size_t) _MAX_BYTES) {
	        malloc_alloc::deallocate(__p, __n); //过大的block，我们通过1级分配器搞定
	        return;
	    }
	    __my_free_list = _S_free_list + _S_freelist_index(__n);
	    // acquire lock
	#       ifndef _NOTHREADS
	        /*REFERENCED*/
	        _Lock __lock_instance;
	#       endif /* _NOTHREADS */
	     //这里是典型的在listHead 的下一个位置添加的操作，这里看出，对于小的block，我们并没有给操作系统，而是
	    //链表保存起来。
	    __q -> _M_free_list_link = *__my_free_list;
	    *__my_free_list = __q;
	    // lock is released here
	  }

# 内存池分配

	/* We allocate memory in large chunks in order to avoid fragmenting     */
	/* the malloc heap too much.                                            */
	/* We assume that size is properly aligned.                             */
	/* We hold the allocation lock.                                         */
	template <bool __threads, int __inst>
	char*
	__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size,
	                                                            int& __nobjs)
	{
	    char* __result;
	    size_t __total_bytes = __size * __nobjs;
	    size_t __bytes_left = _S_end_free - _S_start_free;
	 
	    if (__bytes_left >= __total_bytes) {
	          //内存池足够，分配后返回
	        __result = _S_start_free;
	        _S_start_free += __total_bytes;
	        return(__result);
	    } else if (__bytes_left >= __size) {
	          //内存池不够整个memory block，但是足够一个以上的block。
	        __nobjs = (int)(__bytes_left/__size);
	        __total_bytes = __size * __nobjs;
	        __result = _S_start_free;
	        _S_start_free += __total_bytes;
	        return(__result);
	    } else {
	          //内存池已经一个都不能满足memory block
	        size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
	        // Try to make use of the left-over piece.
	        if (__bytes_left > 0) {
	               //内存池中还有剩余，找到这部分空间并填入相应的memory block list中。
	            _Obj* __VOLATILE* __my_free_list =
	                        _S_free_list + _S_freelist_index(__bytes_left);
	 
	            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
	            *__my_free_list = (_Obj*)_S_start_free;
	        }
	          //内存池为空，我们从heap中找内存
	          //__bytes_to_get是一个不断增加的数字，也就是每次从heap分配的空间越来越多。
	        _S_start_free = (char*)malloc(__bytes_to_get);  
	        if (0 == _S_start_free) {
	                //极端情况， heap的空间不足。
	            size_t __i;
	            _Obj* __VOLATILE* __my_free_list;
	     _Obj* __p;
	            // Try to make do with what we have.  That can't
	            // hurt.  We do not try smaller requests, since that tends
	            // to result in disaster on multi-process machines.
	                //这部分不理解，为什么只能从大的block中分配呢？
	            for (__i = __size; __i <= _MAX_BYTES; __i += _ALIGN) {
	                __my_free_list = _S_free_list + _S_freelist_index(__i);
	                __p = *__my_free_list;
	                if (0 != __p) {
	                          //在freeList 中，我们找到了一个大的block，并且里面有数据
	                    *__my_free_list = __p -> _M_free_list_link;
	                    _S_start_free = (char*)__p;
	                    _S_end_free = _S_start_free + __i;
	                          //从大的freelist中，我们分配出一个来到内存池，然后递归。这次没有
	                              //意外，会找到足够的内存
	                    return(_S_chunk_alloc(__size, __nobjs));
	                    // Any leftover piece will eventually make it to the
	                    // right free list.
	                }
	            }
	                //还是没有满足要求，调用一级分配器看oom机制，是否能够帮助我们
	     _S_end_free = 0; // In case of exception.
	            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
	            // This should either throw an
	            // exception or remedy the situation.  Thus we assume it
	            // succeeded.
	        }
	          //扩充了内存池大小
	        _S_heap_size += __bytes_to_get;
	        _S_end_free = _S_start_free + __bytes_to_get;
	        return(_S_chunk_alloc(__size, __nobjs));//根据新的内存池大小，修正__nobjs
	    }//end else
	     //这里我们看出了，为什么是8的倍数，所以，我们分配的大的block，肯定会在小的block中找到位置，而不会
	      //浪费掉当然，具体的理由肯定还有更多，需要去了解更多的内存方面的知识才能理解。
	}

内存管理这里仅仅是一个最最基本的梳理，事实上其实仅仅是照本宣科而已，因为这里面有太多的细节需要去琢磨。STL设计这样的原因是什么？他的分配回收机制为什么是这样？他的分配粒度，以及处理分配不足的手段。对我来说都是未知，不过好在目前的确不需要思考过多这些问题。仅仅是上层的封装和简单功能的实现就已经让我受益匪浅了。这些问题还是留给时间去沉淀这些未知吧。


