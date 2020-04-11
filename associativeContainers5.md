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
