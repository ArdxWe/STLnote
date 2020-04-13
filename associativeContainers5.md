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
