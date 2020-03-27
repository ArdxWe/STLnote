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

`push_back` 函数:

```cpp
void push_back(const T& x) {
    insert(end(), x);  // 尾部迭代器 值
}

interator insert(iterator positon, const T& x) {
    // 双向链表串起来
    link_type tmp = create_node(x);
    tmp->next = positon.node;
    rmp->prev = position.node->prev;
    (link_type(positon.node->prev))->next = tmp;
    positon.node->prev = tmp;
    return tmp;
}
```

## `deque`

和 `vector` 比较而言, `deque` 属于双向开口的连续线性空间, 可以在头尾两端进行插入和删除操作.

`deque` 维护一段连续空间成为 `map` , `map` 中的每个元素指向另外一块较大的连续空间成为缓冲区, 缓冲区存放主体内容.

```cpp
templete <class T, class Alloc = alloc, size_t BufSiz = 0>
class deque {
    public:

    typedef  T value_type;
    typedef value_type* pointer;
    ...

    protected:

    typedef pointer* map_pointer;
    map_pointer map;  // 双重指针
    size_type map_size;  // map 大小
    ...
}
```

迭代器:

```cpp
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
    typedef __deque_iterator<T, T&, T*, BufSize> iterator;
    typedef __deque_iterator<T, const T&, const T*, BufSiz> const_iterator;
    static size_t buffer_size() {
        return __deque_buf_size(BufSiz, sizeof(T));
    }

    typedef random_access_iterator_tag iterator_category;
    typedef T value_type;
    typedef Ptr Pointer;
    typedef Ref reference;
    typedef size_t size_type;
    typedef prtdiff_t difference_type;
    typedef T** map_pointer;
    typedef __deque_iterator self;


    T* cur;
    T* first;
    T* last;
    map_pointer node;
    ...
}

// 全局函数
inline size_t __deque_buf_size(size_t n, size_t sz) {
    return n != 0 ? n : (sz < 512 ? size_t(512 / sz) : size_t(1));
}
```

指针运算可能需要跳跃缓冲区

```cpp
// 更新 iterator 数据
void set_node(map_pointer new_node) {
    node = new_node;
    first = *new_node;
    last = first + difference_type(buffer_size());  // 末尾
}

// 重载运算符
reference operator*() const {
    return *cur;
}

pointer operator->() const {
    return &(operator*());
}

difference_type operator-(const self& x) const {
    return difference_type(buffer_size()) * (node - x.node - 1) + (cur - first) + (x.last - x.cur))   // 计算缓冲区距离
}

self& operator++() {
    cur++;
    if (cur == last) {  // 跳跃缓冲区
        set_node(node+1);
        cur = first;
    }
    return *this;
}
// i++
self operator++(int) {
    self tmp = *this;
    ++*this;
    return tmp;
}

self& operator--() {
    if (cur == first) {  // 跳跃缓冲区
        set_node(node-1);
        cur = last;
    }
    cur--;  // 切换到前一个
    return *this;
}

self operator--(int) {
    self tmp = *this;
    --*this;
    return tmp;
}

// 随机存取
self& operator+=(difference_type n) {
    difference_type offset = n + (cur - first);  // 与当前迭代器 first 的距离
    if (offset >==0 && offset < difference_type(buffer_size())) {
        // 在当前缓冲区
        cur += n;
    }
    else {  // 跳跃
        difference_type node_offset = offset > 0 ? offset / difference_type(buffer_size()) : -difference_type((-offset - 1) / buffer_size()) - 1;  // node 的距离
        set_node(node + node_offset);  // 跳到新的缓冲区
        cur = first + (offset - node_offset * difference_type(buffer_size()));  // 设置新的当前的缓冲区元素指针
    }
    return *this;
}

self operator+(difference_type n) const {
    self tmp = *this;
    return tmp += n;
}

self& operator-=(difference_type n) const {
    return *this -= -n;
}

self operator-(difference_type n) const {
    self tmp = *this;
    return tmp -= n;
}

reference operator[](difference_type n) const {
    return *(*this + n)
}

bool operator==(const self& x) const {
    // 两个迭代器相等判断 当前指针
    return cur == x.cur;
}

bool operator!=(const self& x) const {
    return !(*this == x);
}

bool operator<(const self& x) const {
    return (node == x.node) ? (cur < x.cur) : (node < x.node);
}
```

`deque` 的数据结构: `deque` 维护一个 `map` 指针, 还要维护两个迭代器, 分别指向第一个缓冲区的第一个元素和最后一个缓冲区的最后一个元素. 看代码:

```cpp
template <class T, class Alloc = alloc, size_t BufSiz = 0>
class deque {
    public:

    typedef T value_type;
    typedef value_type* pointer;
    typedef size_t size_type;
    typedef __deque_iterator<T, T&, T*, BufSiz> iterator;

    protected:

    typedef pointer* map_pointer;

    iterator start;
    iterator finish;
    map_pointer map;
    size_type map_size;

    // 这些操作很容易完成
    public:

    iterator begin() {return start;}
    iterator end() {return finish;}
    bool empty() const {return finish == start;}
    ...
}
```

`deque` 的构造

```cpp
// 两个空间配置器
protected:

typedef simple_alloc<value_type, Alloc> data_allocator;  // 配置元素
typedef simple_alloc<pointer, Alloc> map_allocator;  // 配置指针
// constructor
deque(int n, const value_type& value): start(), finish(), map(0), map_size(0) {
    fill_initialize(n, value);
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::fill_initialize(size_type n, const value_type& value) {
    create_map_and_nodes(n);  // 分配内存
    map_pointer cur;
    __STL_TRY {
        for (cur = start.node; cur < finish.node; cur++) {
            uninitialized_fill(*cur, *cur + buffer_size(), value);  // 填充值
        }
        uninitialized_fill(finish.first, finish.cur, value);  // 最后一个填充到 cur 即可 create_map_and_nodes() 会更新
    }
    catch(...) {
        ...
    }
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements) {
    size_type num_nodes = num_elements / buffer_size() + 1;  // 整除会多分配一个 node
    map_size = max(initial_map_size(), nums_node + 2);  // 8 和 指定 node + 2 的最大值
    map = map_allocator::allocate(map_size);  // 分配 map 内存
    // nstart 和 nfinish 都在中间
    map_pointer nstart = map + (map_size - num_nodes) / 2;
    map_pointer nfinish = nstart + nums_nodes - 1;
    map_pointer cur;
    __STL_TRY {
        for (cur = nstart; cur <= nfinish; cur++) {
            *cur = allocate_node();  // 分配 node 内存
        }
    }
    catch(...) {
        ...
    }
    // 设置 start 和 finish 两个迭代器
    start.set_node(nstart);
    finish.set_node(nfinish);
    start.cur = start.first;
    finish.cur = finish.first + nums_elements % buffer_size();
}
```

`push_back` 函数:

```cpp
void push_back(const value_type& t) {
    if (finish.cur != finish.last - 1) {
        // 缓冲区还有大于等于 2 个的位置
        construct(finish.cur, t);
        ++finish.cur;
    }
    else {
        push_back_aux(t);
    }
}

void push_back_aux(const value_type& t) {
    value_type t_copy = t;
    reverse_map_at_back();  // 符合某种条件会换一个 map
    *(finish.node + 1) = allocate_node();  // 给后面的分配空间
    ___STL_TRY {
        construct(finish.cur, t_copy);  // 构造
        finish.set_node(finish.node + 1);  // 更新 finish.node
        finish.cur = finish.first;  // 更新 finish.cur
    }
    __STL_UNWIND(deallocate_node(*(finish.node + 1)));  // 释放空间
}

void reserve_map_at_back(size_type nodes_to_add = 1) {
    if (nodes_to_add + 1 > map_size - (finish.node - map)) {  // 未用的 node 太少
        reallocate_map(nodes_to_add, false);
    }
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::reallocate_map(size_type nodes_to_add, bool add_at_front) {
    size_type old_num_nodes = finish.node - start.node + 1;
    size_type new_num_nodes = old_num_nodes + nodes_to_add;

    map_pointer new_nstart;
    if (map_size > 2 * new_num_nodes) {
        new_nstart = map + (map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
        if (new_nstart < start.node) {
            copy(start.node, finish.node + 1, new_nstart);
        }
        else {
            copy_backend(start.node, finish.node + 1, new_nstart + old_num_nodes);
        }
    }
    else {
        size_type new_map_size = map_size + max(map_size, nodes_to_add) + 2;
        map_pointer new_map = map_allocator::allocate(new_map_size);
        new_nstart = new_map + (new_map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
        copy(start.node, finish.node + 1, new_nstart);
        map_allocator::deallocate(map, map_size);
        map = new_map;
        map_size = new_map_size;
    }
    start.set_node(new_nstart);
    finish.set_node(new_nstart + old_num_nodes - 1);
}
```
