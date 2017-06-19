# WriteBatch 分析

## 功能

WriteBatch holds a collection of updates to apply atomically to a DB。即将更新的集合原子性的应用到数据库中。

WriteBatchInternal provides static methods for manipulating a WriteBatch that we don't want in the public WriteBatch interface

### rep_ 格式

record的格式如下：

```
8字节的fixed64 sequence num
4字节的fixedint32 count,决定后面有多少个key value
  1字节的tag，值含义为kTypeValue=0x1,kTypeDeletion=0x0
  LengthPrefixedSlice key
  LengthPrefixedSlice value
```

## 具体实现

总共实现了4个方法。以及一个handler接口和一个WriteBatchInternal私有友类。handler配合iterate方法，迭代batch的内容，过程中将解析的结果调用handler的put和delete方法，由handler做具体处理。

为了能保持WriteBatch接口简洁以及不暴露内部实现，通过定义一个内部友类来实现偏内部的辅助操作。

count由WriteBatch的delete和put维护，而sequence则由辅助类在外部设置，因为sequence是这次批量操作的起始编码，故外部设置一次就行，转换成memtable时，会在batch起始sequence上逐条递增，即每个put或delete操作都有一个自己的唯一sequence。具体见下面的MemTableInserter分析。

```cpp
// Store the mapping "key->value" in the database
void WriteBatch::Put(const Slice& key, const Slice& value) {
  //通过辅助类增加rep_中的count
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  //追加类型到rep_
  rep_.push_back(static_cast<char>(kTypeValue));
  //以varint32+data的格式追加到rep_中
  PutLengthPrefixedSlice(&rep_, key);
  PutLengthPrefixedSlice(&rep_, value);
}
// If the database contains a mapping for "key", erase it.  Else do nothing
void WriteBatch::Delete(const Slice& key) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeDeletion));
  //删除仅追加key
  PutLengthPrefixedSlice(&rep_, key);
}
// Clear all updates buffered in this batch.
void Clear();
//由于是const，支持无锁并发
Status WriteBatch::Iterate(Handler* handler) const {
  Slice input(rep_);//由于并不是只读取当前count数目的数据，因此iterate过程中不支持同时Put和Delete
  if (input.size() < kHeader) {
    return Status::Corruption("malformed WriteBatch (too small)");
  }
  //迭代时不关心sequence和count
  input.remove_prefix(kHeader);
  Slice key, value;
  int found = 0;
  while (!input.empty()) {
    found++;
    //根据tag判断后续的数据是Delete还是Put格式。
    char tag = input[0];
    input.remove_prefix(1);
    switch (tag) {
      case kTypeValue:
        //如果是Put,则读取key-value，并调用handler的put，由其做具体处理。
        if (GetLengthPrefixedSlice(&input, &key) &&
            GetLengthPrefixedSlice(&input, &value)) {
          handler->Put(key, value);
        } else {
          return Status::Corruption("bad WriteBatch Put");
        }
        break;
      case kTypeDeletion:
        //delete则只需要读取key，再由handler的delete方法处理。
        if (GetLengthPrefixedSlice(&input, &key)) {
          handler->Delete(key);
        } else {
          return Status::Corruption("bad WriteBatch Delete");
          }
        break;
      default:
        return Status::Corruption("unknown WriteBatch tag");
    }
  }
  //判断数据是否一致。
  if (found != WriteBatchInternal::Count(this)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}
```

internal友类，根据rep_的格式封装了count，sequence读取设置方法。

```cpp
int WriteBatchInternal::Count(const WriteBatch* b) {
  return DecodeFixed32(b->rep_.data() + 8);
}
void WriteBatchInternal::SetCount(WriteBatch* b, int n) {
  EncodeFixed32(&b->rep_[8], n);
}
SequenceNumber WriteBatchInternal::Sequence(const WriteBatch* b) {
  return SequenceNumber(DecodeFixed64(b->rep_.data()));
}
void WriteBatchInternal::SetSequence(WriteBatch* b, SequenceNumber seq) {
  EncodeFixed64(&b->rep_[0], seq);
}
```

而后，又封装了batch转memtable，slice转batch，两个batch合并的功能。其中batch转memtable，则有handler的子类MemTableInserter，配合WriteBatch的iterate将内部的数据转化为memtable数据。

```cpp
Status WriteBatchInternal::InsertInto(const WriteBatch* b,
                                      MemTable* memtable) {
  MemTableInserter inserter;
  inserter.sequence_ = WriteBatchInternal::Sequence(b);
  inserter.mem_ = memtable;
  return b->Iterate(&inserter);
}

void WriteBatchInternal::SetContents(WriteBatch* b, const Slice& contents) {
  assert(contents.size() >= kHeader);
  b->rep_.assign(contents.data(), contents.size());
}

void WriteBatchInternal::Append(WriteBatch* dst, const WriteBatch* src) {
  SetCount(dst, Count(dst) + Count(src));
  assert(src->rep_.size() >= kHeader);
  dst->rep_.append(src->rep_.data() + kHeader, src->rep_.size() - kHeader);
}
```

接下来我们看下MemTableInserter的实现

MemTableInserter将WriteBatch中的每条操作都以递增的唯一sequence添加进memtable，根据第二个参数来辨别是delete还是put，delete时value值为空slice.

```cpp
namespace {
class MemTableInserter : public WriteBatch::Handler {
 public:
  SequenceNumber sequence_;
  MemTable* mem_;
  virtual void Put(const Slice& key, const Slice& value) {
    mem_->Add(sequence_, kTypeValue, key, value);
    sequence_++;
  }
  virtual void Delete(const Slice& key) {
    mem_->Add(sequence_, kTypeDeletion, key, Slice());
    sequence_++;
  }
};
}  // namespace
```

## 技巧

1. 当一个类需要提供一个基于内部的接口时，为了保持本身的简洁，可以通过友类来辅助实现。
2. 当方法只负责解析数据，不负责具体处理时，可以通过传递和调用处理接口，由外部决定做怎样的处理。
3. 数据格式尽量不暴露给外部，由自己维护。如果有多层数据，则各自维护自己所在层，不需要关心其它层的格式，这样其中一层有变，不影响其它层。

