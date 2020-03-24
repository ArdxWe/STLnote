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
```