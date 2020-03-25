# 迭代器 `iterators`

## 迭代器设计思维

`STL` 中心思想: 将数据容器和算法分开, 彼此独立设计, 然后用胶着剂撮合在一起.

一个简单的例子.

```cpp
// find() 接受两个迭代器和一个目标
template <class InputIterator, class T>
InputIterator find(InputIterator first, InputIterator last, const T& value) {
    while (first != last && *first != value){
        first++;
    }
    return first;
}
```

```cpp
// find() 可以对不同的容器进行查找
const int arraySize = 7;
int ia[arraySize] = {0, 1, 2, 3, 4, 5, 6 };

vector<int> ivect(ia, ia+arraySize);
list<int> ilist(ia, ia+arraySize);
deque<int> ideque(is, ia+arraySize);

// 不同容器对应不同迭代器
vector<int>::iterator it1 = find(ivect.begin(), ivect.end(), 4);
list<int>::iterator it2 = find(ilist.begin(), ilist.end(), 4);
vector<int>::iterator it3 = find(ideque.begin(), ideque.end(), 4);
```

上述例子迭代器依附在容器之下, 我们需要设计独立而又泛用的迭代器.

## 迭代器是一种 `smart pointer`

迭代器类似指针, 因此迭代器最重要的工作是对 `operator*` 和 `operator->` 重载.

现在我们为 `list` 设计一个迭代器, 核心思想是包装指针.

```cpp
// file: 3my_list.h
template <typename T>
class List {
    // 包装函数
    void insert_front(T value);
    void insert_end(T value);
    void display(std::ostream &os = std::out) const;
    // ...

    private:
    // 私有成员
    ListItem<T>* _end;
    ListItem<T>* _front;
    long _size;
};

// 实际单向链表
template <typename T>
class ListItem {
    public:

    T value() const {
        return value;
    }

    ListItem* next() const {
        return _next;
    }
    ...

    private:

    T _value;
    ListItem* _next;
}
```

我们设计的迭代器 `Iterator` 进行 `*Iterator` 时应该得到 `ListItem` 对象, 为了广泛应用, 我们设计为 `class template` .

```cpp
// file: 3mylist-iter.h
# include "3mylist.h"

template <class Item>
struct ListIter {
    Item* ptr;

    ListIter (Item* p = 0) : ptr(p) {};

    Item& operator*() const {
        return *ptr;
    }

    Item* operator->() const {
        return ptr;
    }

    // ++i
    ListIter& operator++() {
        ptr = ptr->next();
        return *this;
    }

    // i++
    ListIter operator++(int) {
        ListIter tmp = *this;
        ++(*this);
        return tmp;
    }

    // ==
    bool operator==(const ListIter& i) const {
        return ptr == i.ptr;
    }

    // !=
    bool operator!=(const ListIter& i) const {
        return ptr != i.ptr;
    }
};
```

用迭代器 `ListIter` 将容器 `List` 和算法 `find()` 结合:

```cpp
List<int> mylist;

mylist.insert_front(1);
mylist.insert_end(2);
mylist.display(); // (1, 2)

// ListIter 是容器 listItem<int> 装的数据
ListIter<ListItem<int>> begin(mylist.front());
ListIter<ListItem<int>> end;
ListIter<ListItem<int>> iter = find(begin, end, 2);
```

这里存在的问题是 `find()` 函数比较的时候用的是 `*iter != value` , 而 `value` 是 `int` , `*iter` 类型是 `ListItem<int>`, 所以需要为 `ListItem` 编写 `operator!=` 重载函数.

```cpp
template <typename T>
bool operator!=(const ListItem<T>& item, T n) {
    return item.value() != n;
}
```

看到这里细心的读者可能会想: `ListItem<T>` 也没有 `value()` 函数呀, 这里主要介绍思想, 你只需要明白这些函数的意义就好 `:)`

显然, 为了设计迭代器, 我们暴露了太多 `List` 的实现细节. 也就是说, 迭代器的设计不可避免地需要了解相应的容器或者说数据. 所以每一个 `STL` 容器都有它专属的迭代器.

## 迭代器相应型别

迭代器所指之物就是一种型别, 在使用迭代器时我们必然会用到, 这里可以使用函数模板的参数推导机制:

```cpp
template <class I, class T>
void func_impl(I iter, T t);  // T 即是所指之物型别

template <class I>
void func(I iter) {
    func_impl(iter, *iter);  // 将 *iter 做为实参
}
```

然而迭代器相应型别有五种, 需要一些更全面的方法.

## `traits` 技法

`value_type` 用于返回值上述方法就失效了, 很直观的想法是声明内嵌型别:

```cpp
template <class T>
struct MyIter {
    typedef T value_type;  // 内嵌型别声明
    ...
}

template <class I>
typename I::value_type func(I ite) {  // 注意返回值类型
    return *ite;
}
```

这种做法也存在问题: 原生指针不是 `class type` , 但 `STL` 必然需要接受原生指针作为迭代器, 可以使用 `template partial specialization` .

```cpp
template <class I>
typename iterator_traits<I>::value_type  // 返回值类型 可以对此进行偏特化
func(I ite) {
    return *ite;
}
```

精彩的来了:

```cpp
// 迭代器特性
template <class I>
struct iterator_traits {
    typedef typename I::value_type value_type;
}

// 特化版 原生指针
template <class T>
struct iterator_traits<T*> {  // 迭代器是原生指针 T*
    typedef T value_type;
}

// 特化版 const 指针
// 这里我们的目的是得到指针所指对象的类型 所以应当去掉 const
template <class T>
struct iterator_traits<const T*> {  // 迭代器是原生指针 T*
    typedef T value_type;
}
```

我们形象的称 `iterator_traits` 为特性萃取机.

迭代器 `5` 种相应型别都加入萃取机:

```cpp
template <class I>
struct iterator_traits {
    typedef typename I::iterator_category iterator_category;
    typedef typename I::value_type value_type;
    typedef typename I::difference_type difference_type;
    typedef typename I::pointer pointer;
    typedef typename I::reference reference;
}
```

如上所述, 会为 `pointer` 和 `pointer-to-const` 添加特化版本.

### `difference_type`

表示两个迭代器之间的距离. 例如 `STL` 中的 `count()` .

```cpp
template <class I, class T>
typename iterator_traits<I>::difference_type  // 返回值类型
count (I first, I last, const T& value) {
    typename iterator_traits<I>::difference_type n = 0;
    for ( ; first!=last; ++first) {
        if (*first == value) {
            n++;
        }
    }
    return n;
}

// 泛化版
template <class I>
struct iterator_traits {
    typedef typename I::different_type difference_type;
}

// 特化版本 原生指针 直接返回 ptrdiff_t
template <class T>
struct iterator_traits<T*> {
    typedef ptrdiff_t difference_type;
}

// 特化版本 const 指针 直接返回 ptrdiff_t
template <class T>
struct iterator_traits<const T*> {
    typedef ptrdiff_t difference_type;
}
```

### `reference type`

迭代器分为两种:

* 不允许改变所指对象之内容 `constant iterators` , 例如 `const int* pic` ;

* 允许改变所指对象之内容 `mutable iterators` , 例如 `int* pi` ;

迭代器 `p` `value type` 为 `T` , 当为 `mutable iterators` 时, `*p` 应为 `T&` , 当为 `constant iterators` 时, `*p` 应为 `const T&`. 显然都应该为左值! 实现见下节.

### `pointer type`

显然指的是 迭代器所指之物的指针.看代码, 包含上一章节的 `reference type`.

```cpp
template <class I>
struct iterator_traits {
    ...
    typedef typename I::pointer pointer;
    typedef typename I::reference reference;
};

// 偏特化版 原生指针
template <class T>
struct iterator_traits<T*> {
    ...
    typedef T* pointer;
    typedef T& reference;
};

// 偏特化版 const 指针
template <class T>
struct iterator_traits<const T*> {
    ...
    typedef const T* pointer;
    typedef const T& reference;
};
```

### `iterator_category`

迭代器被分为 `5` 类:

* `Input Iterator` : `read only`

* `Output Iterator` : `write only`

* `Forward Iterator` : `read and write`

* `Bidirectional Iterator` : `read and write` , 双向移动

* `Random Access Iterator` : `read and write` , 随意移动

我们设计的算法应该对每个迭代器提供明确定义, 更强化的迭代器提供允许范围内的更加高效的定义.

举个例子, `advance()` 接受两个参数, 一个迭代器, 一个前近距离, 我们可以针对不同的迭代器设置不同的函数, 根据迭代器的类型来调用不同的函数, 在执行期间判断效率不高, 我们可以重载 `advance()` , 添加第三个参数迭代器类型. 看代码:

```cpp
// 标记型别
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};

// 重载 __advance()
template <class InputIterator, class Distance>
inline void __advance(InputIterator& i, Distance n, input_iterator_tag) {
    while (n--) {
        i++;
    }
}

// 因为是继承 所以 当传递一个 forwardIterator 会直接调用上面的函数 所以可以省略
template <class ForwardIterator, class Distance>
inline void __advance(ForwardIterator& i, Distance n, bidirectional_iterator_tag) {
    __advance(i, n, input_iterator_tag());
}

template <class BidiectionalIterator, class Distance>
inline void __advance(BidiectionalIterator& i, Distance n, bidirectional_iterator_tag) {
    if (n >= 0) {
        while (n--) ++i;
    }
    else {
        while (n++) --i;
    }
}

template <class RandomAccessIterator, class Distance>
inline void __advance(RandomAccessIterator& i, Distance n, random_access_iterator_tag) {
    i += n;
}
```

现在我们需要上层接口, 上层接口当然只需要前两个参数, 接口里面调用 `__advance()` , 其中第三个参数通过 `traits` 机制获得.

```cpp
// 是的 一切都是那么自然
template <class InputIterator, class Distance>
inline void advance(InputIterator& i, Distance n) {
    __advance(i, n, iterator_traits<InputIterator>::iterator_category());
}
```

然后在 `traits` 中加入相应型别:

```cpp
template <class I>
struct iterator_traits {
    typedef I::iterator_category iterator_category;
}

// 偏特化版 原生指针
// 原生指针显然属于 random_access_iterator_tag
template <class T>
struct iterator_traits<T*> {
    typedef random_access_iterator_tag iterator_category;
};

// 偏特化版 const 指针
// const 指针显然属于 random_access_iterator_tag
template <class T>
struct iterator_traits<const T*> {
    typedef random_access_iterator_tag iterator_category;
};
```

## `std::iterator` 的保证

任何迭代器都应提供上述五个内嵌相应型别让 `traits` 提取, 为了方便, `STL` 提供了一个 `iterators class` 供新迭代器的继承:

```cpp
template <class Category,
          class T,
          class Distance = ptrdiff_t,
          class Pointer = T*,
          class Reference = T&>
struct iterator {
    typedef Category iterator_category;
    typedef T value_type;
    typedef Distance difference_type;
    typedef Pointer pointer;
    typedef Reference reference;
};
```

后三个有默认值, 所以继承只需要提供前两个参数, 迭代器种类和值类型, 举个例子:

```cpp
template <class Item>
struct ListIter : public std::iterator<std::forward_iterator_tag, Item> {...}
```

`traits` 技法利用内嵌型别和编译器的参数推导, 增强了 `C++` 在参数推导方面的能力, 大量用于 'STL` 的实现.