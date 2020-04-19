# 关联式容器

二叉搜索树 平衡树 红黑树的意义如果你不懂, 请 [Google](http://www.google.com/) 之后再看下文.

## RB-tree

### 节点设计

```cpp
typedef bool __rb_tree_color_type;
const __rb_tree_color_type __rb_tree_red = false;  // red 为 false
const __rb_tree_color_type __rb_tree_black = true;  // black 为 true

struct __rb_tree_node_base {
    typedef __rb_tree_color_type color_type;
    typedef __rb_tree_node_base* base_ptr;  // 指向自身的指针

    color_type color; // 颜色 red or black
    base_ptr parent;  // 父指针
    base_ptr left;  // 左指针
    base_ptr right;  // 右指针

    // 二叉搜索树的性质
    // min
    static base_ptr minimum(base_ptr x) {
        while(x->left != 0) x = x->left;
        return x;
    }

    // max
    static base_ptr maximum(base_ptr x) {
        while(x->right != 0) x = x->right;
        return x;
    }
};

template <class Value>
struct  __rb_tree_node: public __rb_tree_node_base  // 继承
{
    typedef __rb_tree_node<Value>* link_type;
    Value value_field;  // 节点值
};
```

### 迭代器

```cpp
// 基层迭代器
struct __rb_tree_base_iterator {
    typedef __rb_tree_node_base::base_ptr base_ptr;
    typedef bidirectional_iterator_tag iterator_category;
    typedef ptrdiff_t difference_type;

    base_ptr node;  // 基层节点指针

    // ++
    void increment() {
        if (node->right != 0) {  // 如果当前节点有右子节点 答案是右子节点的最左节点
            node = node->right;
            while(node->left != 0) {
                node = node->left;
            }
        }
        else {  // 没有右子节点
            base_ptr y = node->parent;  // 父节点
            while (node == y->right) {  // 如果是父节点的右儿子 一直往上
                node = y;
                y = y->parent;
            }
            if (node->right != y) {  // node 不为 header
                node = y;
            }
        }
    }

    // --
    void decrement() {
        if (node->color == __rb_tree_red && node->parent->parent == node) {  // node 为 header
            node = node->right;
        }
        else if (node->left != 0) {
            base_ptr y = node->left;
            while (y->right != 0) {
                y = y->right;
            }
            node = y;
        }
        else {
            base_ptr y = node->parent;
            while (node == y->left) {
                node = y;
                y = y->parent;
            }
            node = y;
        }
    }
};
```

```cpp
// 正规迭代器
template <class Value, class Ref, class Ptr>
struct __rb_tree_iterator : public __rb_tree_base_iterator {
    typedef Value value_type;
    typedef Ref reference;
    typedef Ptr pointer;
    typedef __rb_tree_iterator<Value, Value&, Value*> iterator;
    typedef __rb_tree_iterator<Value, const Value&, const Value*> const_iterator;
    typedef __rb_tree_iterator<Value, Ref, Ptr> self;
    typedef __rb_tree_node<Value>* link_type;

    __rb_tree_iterator() {}
    __rb_tree_iterator(link_type x) {
        node = x;
    }
    __rb_tree_iterator(const_iterator& it) {
        node = it.node;
    }

    reference operator*() const {  // 迭代器行为应该像一个指针 解除引用 应该是得到得到节点值
        return link_type(node)->value_field;
    }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
    pointer operator->() const {  // 应该是对节点数据的操作
        return &(operator*());
    }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

    // ++i
    self& operator++() {
        increment();
        return *this;
    }
    // i++
    self operator++(int) {
        self tmp = *this;
        increment();
        return tmp;
    }
    // --i
    self& operator--() {
        decrement();
        return *this;
    }
    // i--
    self operator--(int) {
        self tmp = *this;
        decrement();
        return tmp;
    }
};

```

### `RB-tree` 的数据结构

```cpp
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
protected:
    typedef void* void_pointer;
    typedef __rb_tree_node_base* base_ptr;
    typedef __rb_tree_node<Value> rb_tree_node;
    typedef simple_alloc<rb_tree_node, Alloc> rb_tree_node_allocator;
    typedef __rb_tree_color_type color_type;

public:
    typedef Key key_type;
    typedef Value value_type;
    typedef value_type* pointer;
    typedef const value_type* const_pointer;
    typedef value_type& reference;
    typedef const value_type& const_reference;
    typedef rb_tree_node* link_type;
    typedef size_t size_type;
    typedef ptrdiff_t differnece_type;

protected:
    link_type get_node() {
        return rb_tree_node_allocator::allocate();
    }
    void put_node(link_type p) {
        rb_tree_node_allocator::deallocate(p);
    }

    link_type create_node(const value_type& x) {
        link_type tmp = get_node();  // 配置空间
        __STL_TRY {
            construct(&tmp->value_field, x);  // 构造内容
        }
        __STL_UNWIND(put_node(temp));
        return tmp;  // 返回指针
    }

    link_type clone_node(link_type x) {
        link_type tmp = create_node(x->value_field);
        tmp->color = x->color;
        tmp->left = 0;
        tmp->right = 0;  // clone 为什么没有 tmp->parent ?
        return tmp;
    }

    void destroy_node(link_type p) {
        destroy(&p->value_field);  // 析构
        put_node(p);  // 回收
    }

protected:
    // 内部维护三个数据
    size_type node_count;  // 实际节点个数
    link_type header;  // 头指针 实际不存数据 左指向最小 右指向最大 parent 指向根节点
    Compare key_compare;  // 比较

    // 方便取得 header 成员的三个函数
    // 需要强制转换为引用类型吗 一般都是返回实际对象 参数加个引用就好
    // 类似于 return (link_type)header->parent
    link_type& root() const {
        return (link_type&)header->parent;  // header 的父节点指向实际 rb-tree 的根节点
    }

    link_type& leftmost() const {
        return (link_type&)header->left;  // header 左指向最小节点
    }

    link_type& rightmost() const {
        return (link_type&)header->right;  // header 右边部分指向最大节点
    }

    // 以下六个函数方便取得 header 成员
    // 强制转换为引用总觉得很变扭
    static link_type& left(link_type x) {
        return (link_type&)(x->left);
    }

    static link_type& right(link_type x) {
        return (link_type&)(x->right);
    }

    static link_type& parent(link_type x) {
        return (link_type&)(x->parent);
    }

    static reference value(link_type x) {
        return (link_type&)(x->value_field);
    }

    static const Key& key(link_type x) {
        return KeyOfValue()(value(x));  // 这是什么语法...
    }

    static color_type& color(link_type x) {
        return (color_type&)(x->color);
    }

    // 以下六个函数方便取得 header 成员
    // 针对 base_ptr
    static link_type& left(base_ptr x) {
        return (link_type&)(x->left);
    }

    static link_type& right(base_ptr x) {
        return (link_type&)(x->right);
    }

    static link_type& parent(base_ptr x) {
        return (link_type&)(x->parent);
    }

    static reference value(base_ptr x) {
        return ((link_type)x)->value_field;
    }

    static const Key& key(base_ptr x) {
        return KeyOfValue()(value((link_type(x)));  // 这是什么语法...
    }

    static color_type& color(base_ptr x) {
        return (color_type&)(link_type(x)->color);
    }

    // 求极大值 极小值
    static link_type minimum(link_type x) {
        return (link_type) __rb_tree_node_base::minimum(x);
    }

    static link_type maximum(link_type x) {
        return (link_type) __rb_tree_node_base::maximum(x);
    }

public:
    typedef __rb_tree_iterator<value_type, reference, pointer> iterator;  // 迭代器

private:
    iterator __insert(base_ptr x, base_ptr y, const value_type& v);
    link_type __copy(link_type x, link_type p);
    void __erase(link_type x);
    void init() {  // 创建 header
        header = get_node();
        color(header) = __rb_tree_red;  // red

        root() = 0;  // 父节点
        leftmost() = header;  // 左
        rightmost() = header;  // 右
    }

public:
    rb_tree(const Compare& comp = Compare()) : node_count(0), key_compare(comp) {  // 初始化三个数据 可以指定比较
        init();  // 初始化 header
    }

    ~rb_tree() {
        clear();
        put_node(header);
    }

    rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& operator=(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x);

    Compare key_comp() const {  // 当前对象的比较
        return key_compare;
    }

    iterator begin() {
        return leftmost();  // 最小节点
    }

    iterator end() {
        return rightmost();  // 最大节点
    }

    bool empty() {
        return node_count == 0;
    }

    size_type size() const {
        return node_count;
    }

    size_type max_size() const {  // ffffffff...
        return size_type(-1);
    }

    pair<iterator, bool> insert_unique(const value_type& x);
    iterator insert_equal(const value_type& x);
    ...
};
```

### 元素操作

`RB-tree` 提供两种插入操作:

* `insert_unique()` : `key` 独一无二
* `insert_equal()` : `key` 可以重复

```cpp
// insert_equal
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::insert_equal(const Value& v) {
    link_type y = header;  // header
    link_type x = root(); // 根节点
    while (x != 0) {
        y = x;
        x = key_compare(KeyOfValue()(v), key(x)) ? left(x) : right(x);  // 大往左 小往右
    }
    return __insert(x, y, v);  // 插入位置 父节点 值
}

// insert_unique
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
pair<typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator, bool> rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::insert_unique(const Value& v) {
    link_type y = header;  // header
    link_type x = root(); // 根节点
    bool comp = true;
    while(x != 0) {
        y = x;
        comp = key_compare(KeyOfValue()(v), key(x));
        x = comp ? left(x) : right(x);  // 大往左 小往右
    }
    iterator j = iterator(y);  // 插入节点父节点的迭代器
    if (comp) {  // 应插入左侧
        if (j == begin()) {  // 父节点为最左节点 比最小的小 直接插入即可
            return pair<iterator, bool>(__insert(x, y, v), true);
        }
        else {
            --j;  // 得到最近的比 v 小的迭代器
        }
    }
    if (key_compare(key(j.node), KeyOfValue()(v))) {  // 不存在相等
        return pair<iterator, bool>(__insert(x, y, v), true);
    }
    return pair<iterator, bool>(j, false);  // 相等 返回相等的节点对应的迭代器 和 false
}

// __insert 实际插入
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::__insert(base_ptr x_, base_ptr y_, const Value& v) {
    link_type x = link_type(x_);  // 强转为子类型指针
    link_type y = link_type(y_);
    link_type z;

    if (y == header || x != 0 || key_compare(KeyOfValue()(v), key(y))) {
        z = create_node(v);
        left(y) = z;  // 插入 y 左侧 改变 y->left
        if (y  == header) {
            root() = z;
            rightmost() = z;
        }
        elif (y == leftmost()) {
            leftmost() = z;
        }
    }
    else {  // 插入右侧
        z = create_node(v);
        right(y) = z;  // 插入 y 右侧 更改 y->right
        if (y == rightmost()) {  // 如果 y 本身为最右 更改最右
            rightmost()  = z;
        }
    }
    // 更新新节点的 左 右 父亲指针
    parent(z) = y;
    left(z) = 0;  // 一定是叶子节点 所以左右都为 nullptr
    right(z) = 0;

    __rb_tree_rebalance(z, header->parent);  // 调整平衡 参数为 新增节点 根节点
    ++node_count;
    return iterator(z);  // 返回新增节点的迭代器
}
```

调整 `RB-tree` 旋转和变色

```cpp
// 插入节点后令树平衡
inline void __rb_tree_rebalance(__rb_tree_node_base* x, __rb_tree_node_base*& root) {  // 新增节点 树的根节点
    x->color = __rb_tree_red;  // 新节点为红 保证性质1: 任意一结点到每个叶子结点的路径都包含数量相同的黑结点
    while (x != root && x->parent->color == __rb_tree_red) {  // 父节点为红 违反性质5: 红节点子节点必为黑色
        if (x->parent == x->parent->parent->left) {  // 父节点为祖父节点左子节点
            __rb_tree_node_base* y = x->parent->parent->right;  // y 为伯父节点
            // 四种情况之一: 父节点为红 父节点为祖父节点的左子节点 伯父节点为红
            if (y && y->color == __rb_tree_red) {  // 伯父节点为红
                x->parent->color = __rb_tree_black;  // 父节点先改为黑
                y->color = __rb_tree_black;  // 伯父节点改为黑色
                x->parent->parent->color = __rb_tree_red;   // 祖父节点改为红
                x = x->parent->parent;  // 显然若祖父节点由黑改为红 祖父节点的父节点若为红则会违反性质5: 红节点子节点必为黑色 所以视祖父节点为新增节点 继续 while 循环
            }
            // 四种情况之二: 父节点为红 父节点为祖父节点的左子节点 伯父节点为黑 ( null 视为黑色)
            else {
                if (x == x->parent->right) {
                    x = x->parent;
                    __rb_tree_rotate_left(x, root);  // 左旋 参数: 左旋点 根节点
                }
                x->parent->color = __rb_tree_black;  // x 父节点改为红色
                x->parent->parent->color = __rb_tree_red;  // x 祖父节点改为红色
                __rb_tree_rotate_right(x->parent->parent, root);  // 右旋 参数 右旋点 根节点
            }
        }
        else {  // 父节点为祖父节点右子节点
            __rb_tree_node_base* y = x->parent->parent->left;  // 伯父节点
            // 四种情况之三: 父节点为红 父节点为祖父节点的右子节点 伯父节点为红
            if (y && y->color == __rb_tree_red) {
                x->parent->color = __rb_tree_black;
                y->color = __rb_tree_black;
                x->parent->parent->color = __rb_tree_red;
                x = x->parent->parent;  // 往上走 继续 while
            }
            // 四种情况之四: 父节点为红 父节点为祖父节点的右子节点 伯父节点为黑
            else {
                if (x == x->parent->left) {
                    x = x->parent;
                    __rb_tree_rotate_right(x, root);
                }
                x->parent->color = __rb_tree_black;
                x->parent->parent->color = __rb_tree_red;
                __rb_tree_rotate_left(x->parent->parent, root);
            }
        }
    }  // end while
    root->color = __rb_tree_black;  // 置根节点为黑 保持性质2: 根节点为黑色
}
```

左旋和右旋函数:

```cpp
// 左旋
inline void __rb_tree_rotate_left(__rb_tree_node_base* x, __rb_tree_node_base*& root) {  // 左旋节点 根节点
    __rb_tree_node_base* y = x->right;  // y 为左旋点的右子节点
    x->right = y->left;  // 先接上
    if (y->left != 0) {
        y->left->parent = x;  // 设定父节点
    }
    y->parent = x->parent;  // 新的顶点接管 x 的父节点

    if (x == root) {  // 若 x 为根节点 更新 root 注意: 参数 root 为引用
        root = y;
    }
    else if (x == x->parent->left) {  // x 父节点接管 y
        x->parent->left = y;
    }
    else {
        x->parent->right = y;
    }
    // 非常自然
    y->left = x;
    x->parent = y;
}

// 右旋
// 类比左旋 不再注释
inline void __rb_tree_rotate_right(__rb_tree_node_base* x, __rb_tree_node_base* root) {
    __rb_tree_node_base* y = x->left;
    x->left = y->right;
    if (y->right != 0) {
        y->right->parent = x;
    }
    y->parent = x->parent;

    if (x == root) {
        root = y;
    }
    else if (x == x->parent->right) {
        x->parent->right = y;
    }
    else {
        x->parent->left = y;
    }
    x->right = x;
    x->parent = y;
}
```

## `set`

所有元素会根据键值自动排序, 所以 `set` 的迭代器不可以改变 `set` 的元素值, 是一种 `constant iterators` .

标准 `set` 以 `RB-tree` 为底层机制, 几乎所有的 `set` 操作, 都转为 `RB-tree` 的操作. 看源代码:

```cpp
class set {
public:
    // typedefs
    typedef Key key_type;
    typedef Key value_type;
    // key_compare 与 value_compare 使用同一个比较函数
    typedef Compare key_compare;
    typedef Compare value_compare;

private:
    template <class T>
    struct identity : public unary_function<T, T>  
    {
        const T& operator()(const T& x) const {return x;}
    };

    typedef rb_tree<key_type, value_type, identity<value_type>, key_compare, Alloc> rep_type;
    rep_type t;  // 用 rb-tree 表现 set

public:
    // 直接使用 rb-tree
    typedef typename rep_type::const_pointer pointer;  // set 不可更改
    typedef typename rep_type::const_pointer const_pointer;
    typedef typename rep_type::const_reference reference;  // set 不可更改
    typedef typename rep_type::const_reference const_reference;
    typedef typename rep_type::const_iterator iterator;  // set 不可更改
    typedef typename rep_type::const_iterator const_iterator;
    typedef typename rep_type::const_reverse_iterator reverse_iterator;  // set 不可更改
    typedef typename rep_type::const_reverse_iterator const_reverse_iterator;
    typedef typename rep_type::size_type size_type;
    typedef typename rep_type::difference_type difference_type;

    // 默认构造函数 直接构造 t
    set() : t(Compare()) {}

    // 指定比较函数的情况
    explicit set(const Compare& comp) : t(comp) {}

    // 指定首尾迭代器
    template <class InputIterator>
    set(InputIterator first, InputIterator last) : t(Compare()) {t.insert_unique(first, last)}  // set 不允许相同键值

    // 指定首尾迭代器和比较函数
    template <class InputIterator>
    set(InputIterator first, InputIterator last， const Compare& comp) : t(comp) {t.insert_unique(first, last)}  // set 不允许相同键值

    // 拷贝构造
    set(const set<Key, Compare, Alloc>& x) : t(x.t) {}

    // 赋值
    set<Key, Compare, Alloc>& operator=(const set<Key, Compare, Alloc>& x) {
        t = x.t;
        return *this;
    }

    // 以下 set 仅传递调用 rb-tree 已经提供
    key_compare key_comp() const {return t.key_comp(); }
    value_compare value_comp() const {return t.key_comp(); }
    iterator begin() const {return t.begin(); }
    iterator end() const {return t.end(); }
    reverse_iterator rbegin const {return t.rbigin(); }
    reverse_iterator rend() const {return t.rend(); }
    bool empty() const {return t.empty(); }
    size_type size() const {return t.size(); }
    size_type max_size() const {return t.max_size(); }
    void swap(set<Key, Compare, Alloc>& x) {t.swap(x.t); }

    // insert erase
    // 参数 : 值
    typedef pair<iterator, bool> pair_iterator_bool;
    pair<iterator, bool> insert(const value_type& x) {
        pair<typename, rep_type::iterator, bool> p = t.insert_unique(x);
        return pair<iterator, bool>(p.first, p.second);
    }

    // 参数 : 位置 值
    iterator insert(iterator positon, const value_type& x) {
        typedef typename rep_type::iterator rep_iterator;
        return t.insert_unique((rep_iterator&)positon, x);
    }

    // 参数 : 首尾迭代器
    template <class InputIterator>
    void insert(iterator first, iterator last) {
        t.insert_unique(first, last);
    }

    void erase(iterator position) {
        typedef typename rep_type::iterator rep_iterator;
        t.erase((rep_iterator&)position);
    }

    size_type erase(const key_type& x) {
        return t.erase(x);
    }

    void erase(iterator first, iterator last) {
        typedef typename rep_type::iterator rep_iteraotor;
        t.erase((rep_iteraotor&)first, (rep_iteraotor&)last);
    }

    void clear() {t.clear(); }

    // 操作
    iterator find(const key_type& x) const {return t.find(x); }
    size_type count(const key_type& x) const {return t.count(x); }
    iterator lower_bound(const key_type& x) const {
        return t.lower_bound(x);
    }
    iterator upper_bound(const key_type& x) const {
        return t.upper_bound(x);
    }
    pair<iterator, iterator> equal_range(const key_type& x) const {
        return t.equal_range(x);
    }

    friend bool operator== __STL_NULL_TMPL_ARGS (const set&, const set&);
    friend bool operator< __STL_NULL_TMPL_ARGS (const set&, const set&);
};

template <class Key, class Compare, class Alloc>
inline bool operator==(const set<Key, Compare, Alloc>& x, const set<Key, Compare, Alloc>& y) {
    return x.t == y.t;
}

template <class Key, class Compare, class Alloc>
inline bool operator<(const set<Key, Compare, Alloc>& x, const set<Key, Compare, Alloc>& y) {
    return x.t < y.t;
}

```
