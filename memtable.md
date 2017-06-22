# memtable分析

### SkipList's key 格式

``` 
// Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()
// internal_key format:
  // user_key : char[user_key.size()]
  // SequenceNumber[58bit]+valueType[8bit] :FixedInt64
```



## 设计实现

### Option

构造函数会初始化默认参数，comparator默认使用的是BytewiseComparator。

```cpp
Options::Options()
    : comparator(BytewiseComparator()),
      create_if_missing(false),
      error_if_exists(false),
      paranoid_checks(false),
      env(Env::Default()),
      info_log(NULL),
      write_buffer_size(4<<20),
      max_open_files(1000),
      block_cache(NULL),
      block_size(4096),
      block_restart_interval(16),
      max_file_size(2<<20),
      compression(kSnappyCompression),
      reuse_logs(false),
      filter_policy(NULL) {
}
```

原生的comparator传进来后，赋值给internal_comparator_，调用的是InternalKeyComparator的构造函数，赋值给InternalKeyComparator的user_comparator\_变量。即InternalKeyComparator包装了UserComparator，这里即为BytewiseComparator。

```cpp
DBImpl::DBImpl(const Options& raw_options, const std::string& dbname)
    : env_(raw_options.env),
      internal_comparator_(raw_options.comparator),
      internal_filter_policy_(raw_options.filter_policy),
      options_(SanitizeOptions(dbname, &internal_comparator_,
                               &internal_filter_policy_, raw_options)),
      owns_info_log_(options_.info_log != raw_options.info_log),
      owns_cache_(options_.block_cache != raw_options.block_cache),
      dbname_(dbname),
      db_lock_(NULL),
      shutting_down_(NULL),
      bg_cv_(&mutex_),
      mem_(NULL),
      imm_(NULL),
      logfile_(NULL),
      logfile_number_(0),
      log_(NULL),
      seed_(0),
      tmp_batch_(new WriteBatch),
      bg_compaction_scheduled_(false),
      manual_compaction_(NULL) {
  has_imm_.Release_Store(NULL);

  // Reserve ten files or so for other uses and give the rest to TableCache.
  const int table_cache_size = options_.max_open_files - kNumNonTableCacheFiles;
  table_cache_ = new TableCache(dbname_, &options_, table_cache_size);

  versions_ = new VersionSet(dbname_, &options_, table_cache_,
                             &internal_comparator_);
}
```

```cpp
// A comparator for internal keys that uses a specified comparator for
// the user key portion and breaks ties by decreasing sequence number.
class InternalKeyComparator : public Comparator {
 private:
  const Comparator* user_comparator_;
 public:
  explicit InternalKeyComparator(const Comparator* c) : user_comparator_(c) { }
  virtual const char* Name() const;
  virtual int Compare(const Slice& a, const Slice& b) const;
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const;
  virtual void FindShortSuccessor(std::string* key) const;

  const Comparator* user_comparator() const { return user_comparator_; }

  int Compare(const InternalKey& a, const InternalKey& b) const;
};
```

**为什么sequence number要倒序排列呢？**

原因是相同的key，seq+type大的在前面，即新数据在前面，因此从小到大查找时，可以少遍历一些老数据。举个例子，已有数据如下所示：

key:'Yintao' @ 13 : 0,value:
key:'Yintao' @ 9 : 1,value:Hello Tao!

此时我们要get key=Yintao seq=13 type=1，如果正向排序，则需要先遍历Yintao 9。反向排序则只要遍历到第一条即可。由于131比130小【seq+type倒序排】，因此FindEqualOrGreater返回的是逻辑上较大的数据130即key:'Yintao' @ 13 : 0,value:

```cpp
InternalKeyComparator的compare策略是，userkey部分使用BytewiseComparator，从小到大正序排，而tag部分，即sequence+type为倒序。
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

MemTable的add方法，是将数据编码成key_size+key_bytes+value_size+value_bytes格式，然后插入skipList中。

```cpp
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;
  const size_t encoded_len =
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size;
  char* buf = arena_.Allocate(encoded_len);
  char* p = EncodeVarint32(buf, internal_key_size);
  memcpy(p, key.data(), key_size);
  p += key_size;
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  p = EncodeVarint32(p, val_size);
  memcpy(p, value.data(), val_size);
  assert((p + val_size) - buf == encoded_len);
  table_.Insert(buf);
}
```

memtable的Get方法，依据memkey【klength+userkey+tag】，通过iterator的seek方法找到skipList中equalOrGreater的Key，由于memkey的tag中的type为1[ValueType]，故如果skipList中由删除类型的key，则会找到大于该key的值，或者没有值返回。

为什么在seek后还需要check一下返回的key等于user key呢，原因是seek会返回大于该key的值，或者如果是相等，也有可能是不同的key压缩后正好等于user key的格式，但实际并不相等。

既然LookupKey中的memkey的type为valueType，那是否还有可能找到的user key的type为DeleteType呢，还是有可能的，如果SkipList中存在key的sequence比LookupKey的sequence大，且正好是delete key，则有可能找到delete key。

**由于Get的时候需要Sequence number，那怎么直到这个user key的sequence呢？**

```cpp
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    const char* entry = iter.key();
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, &key_length);
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {
      // Correct user key
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

## 技巧

Wrapper wrapper;这样会调用Test的explicit Test(const Test* test)构造函数。如果Wrapper的const Test test_;改为const Test* test\_ 则不会调用相当于直接赋值。

```cpp
class Test{
    public:
    explicit Test(){};
    //explicit Test(const std::string* s):s_(s){}
    explicit Test(const Test* test){
        cout<<"Test construct"<<endl;
    }
};
class Wrapper{
    private:
    const Test test_;
    public:
    //explicit Test(const std::string* s):s_(s){}
    explicit Wrapper():test_(new Test()){};
};
```

