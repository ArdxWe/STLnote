# 序列式容器

## `vector`

与普通数组非常相似, 优点是动态空间, 随着新元素的加入, 其内部机制会自动扩充. `vector` 的实现关键在于两点:

* 对容量大小的控制

* 重新配置时的数据移动效率

相信用过 `C++` 的人都用过 `vector` , 我们来看一下 `vector` 部分源代码:

```cpp
template <class T, class Alloc = alloc>  // T代表容器内嵌型别, 空间适配器默认使用第二级空间适配器 所以定义时可以 vector<int>
class vector {
    public:
    // 嵌套型别定义 如果你不懂 请看第三章 traits 部分
    typedef T value_type;
    typedef value_type* pointer;
    typedef value_type* iterator;  // 迭代器是原生指针
    typedef value_type& reference;
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;

    protected:

    typedef simple_alloc<value_type, Alloc> data_allocator;  // 空间配置器

    iterator start;  // 头
    iterator finish;  // 使用元素的尾部
    iterator end_of_storage;  // 可用空间的尾部
    ...
}
```

显然, 利用 `start` `finish` `end_of_storage` 很容易写出 `vector` 大小, 容量, 空容器的判断这些函数, 这里不再赘述.

缺省使用 `alloc` 做为空间配置器, 还定义了对应的 `simple_alloc` : `data_allocator` . 真正配置空间时使用 `data_allocator::allocate(n)` 等函数.

例如一个允许指定大小和初值的 `constructors` :

```cpp
vector(size_type n, const T& value) {  // 个数 n , 初值 value
    fill_initialize(n, value);  // 填充并初始化 显然还应该处理 vector 维护的数据结构, 如 start finish 等
}

void fill_ininialize(size_type n, const T& value) {
    start = allocate_and_fill(n, value);  // 分配空间并填初值 返回起始位置迭代器
    finish = start + n;
    end_of_storage = finish;
}

iterator allocate_and_fill(size_type n, const T& x) {
    // 第二章都有介绍
    iterator result = data_allocator::allocate(n);  // 分配 n * sizeof(T) 内存 返回头指针
    uninitialized_fill_n(result, n, x);  // 填充值 通过 traits 机制提升效率
    return result;
}
```

看 `vector` 的插入函数 `push_back` :

```cpp
void push_back(const T& x) {
    if (finish != end_of_storage) {  // 还有空间
        construct(finish, x);
        finish++;
    }
    else {
        insert_aux(end(), x);  // 尾部
    }
}

template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator positon, const T& x) {
    if (finish != end_of_storage) {  // 还有空间
        construct(finish, *(finish-1));  // 最后一个元素复制到新的空间
        finish++;
        T x_copy = x;  // 不是很懂这里为什么要保存 x
        copy_backend(positon, finish - 2, finish - 1);  // positon 到 finish - 2 往后移动
        *positon = x_copy;  // 赋值
    }
    else {  // 无多余空间
        const size_type old_size = size();
        const size_type len = old_size != 0 ? 2*old_size : 1;  // 初始是 0 就变为 1 不为 0 就 乘 2
        iterator new_start = data_allocator::allocate(n);  // 分配空间
        iterator new_finish = new_start;
        try {
            // 简单的复制
            new_finish = uninitialized_copy(start, positon, new_start);
            construct(new_finish, x);
            new_finish++;
            new_finish = uninitialized_copy(positon, finish, new_finish);
        }
        catch(...) {
            destroy(new_start, new_finish);  // 析构
            data_allocator::deallocate(new_start, len);  // 释放空间
            throw;
        }

        destroy(begin(), end());  // 原来的空间析构
        deallocate();  // 释放空间
        // 更新迭代器标志
        start = new_start;
        finish = new_finish;
        end_of_storage = new_start + len;
    }
}
```

## list

节点设计:

```cpp
// 双向链表
template <class T>
struct __list_node {
    typedef void* void_pointer;
    void_pointer prev;
    void_pointer next;
    T data;
}
```

迭代器设计

```cpp
template <class T, class Ref, class Ptr>
struct __list_iterator {
    typedef __list_iterator<T, T&, T*> iterator;
    typedef __list_iterator<T, Ref, Ptr> self;
    // traits 型别
    typedef bidirectional_iterator_tag iterator_category;
    typedef T value_type;
    typedef Ptr Pointer;
    typedef Ref reference;
    typedef __list_node<T>* link_type;
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;

    link_type node;  // 指向节点的指针

    // constructor
    __list_iterator(link_type x) : node(x) {}  // 节点指针
    __list_iterator() {}
    __list_iterator(const iterator& x) : node(x.node) {}  // iterator 的引用
    //  == 看指向节点的指针
    bool operator==(const self& x) const {
        return node == x.node;
    }
    // != 看指向节点的指针
    bool operator!=(const self& x) const {
        return node != x.node;
    }
    // 解引用 得到节点的数据
    reference operator*() const {
        return (*node).data;
    }
    // -> 返回对应数据的指针
    pointer operator->() const {
        return &(operator*());
    }
    // ++i
    self& operator++() {
        node = (link_type) ((*node).next());
        return *this;
    }
    // i++
    self operator++(int) {
        self tmp = *this;
        ++(*this);
        return tmp;
    }

    self& operator--() {
        node = (link_type) ((*node).prev());
        return *this;
    }

    self operator--(int) {
        self tmp = *this;
        --(*this);
        return tmp;
    }
}
```

`list` 的构造

```cpp
template <class T, class Allloc = alloc>  // 默认使用 alloc
class list {
    protected:

    typedef __list_node<T> list_node;
    typedef simple_alloc<list_node, Alloc> list_node_allocator;

    link_type get_node() {
        return list_node_allocator::allocate();
    }

    link_type put_node(link_type p) {
        return list_node_allocator::deallocate(p);
    }

    link_type create_node(const T& x) {
        link_type p = get_node();
        construct(&p->data, x);  // 指针 值
        return p;
    }

    void destroy_node(link_type p) {
        destroy(&p->data);
        put_node(p);
    }

    void empty_initialize() {
        node = get_node();
        node->next = node;  // 双向链表
        node->prev = node;
    }

    public:
    // constructor
    list() {
        empty_initialize();
    }
}
```
