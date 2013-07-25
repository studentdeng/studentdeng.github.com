---
layout: post
title: "SGI STL 学习笔记二 vector"
date: 2011-01-01 21:39
comments: true
categories: [STL, algorithms]
---

sequence containers
	Array
	Vector
	Heap
	Priority_queue
	List
	sList（not in standard）
	Deque
	Stack
	Queue
Sequence Containers 其中的元素 都是可序的（ordered），但并不一定有序（sorted）。STL 中有vector ，list ，deque，stack，queue，priority_queue等序列容器。Stack queue 由于只是将deque重新封装而成，在技术上被归类为一种配接器(adapter)。

Vector
Vector 的数据为动态空间，随着元素的加入。内部会通过机制自行扩充空间，以容纳新元素。
Vector 的效率，在于对大小的控制，重新分配时数据移动效率。当空间不足时，vector会选择策略扩充容量。
Vector resize之后，很可能使所有迭代器均失效。 插入后，插入点之前的Iterator 有效，其他则无效。eraser迭代器失效。
Vector实现
Vector 实现比较简单。这里仅仅作为打开SGI STL的敲门砖。

我这里的SGI STL 对vector有进行了进一步封装。在头文件中，也给出了我们的解释。

// The vector base class serves two purposes. First, its constructor

// and destructor allocate (but don't initialize) storage. This makes

// exception safety easier. Second, the base class encapsulates all of

// the differences between SGI-style allocators and standard-conforming

// allocators.

这里根据 宏 __STL_USE_STD_ALLOCATORS 来决定是否资源分配器。如果定义了__STL_USE_STD_ALLOCATORS， 则使用allocator< _Tp >，否则为alloc


	//这里的 _Vector_base 为我们隐藏了 使用STL 标准分配器，和SGI 自己特有的分配器之间的不同
	//我们现在先把这里具体的分配细节透明。
	//这是，使用SGI 自己的分配器
	template <class _Tp, class _Alloc> 
	class _Vector_base {
	public:
	  typedef _Alloc allocator_type;
	  allocator_type get_allocator() const { return allocator_type(); }
	 
	  _Vector_base(const _Alloc&)
	    : _M_start(0), _M_finish(0), _M_end_of_storage(0) {}
	  _Vector_base(size_t __n, const _Alloc&)
	    : _M_start(0), _M_finish(0), _M_end_of_storage(0) 
	  {
	    _M_start = _M_allocate(__n);
	    _M_finish = _M_start;
	    _M_end_of_storage = _M_start + __n;
	  }
	 
	  ~_Vector_base() { _M_deallocate(_M_start, _M_end_of_storage - _M_start); }
	 
	protected:
	  _Tp* _M_start;
	  _Tp* _M_finish;
	  _Tp* _M_end_of_storage;
	 
	  typedef simple_alloc<_Tp, _Alloc> _M_data_allocator;
	  _Tp* _M_allocate(size_t __n)
	    { return _M_data_allocator::allocate(__n); }
	  void _M_deallocate(_Tp* __p, size_t __n) 
	    { _M_data_allocator::deallocate(__p, __n); }
	};
 

	template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >
	class vector : protected _Vector_base<_Tp, _Alloc> 
	{
	private:
	  typedef _Vector_base<_Tp, _Alloc> _Base;
	public:
	  typedef _Tp value_type;
	  typedef value_type* pointer;
	  typedef const value_type* const_pointer;
	  typedef value_type* iterator;
	  typedef const value_type* const_iterator;
	  typedef value_type& reference;
	  typedef const value_type& const_reference;
	  typedef size_t size_type;
	  typedef ptrdiff_t difference_type;
	 
	  typedef typename _Base::allocator_type allocator_type;
	  allocator_type get_allocator() const { return _Base::get_allocator(); }
	…
	...
	};
 

分析vector，首先看他的Iterator。

typedef value_type* iterator;

typedef const value_type* const_iterator;

我们可以看出，vector的Iterator 就是一个指针。若是定义

vector<int>:: iterator iter1;

vector<RECT>:: iterator iter2;

那么，Iter1 其实，就是int *， iter2其实就是RECT * 。

看一下，部分的vector 函数，也是我们常常使用的。

Vector 的数据，什么时候被释放。我们需要看析构函数。

	~vector() { destroy(_M_start, _M_finish); }
	 
	template <class _ForwardIterator>
	inline void destroy(_ForwardIterator __first, _ForwardIterator __last) {
	  __destroy(__first, __last, __VALUE_TYPE(__first));
	}
	 
	#define __VALUE_TYPE(__i)        __value_type(__i)

下面是2个偏特化版本。可以看出，在一些特殊情况下，我们找到了最快速的方法。什么也不干。

	inline void destroy(char*, char*) {}
	inline void destroy(wchar_t*, wchar_t*) {}
	 
	template <class _Iter>
	inline typename iterator_traits<_Iter>::value_type*
	__value_type(const _Iter&)
	{
	   //这里，仅仅构造一个临时对象（准确说是指针）来做返回值，事实上，我们不关心他到底是个什么，只是关心她的类型。
	    //用这个类型来激发函数重载，所以，用0来构造也无妨。
	  return static_cast<typename iterator_traits<_Iter>::value_type*>(0);
	}
	 
	template <class _ForwardIterator, class _Tp>
	inline void
	__destroy(_ForwardIterator __first, _ForwardIterator __last, _Tp*)//这里多了一个接受这个类型对象参数
	{
	    //根据这个类型_Tp，我们根据__type_traits<_Tp>，找到了这个类型是否有has_trivial_destructor。
	  typedef typename __type_traits<_Tp>::has_trivial_destructor _Trivial_destructor;
	    //然后构造一个临时的对象来激发函数重载。
	  __destroy_aux(__first, __last, _Trivial_destructor());
	}
	 
	//下面2个便是特化后的结果。
	//__false_type,我们老老实实的该干什么干什么。
	template <class _ForwardIterator>
	inline void
	__destroy_aux(_ForwardIterator __first, _ForwardIterator __last, __false_type)
	{
	  for ( ; __first != __last; ++__first)
	    destroy(&*__first);
	}
	 
	//__true_type 我们实在是没有这个必要和他纠结了。
	template <class _ForwardIterator>
	inline void __destroy_aux(_ForwardIterator, _ForwardIterator, __true_type) {}

这是可能怀疑，内存到那里释放呢？ 别忘了，我们的vector 是继承自_Vector_base，内存释放，管理都隐藏在他那里。

~_Vector_base() { _M_deallocate(_M_start, _M_end_of_storage - _M_start); }

这里才真正的执行内存的回收。但是这里又涉及到了SGI STL 的内存管理，这部分是给操作系统，还是给内存池呢？

在没有研究过细致的内存管理之前。我们还是将这里透明吧。

基本操作

	iterator begin() { return _M_start; }
	const_iterator begin() const { return _M_start; }
	iterator end() { return _M_finish; }
	const_iterator end() const { return _M_finish; }
	size_type size() const { return size_type(end() - begin()); }
	size_type capacity() const { return size_type(_M_end_of_storage - begin()); }
	bool empty() const { return begin() == end(); }
	 
	void push_back(const _Tp& __x) {
	    if (_M_finish != _M_end_of_storage) {
	      construct(_M_finish, __x);
	      ++_M_finish;
	    }
	    else
	      _M_insert_aux(end(), __x);
	  }
	 
	  void push_back() {
	    if (_M_finish != _M_end_of_storage) {
	      construct(_M_finish);
	      ++_M_finish;
	    }
	    else
	      _M_insert_aux(end());
	  }
	 
	void resize(size_type __new_size, const _Tp& __x) {
	    if (__new_size < size()) 
	      erase(begin() + __new_size, end());
	    else
	      insert(end(), __new_size - size(), __x);
	  }
	 
	  void resize(size_type __new_size) { resize(__new_size, _Tp()); }
 

删除 erase

	iterator erase(iterator __position) {
	    if (__position + 1 != end())
	      copy(__position + 1, _M_finish, __position);
	    --_M_finish;
	    destroy(_M_finish);
	    return __position;
	  }
	  iterator erase(iterator __first, iterator __last) {
	    iterator __i = copy(__last, _M_finish, __first);
	    destroy(__i, _M_finish);
	    _M_finish = _M_finish - (__last - __first);
	    return __first;
	  }

Copy 是全局函数，操作简单，同样有多个特化版本。Vector 和一般数组的删除动作一样，将后面元素一个个往前搬。最后修改个数。

插入 insert

	iterator insert(iterator __position, const _Tp& __x) {
	    size_type __n = __position - begin();
	    if (_M_finish != _M_end_of_storage && __position == end()) {
	      construct(_M_finish, __x);
	      ++_M_finish;
	    }
	    else
	      _M_insert_aux(__position, __x);
	    return begin() + __n;
	  }
	  iterator insert(iterator __position) {
	    size_type __n = __position - begin();
	    if (_M_finish != _M_end_of_storage && __position == end()) {
	      construct(_M_finish);
	      ++_M_finish;
	    }
	    else
	      _M_insert_aux(__position);
	    return begin() + __n;
	  }
	 
	 
	template <class _Tp, class _Alloc>
	void
	vector<_Tp, _Alloc>::_M_insert_aux(iterator __position)
	{
	  if (_M_finish != _M_end_of_storage) {
	    construct(_M_finish, *(_M_finish - 1));
	    ++_M_finish;
	    copy_backward(__position, _M_finish - 2, _M_finish - 1);
	    *__position = _Tp();
	  }
	  else {
	    const size_type __old_size = size();
	    const size_type __len = __old_size != 0 ? 2 * __old_size : 1;
	    iterator __new_start = _M_allocate(__len);
	    iterator __new_finish = __new_start;
	    __STL_TRY {
	      __new_finish = uninitialized_copy(_M_start, __position, __new_start);
	      construct(__new_finish);
	      ++__new_finish;
	      __new_finish = uninitialized_copy(__position, _M_finish, __new_finish);
	    }
	    __STL_UNWIND((destroy(__new_start,__new_finish), 
	                  _M_deallocate(__new_start,__len)));
	    destroy(begin(), end());
	    _M_deallocate(_M_start, _M_end_of_storage - _M_start);
	    _M_start = __new_start;
	    _M_finish = __new_finish;
	    _M_end_of_storage = __new_start + __len;
	  }
	}

的确很简单，和我们在学校学的并没有什么大的不同，只是在对新增元素的构造上不同。

	construct(__new_finish)，

	construct(_M_finish, *(_M_finish - 1));

	以上construct是全局函数，同样有特化版本。将类的构造分成，资源分配 + 构造函数，来做到提高效率。这样在大量数据上效果应该很明显，并没有具体测试。

	对一次插入大量元素时，vector 的策略是。
	if (插入元素个数 == 0 ) return
	if (判断容量是否足够)
	{
	    if (插入点后的元素个数 > 待插入元素个数)
	    {
	       按照最后一个元素，构造插入元素个数个元素。
	         向插入点数据向后搬运。
	         移动指针。
	         将待插入元素顺次插入。
	    }
	    else
	    {
	       先以__x构造元素，在不需要移动位置的地方。
	         将原来的元素，移动到最后。
	         在插入位置处，以__x构造元素。
	    }
	}
	else
	{
	    根据策略分配空间（这里至少PJ 和SGI的策略不同，这里应该和不同的内存管理策略有关）
	    将插入点之前的原有的数据复制到新空间
	    依次复制新元素到新空间。
	    依次复制原来数据到新空间
	}


	template <class _Tp, class _Alloc>
	void vector<_Tp, _Alloc>::insert(iterator __position, size_type __n, const _Tp& __x)
	{
	  if (__n != 0) {
	    if (size_type(_M_end_of_storage - _M_finish) >= __n) {
	      _Tp __x_copy = __x;
	      const size_type __elems_after = _M_finish - __position;
	      iterator __old_finish = _M_finish;
	      if (__elems_after > __n) {
	        uninitialized_copy(_M_finish - __n, _M_finish, _M_finish);
	        _M_finish += __n;
	        copy_backward(__position, __old_finish - __n, __old_finish);
	        fill(__position, __position + __n, __x_copy);
	      }
	      else {
	        uninitialized_fill_n(_M_finish, __n - __elems_after, __x_copy);
	        _M_finish += __n - __elems_after;
	        uninitialized_copy(__position, __old_finish, _M_finish);
	        _M_finish += __elems_after;
	        fill(__position, __old_finish, __x_copy);
	      }
	    }
	    else {
	      const size_type __old_size = size();        
	      const size_type __len = __old_size + max(__old_size, __n);
	      iterator __new_start = _M_allocate(__len);
	      iterator __new_finish = __new_start;
	      __STL_TRY {
	        __new_finish = uninitialized_copy(_M_start, __position, __new_start);
	        __new_finish = uninitialized_fill_n(__new_finish, __n, __x);
	        __new_finish
	          = uninitialized_copy(__position, _M_finish, __new_finish);
	      }
	      __STL_UNWIND((destroy(__new_start,__new_finish), 
	                    _M_deallocate(__new_start,__len)));
	      destroy(_M_start, _M_finish);
	      _M_deallocate(_M_start, _M_end_of_storage - _M_start);
	      _M_start = __new_start;
	      _M_finish = __new_finish;
	      _M_end_of_storage = __new_start + __len;
	    }
	  }
	}
