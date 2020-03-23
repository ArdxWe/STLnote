# 空间配置器 allocator

## SGI空间适配器 std::alloc

一般我们所习惯的C++内存配置和释放操作如下

```C++
class Foo { ... };
Foo *pf = new Foo;
delete pf;
```

其中new包含两个阶段

* 调用 `::operator new` 配置内存
* 调用 `Foo::Foo()` 构造对象内容

delete包含类似两个阶段

* 调用 `Foo:~Foo()` 将对象析构
* 调用 `::operator delete` 释放内存

STL allocator将两个阶段的操作区分

* 内存配置 `alloc::allocate()`
* 内存释放 `alloc::deallocate()`
* 对象构造 `::construct()`
* 对象析构 `::destroy()`

## 对象构造

```cpp
template <class T1, class T2>
inline void construct(T1* p, const T2& value) {
    new (p) T1(value);  // placement new
}
```

`construct()` 接受一个指针 `p` 和一个初值 `value` , 将初值设定到指针所指空间

## 对象析构

```C++
// first version
template <class T>
inline void destroy(T* pointer) {
    pointer->~T();
}
```

这是 `destroy()` 的第一个版本, 接受一个指针, 直接调用该对象的析构函数.

```cpp
// second version
// 接受两个迭代器, 找出元素数值型别, 利用 __type_traits<> 调用不同函数
template <class ForwardIterator>
inline void destroy(ForwardIterator first, ForwardIterator last) {
    __destroy(first, last, value_type(first));
}

// 判断 value_type 是否有 trivial destructor 即无析构函数
template <class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) {
    typedef typename __type_traits<T>::has_trivial_destructor trivial_destructor;
    __destroy_aux(first, last, trivial_destructor());
}

// non-trivial destructor
template <class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last, __false_type) {
    for( ; first < last; ++first) {
        destroy(&*first);  // 调用 first version 参数为迭代器所指对象的地址
    }
}

// trivial destructor
template <class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last, __true_type) {}  // do nothing

// second version 针对迭代器为 char* 和 wchar_t* 的特化版
inline void destroy(char*, char*) {}  // do nothing
inline void destroy(wchar_t*, wchar_t*) {} // do nothing

```

第二版本, 接受 `first` 和 `last` 两个迭代器, 将 `[first, last)` 范围内所有对象析构. 利用 `value_type()` 得到迭代器所指对象的类型, 然后利用 `_type_traits<T>`判断该类别析构函数是否无关痛痒, 是则不做任何事, 否则遍历整个范围一次调用第一个版本的 `destroy()`  
`value_type()` 和 `__type_traits_()` 具体实现见下章.

## 内存配置与释放

SGI有双层级配置器

* 第一级使用 `malloc()` 和 `free()`
* 第二级使用不同策略, 区块大于 `128bytes` 时, 调用第一级, 小于 `128bytes` 时, 使用 `memory pool`  

若 `__USE_MALLOC` 被定义, 开放第一级, 否则开放第二级.

```cpp
# ifdef __USE_MALLOC

    typedef __malloc_alloc_template<0> malloc_alloc;
    typedef malloc_alloc alloc; // 第一级

# else
    typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc;  // 第二级
# endif /* ! __USE_MALLOC */
```

`alloc` 不接受 `template` 参数, `SGI` 会为配置器包装一个接口来符合 `STL` 规格:

```cpp
template <class T, class Alloc>
class simple_alloc {
    public:
    // 简单的转调用
    static T* allocate(size_t n) {
        return 0 == n ? 0 : (T*) Alloc::allocate(n * sizeof(T));
    }

    static T* allocate(void) {
        return (T*) Alloc::allocate(sizeof(T));
    }

    static void deallocate(T* p, size_t n) {
        if (0 != n) {
            Alloc::deallocate(p, n * sizeof(T));
        }
    }

    static void deallocate(T *p) {
        Alloc::deallocate(p, sizeof(T));
    }
};
```

`SGI` 容器全部使用这个 `simple_alloc` 接口:

```cpp
// 实际运用方式
template <class T, class Alloc = alloc>  // 缺省使用alloc
class vector {
    typedef simple_alloc<T, Alloc> data_allocator;

    data_allocator::allocate(n);

};
```

### 第一级配置器 `__malloc_alloc_template`

```cpp
template <int inst>
class __malloc_alloc_template {
    private:
    // 处理内存不足
    static void* oom_malloc(size_t);
    static void* oom_realloc(void*, size_t);
    static void (*__malloc_alloc_oom_handler) ();

    public:
    // 分配 n bytes
    static void* allocate(size_t n) {
        void* result = malloc(n);  // 直接malloc
        // out of memory
        if (0 == result) {
            result = oom_malloc(n);
        }
        return result;
    }

    // 回收 p
    static void deallocate(void* p, size_t /* n */) {
        free(p);
    }

    // 再分配 p new_sz bytes
    static void* reallocate(void* p, size_t /* old_sz */, size_t new_sz) {
        void* result = realloc(p, new_sz);  // 直接realloc
        // out of memory
        if (0 == result) {
            result = oom_realloc(p, new_sz);
        }
        return result;
    }

    // 指定 out of memory handler
    static void (*set_malloc_handler(void (*f)()) ) () {  // 这一行的语法我并没有看懂
        void (*old)() = __malloc_alloc_oom_handler;
        __malloc_alloc_oom_handler = f;
        return (old);  // 返回之前的 out of memory handler
    }
};

// 初始设为空指针
template <int inst>
void (* __malloc_alloc_template<inst>::__malloc_alloc_oom_handler) () = 0;

// out of memory malloc
template <int inst>
void* __malloc_alloc_template<inst>::oom_malloc(size_t n) {
    void (*my_alloc_handler) ();
    void *result;

    for (;;) {  // 一直尝试 释放 配置...
        my_malloc_handler = __malloc_alloc_oom_handle;
        if (0 == my_malloc_handler) {
            __THROW_BAD_ALLOC;  // 未指定处理例程 直接抛异常
        }
        (*my_malloc_handler)();  // 处理例程
        result = malloc(n);  // 配置内存
        if (result) {
            return result;
        }
    }
}

// out of memory realloc
template <int inst>
void* __malloc_alloc_template<inst>::oom_realloc(void* p, size_t n) {
    void (*my_alloc_handler) ();
    void *result;

    for (;;) {  // 一直尝试 释放 配置...
        my_malloc_handler = __malloc_alloc_oom_handle;
        if (0 == my_malloc_handler) {
            __THROW_BAD_ALLOC;  // 未指定处理例程 直接抛异常
        }
        (*my_malloc_handler)();  // 处理例程
        result = realloc(p, n);  // 配置内存
        if (result) {
            return result;
        }
    }
}

// inst 指定为 0
typedef __malloc_alloc_template<0> malloc_alloc;
```

`__malloc_alloc_oom_handler` 是内存不足处理例程, 设定此函数是客端责任.

### 第二级配置器 `__default_alloc_template`

为了避免小额区块造成的内存碎片和配置时的额外负担, 采取以下策略
