# BlockBuilder分析

## 功能

按照一定的格式将多个key value编码到一个block中。

注意，这个不负责将key value解析出来。

## 设计详情

### 格式

```cpp
// An entry for a particular key-value pair has the form:
//     shared_bytes: varint32
//     unshared_bytes: varint32
//     value_length: varint32
//     key_delta: char[unshared_bytes]
//     value: char[value_length]
// shared_bytes == 0 for restart points.
//
// The trailer of the block has the form:
//     restarts: uint32[num_restarts]
//     num_restarts: uint32
// restarts[i] contains the offset within the block of the ith restart point.
```

### 编码方式

首先每隔一定数量的key，存在一个restart key【即共享key的起始点】，在尾部有对这些restart key的索引【相对于文件起始位置的偏移】，其目的是为了方便二分查找特定key。

为了减少对空间的占用，每个key都会相对于上一个key将相同的部分进行裁剪，当然restart key除外。

我觉得restart key存在的另一个价值是可以减少遍历的数量，如果没有restart key，重头到位都按照这个规格裁剪，则只能进行顺序查找，没法跳过一些数据，因为每个都基于上一个进行了裁剪，而有了restart key后就可以直接比较从而跳过数据。

当restart key间隔越大，则查找越慢【解析的数据更多】，但占用的空间越小。间隔越小，则查找越快，但占用空间越多。

BlockBuilder通过add将数据编码进目的缓存buffer\_中，最后再调用finish将restart合并到buffer\_末尾，然后返回。

```cpp
class BlockBuilder {
 public:
  explicit BlockBuilder(const Options* options);

  // Reset the contents as if the BlockBuilder was just constructed.
  void Reset();

  // REQUIRES: Finish() has not been called since the last call to Reset().
  // REQUIRES: key is larger than any previously added key
  void Add(const Slice& key, const Slice& value);

  // Finish building the block and return a slice that refers to the
  // block contents.  The returned slice will remain valid for the
  // lifetime of this builder or until Reset() is called.
  Slice Finish();

  // Returns an estimate of the current (uncompressed) size of the block
  // we are building.
  size_t CurrentSizeEstimate() const;

  // Return true iff no entries have been added since the last Reset()
  bool empty() const {
    return buffer_.empty();
  }

 private:
  const Options*        options_;
  std::string           buffer_;      // Destination buffer
  std::vector<uint32_t> restarts_;    // Restart points
  int                   counter_;     // Number of entries emitted since restart
  bool                  finished_;    // Has Finish() been called?
  std::string           last_key_;

  // No copying allowed
  BlockBuilder(const BlockBuilder&);
  void operator=(const BlockBuilder&);
};
```

