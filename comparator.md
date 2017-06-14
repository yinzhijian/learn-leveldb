# Comparator分析

## 主要功能

用于保证在sstable和database中key的总顺序。定义了接口，用户可以选择自己实现，系统默认实现是BytewiseComparator，下面主要讲其实现。

## 具体实现

name 方法用来辨别db创建时使用的跟访问使用的是否是同一个comparator

```cpp
virtual const char* Name() const {
  return "leveldb.BytewiseComparator";
}
```

compare方法直接使用的slice的compare

```cpp
  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  virtual int Compare(const Slice& a, const Slice& b) const {
    return a.compare(b);
  }
```

FindShortestSeparator用来减少比如索引block等内部数据结构的空间使用

为什么要在未被包含的情况下将最后一个字节值+1呢？

```cpp
  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct. 
virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const {
    // Find length of common prefix
  	//找到start跟limit共有的长度
    size_t min_length = std::min(start->size(), limit.size());
    size_t diff_index = 0;
    while ((diff_index < min_length) &&
           ((*start)[diff_index] == limit[diff_index])) {
      diff_index++;
    }
    //如果共有的长度已经等于其中一个长度，则说明一个包含另一个，则不缩短
    if (diff_index >= min_length) {
      // Do not shorten if one string is a prefix of the other
    } else {
      // start的第一个不相同的字节值在小于0xff及+1也小于limit同位置字节值得情况下+1
      uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
      if (diff_byte < static_cast<uint8_t>(0xff) &&
          diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
        (*start)[diff_index]++;
        start->resize(diff_index + 1);
        assert(Compare(*start, limit) < 0);
      }
    }
```

FindShortSuccessor将key的最先小于0xff的值+1，并截断后面的数据。

```cpp
  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
virtual void FindShortSuccessor(std::string* key) const {
    // Find first character that can be incremented
    size_t n = key->size();
    for (size_t i = 0; i < n; i++) {
      const uint8_t byte = (*key)[i];
      if (byte != static_cast<uint8_t>(0xff)) {
        (*key)[i] = byte + 1;
        key->resize(i+1);
        return;
      }
    }
    // *key is a run of 0xffs.  Leave it alone.
  }
```

## 技巧

通过定义一个全局函数，返回仅实例化一次的全局实例

```cpp
static port::OnceType once = LEVELDB_ONCE_INIT;
static const Comparator* bytewise;
static void InitModule() {
  bytewise = new BytewiseComparatorImpl;
}
const Comparator* BytewiseComparator() {
  port::InitOnce(&once, InitModule);
  return bytewise;
}
```

