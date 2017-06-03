

# SkipList分析

## 功能

组织数据，方便查找

## 实现方式

### skiplist数据结构实现

```cpp
template<typename Key, class Comparator> class SkipList{
  class Iterator;
  template<typename Key, class Comparator> struct Node;
  private:
  enum { kMaxHeight = 12 };
  // Immutable after construction
  Comparator const compare_;
  Arena* const arena_;    // Arena used for allocations of nodes
  Node* const head_;
  port::AtomicPointer max_height_;   // Height of the entire list
  Random rnd_;
}
```

SkipList的构造函数，头部指针head_被赋予一个新节点，其高度为最高。max_height初始为1。

```cpp
template<typename Key, class Comparator>
SkipList<Key,Comparator>::SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(reinterpret_cast<void*>(1)),
      rnd_(0xdeadbeef) {
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, NULL);
  }
}
```



如何创建新节点呢，通过arena分配指定数量的内存，并将该内存通过new (mem) Node(key)形式返回。由于高度不确定，需要动态分配高度数组所占用空间，通过将数组放在class末尾，并计算新增的数组所占用大小来分配内存。

```cpp
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::NewNode(const Key& key, int height) {
  char* mem = arena_->AllocateAligned(
      sizeof(Node) + sizeof(port::AtomicPointer) * (height - 1));
  return new (mem) Node(key);
}
```

下面讲下Node的实现，Node维护了所存储的key以及自身的高度所对应的指针，对外提供了设置及获取各高度值的方法。

```cpp
template<typename Key, class Comparator> struct SkipList<Key,Comparator>::Node {
  explicit Node(const Key& k) : key(k) { }
  Key const key;
  // Accessors/mutators for links.  Wrapped in methods so we can add the appropriate barriers as necessary.
  Node* Next(int n) {
    assert(n >= 0);
    return reinterpret_cast<Node*>(next_[n].Acquire_Load());
  }
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this pointer observes a fully initialized version of the inserted node.
    next_[n].Release_Store(x);
  }
  // 省略NoBarrier
  private:
  // Array of length equal to the node height.  next_[0] is lowest level link.
  port::AtomicPointer next_[1];
}
```

下面再看SkipList如何插入新节点

```cpp
template<typename Key, class Comparator> void SkipList<Key,Comparator>::Insert(const Key& key) {
  Node* prev[kMaxHeight];
  // 通过FindGreaterOrEqual找到该key所对应的node，以及该node每个level之前的node。
  Node* x = FindGreaterOrEqual(key, prev);
  // Our data structure does not allow duplicate insertion
  assert(x == NULL || !Equal(key, x->key));
  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    //在改变max_height时没有使用同步操作，是因为reader在读取新height后，head_指针的新level指向的是NULL，reader直接到下一level。如果是在head_的新level指向新node时，则reader可以使用该新node。
    max_height_.NoBarrier_Store(reinterpret_cast<void*>(height));
  }
  // 生成一个新节点
  x = NewNode(key, height);
  // 修改新节点所有level的next指针，以及对应level前一个元素的指针。
  for (int i = 0; i < height; i++) {
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}
```

插入新node用到了FindGreaterOrEqual，找到对应node以及该node在每个level的前一个node。分析下具体代码

```cpp
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* SkipList<Key,Comparator>::FindGreaterOrEqual(const Key& key, Node** prev)
    const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    //如果下一个节点的key值小于指定的key，则继续搜索
    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      x = next;
    } else {
      // 将当前节点记录到prev当前level中。如果已经是最底层则返回下一个节点。否则继续搜索。
      if (prev != NULL) prev[level] = x;
      if (level == 0) {
        return next;
      } else {
        // Switch to next list
        level--;
      }
    }
  }
}
```

下面我们继续看下如何随机高度的。比较简单就是4分之一的几率高度加一，且高度有最大值12

```cpp
template<typename Key, class Comparator>
int SkipList<Key,Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  //四分之一的机会高度加一
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

找到小于指定key的最近节点，逻辑是从当前最高level，一直next直到碰到比指定key大的node，然后降level继续寻找，直到level 0，返回当前node。

```cpp
// Return the latest node with a key < key.
// Return head_ if there is no such node.
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::FindLessThan(const Key& key) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    assert(x == head_ || compare_(x->key, key) < 0);
    Node* next = x->Next(level);
    if (next == NULL || compare_(next->key, key) >= 0) {
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}
```

寻找最后一个节点，逻辑跟查找一样，通过从最高level一直next，如果碰到NULL，就level -- ，直到到达level 0。

```cpp
// Return the last node in the list.
// Return head_ if list is empty.
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* SkipList<Key,Comparator>::FindLast()
    const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (next == NULL) {
      if (level == 0) {
        return x;
      } else {
        // Switch to next list
        level--;
      }
    } else {
      x = next;
    }
  }
}
```

如何得到skiplist是否包含某个key呢，通过contains实现，逻辑调用FindGreaterOrEqual，如果找到对应的x，且key值一致，则为返回true，否则false。

```cpp
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, NULL);
  if (x != NULL && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

### iterator实现

skiplist本身对外只提供了contains一个查找方式，为了让外部不对内部的存储结构有耦合，在内部实现了一个iterator访问器，屏蔽了skiplist的存储方式。

**iterator**内部维护了两个变量，以支持迭代操作

```cpp
const SkipList* list_;//由构造函数赋值
Node* node_;//指向当前的位置。
```

通过next和prev移动当前指针，并通过key()获取当前指针所指向的值。key next prev都需要在当前指针有效的情况下才能继续操作。**由于prev需要重新搜索定位当前key的上一个node，因此效率低，可能是因为leveldb没有大量prev的操作**

```cpp
template<typename Key, class Comparator>
inline bool SkipList<Key,Comparator>::Iterator::Valid() const {
  return node_ != NULL;
}

template<typename Key, class Comparator>
inline const Key& SkipList<Key,Comparator>::Iterator::key() const {
  assert(Valid());
  return node_->key;
}

template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Next() {
  assert(Valid());
  node_ = node_->Next(0);
}

template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Prev() {
  // Instead of using explicit "prev" links, we just search for the
  // last node that falls before key.
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = NULL;
  }
}
```

初始迭代时，需要先定位当前指针，提供了3个方法，其中两个都是使用的skiplist的内部方法，iterator仅仅是封装了下。

```cpp
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, NULL);
}

template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::SeekToFirst() {
  node_ = list_->head_->Next(0);
}

template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::SeekToLast() {
  node_ = list_->FindLast();
  if (node_ == list_->head_) {
    node_ = NULL;
  }
}
```

### 技巧

1. 通过设置一个大小1的数组，再根据实际大小动态分配更多的内存，从而实现动态数组。

2. explicit 只对构造函数起作用，用来抑制隐式转换。比如

   1. ```cpp
      class Iterator{
      explicit Iterator(const SkipList* list);
      }
      // 构造函数加上explicit后，Iterator iter = listptr;编译就会报错，
      // 隐式转换后等价于Iterator iter = Iterator(listptr);
      ```

3. 类尽量只做自己职责，如果要提供更高级的方法，可以通过组合的方式实现，如iterator包含skiplist。