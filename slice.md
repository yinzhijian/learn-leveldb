# Slice分析

## 功能

slice是一个简单的数据结构，存储着指向外部数据的指针以及数据大小。需要确保对应的外部数据回收后不再使用slice。

同时提供了数据大小的比较，以及前缀包含的功能。因此，

## 设计实现

两个私有变量，一个是指向外部数据的const char*指针，指向常量，说明不能修改指向的数据，另一个是大小。

```cpp
 private:
  const char* data_;
  size_t size_;
```

定义了4种构造函数

```cpp
  // Create an empty slice.
  Slice() : data_(""), size_(0) { }
  // Create a slice that refers to d[0,n-1].
  Slice(const char* d, size_t n) : data_(d), size_(n) { }
  // Create a slice that refers to the contents of "s"
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) { }
  // Create a slice that refers to s[0,strlen(s)-1]
  Slice(const char* s) : data_(s), size_(strlen(s)) { }
```

数据访问方面，重写了[]符号，可以访问指定index的数据

```cpp
  // Return the ith byte in the referenced data.
  // REQUIRES: n < size()
  char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
  }
```

slice判断是否相等，大小方面实现了3个方法

```cpp
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

inline bool operator!=(const Slice& x, const Slice& y) {
  return !(x == y);
}

inline int Slice::compare(const Slice& b) const {
  // 比较两个二进制数据的大小，先比较两个最小长度的数据的大小通过memcmp，如果这一步就不相等，可以直接返回；如果相等，再比较长度，长的那个大，如果长度相等，则说明相等。
  const size_t min_len = (size_ < b.size_) ? size_ : b.size_;
  int r = memcmp(data_, b.data_, min_len);
  if (r == 0) {
    if (size_ < b.size_) r = -1;
    else if (size_ > b.size_) r = +1;
  }
  return r;
}
```

最后还有一个判断是否以slice的开头

```cpp
  // Return true iff "x" is a prefix of "*this"
  bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) &&
            (memcmp(data_, x.data_, x.size_) == 0));
  }
```

