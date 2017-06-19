# LogReader分析

## 功能

提供了以record为单位读取log文件的功能。

### 文件格式

``` 
block := record* trailer?
record :=
  checksum: uint32     // crc32c of type and data[] ; little-endian
  length: uint16       // little-endian
  type: uint8          // One of FULL, FIRST, MIDDLE, LAST
  data: uint8[length]
```

当block最后只有6个字节时，由于一条记录最少要占用7个字节【4+2+1】，则这6个字节会被跳过。

当最后有7个字节时，且新增一个非0长度的record时，则需要将这7个字节以FIRST类型填充，用户数据则放在紧接着的block中。**当前长度应该设为0**。

### 疑问

为什么要分block，直接按record写不就行了？**原因是读取log文件时，需要指定读取大小，如果不按block size读取，则只能一次性读取log中全部数据**

### type类型

```
// Zero is reserved for preallocated files
kZeroType = 0,
FULL == 1
// For fragments
FIRST == 2
MIDDLE == 3
LAST == 4
```

## 具体实现

SkipToInitialBlock提供了跳过某个block之前的数据功能。

```cpp
bool Reader::SkipToInitialBlock() {
  //在block中的偏移
  size_t offset_in_block = initial_offset_ % kBlockSize;
  //得到block的开始位置
  uint64_t block_start_location = initial_offset_ - offset_in_block;
  // Don't search a block if we'd be in the trailer
  //如果偏移在block的最后6字节[为无效数据]，则跳过该block，到下一个block开始位置。
  if (offset_in_block > kBlockSize - 6) {
    offset_in_block = 0;
    block_start_location += kBlockSize;
  }
  end_of_buffer_offset_ = block_start_location;
  // Skip to start of first block that can contain the initial record
  if (block_start_location > 0) {
    Status skip_status = file_->Skip(block_start_location);
    if (!skip_status.ok()) {
      //私有方法，封装了判断report类是否存在，及调用report类的Corruption方法。
      ReportDrop(block_start_location, skip_status);
      return false;
    }
  }
  return true;
}
```

ReadPhysicalRecord只读取record，并不根据type来组装成一个完整的record。

首先会预读block大小的数据到buffer中，然后在达到buffer末尾时，再次从文件中读取，直到EOF。

其主要职责是解析record数据，并校验数据是否有误，返回该数据。

```cpp
unsigned int Reader::ReadPhysicalRecord(Slice* result) {
  while (true) {
    //如果buffer的大小小于7bytes，则说明达到了block的末尾，需要读取新的block
    if (buffer_.size() < kHeaderSize) {
      if (!eof_) {
        // Last read was a full read, so this is a trailer to skip
        buffer_.clear();
        Status status = file_->Read(kBlockSize, &buffer_, backing_store_);
        end_of_buffer_offset_ += buffer_.size();
        //如果读取失败，则报告并退出
        if (!status.ok()) {
          buffer_.clear();
          ReportDrop(kBlockSize, status);
          eof_ = true;
          return kEof;
        } else if (buffer_.size() < kBlockSize) {
          //小于需要读取的大小，说明已经到文件末尾了。
          eof_ = true;
        }
        continue;
      } else {
        // Note that if buffer_ is non-empty, we have a truncated header at the
        // end of the file, which can be caused by the writer crashing in the
        // middle of writing the header. Instead of considering this an error,
        // just report EOF.
        buffer_.clear();
        return kEof;
      }
    }
    // Parse the header
    const char* header = buffer_.data();
    const uint32_t a = static_cast<uint32_t>(header[4]) & 0xff;
    const uint32_t b = static_cast<uint32_t>(header[5]) & 0xff;
    const unsigned int type = header[6];
    const uint32_t length = a | (b << 8);
    if (kHeaderSize + length > buffer_.size()) {
      size_t drop_size = buffer_.size();
      buffer_.clear();
      //数据长度跟实际不相符，如果不是达到文件末尾了，则报告错误，否则返回EOF
      if (!eof_) {
        ReportCorruption(drop_size, "bad record length");
        return kBadRecord;
      }
      // If the end of the file has been reached without reading |length| bytes
      // of payload, assume the writer died in the middle of writing the record.
      // Don't report a corruption.
      return kEof;
    }
    if (type == kZeroType && length == 0) {
      // Skip zero length record without reporting any drops since
      // such records are produced by the mmap based writing code in
      // env_posix.cc that preallocates file regions.
      buffer_.clear();
      return kBadRecord;
    }
    // Check crc
    if (checksum_) {
      uint32_t expected_crc = crc32c::Unmask(DecodeFixed32(header));
      uint32_t actual_crc = crc32c::Value(header + 6, 1 + length);
      if (actual_crc != expected_crc) {
        // Drop the rest of the buffer since "length" itself may have
        // been corrupted and if we trust it, we could find some
        // fragment of a real log record that just happens to look
        // like a valid log record.
        size_t drop_size = buffer_.size();
        buffer_.clear();
        ReportCorruption(drop_size, "checksum mismatch");
        return kBadRecord;
      }
    }
    buffer_.remove_prefix(kHeaderSize + length);
    // Skip physical record that started before initial_offset_
    if (end_of_buffer_offset_ - buffer_.size() - kHeaderSize - length <
        initial_offset_) {
      result->clear();
      return kBadRecord;
    }
    *result = Slice(header + kHeaderSize, length);
    return type;
  }
}
```

```cpp
//scratch的目的是为了存储中间数据，如果是直接返回而没有中间数据则直接使用record[比如FullType].
//如果实在函数内部自己生成string实例以slice返回回去，则删除它也挺费劲。如果是在外部的一个非指针实例，则可以方便的析构。
bool Reader::ReadRecord(Slice* record, std::string* scratch) {
  if (last_record_offset_ < initial_offset_) {
    if (!SkipToInitialBlock()) {
      return false;
    }
  }
  scratch->clear();
  record->clear();
  bool in_fragmented_record = false;
  // Record offset of the logical record that we're reading
  // 0 is a dummy value to make compilers happy
  uint64_t prospective_record_offset = 0;
  Slice fragment;
  while (true) {
    //从文件中读取record片段
    const unsigned int record_type = ReadPhysicalRecord(&fragment);
    // ReadPhysicalRecord may have only had an empty trailer remaining in its
    // internal buffer. Calculate the offset of the next physical record now
    // that it has returned, properly accounting for its header size.
    //physical_record_offset为当前读取片段的起始位置。
    uint64_t physical_record_offset =
        end_of_buffer_offset_ - buffer_.size() - kHeaderSize - fragment.size();
   	//如果设置了initial_offset>0，则首次读取的数据是中间数据时[即Middle和Last]，略过它们。[因为便宜到第二个block或以上时，虽然是block的起始位置，但很有可能碰到中间片段]
    if (resyncing_) {
      if (record_type == kMiddleType) {
        continue;
      } else if (record_type == kLastType) {
        resyncing_ = false;
        continue;
      } else {
        resyncing_ = false;
      }
    }
    switch (record_type) {
      case kFullType:
        if (in_fragmented_record) {
          // Handle bug in earlier versions of log::Writer where
          // it could emit an empty kFirstType record at the tail end
          // of a block followed by a kFullType or kFirstType record
          // at the beginning of the next block.
          if (scratch->empty()) {
            in_fragmented_record = false;
          } else {
            ReportCorruption(scratch->size(), "partial record without end(1)");
          }
        }
        //如果是整块的record，则直接返回。
        prospective_record_offset = physical_record_offset;
        scratch->clear();
        *record = fragment;
        last_record_offset_ = prospective_record_offset;
        return true;
      case kFirstType:
        if (in_fragmented_record) {
          // Handle bug in earlier versions of log::Writer where
          // it could emit an empty kFirstType record at the tail end
          // of a block followed by a kFullType or kFirstType record
          // at the beginning of the next block.
          if (scratch->empty()) {
            in_fragmented_record = false;
          } else {
            ReportCorruption(scratch->size(), "partial record without end(2)");
          }
        }
        //如果是First，则将数据保存在scratch里
        prospective_record_offset = physical_record_offset;
        scratch->assign(fragment.data(), fragment.size());
        in_fragmented_record = true;
        break;
      case kMiddleType:
        if (!in_fragmented_record) {
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(1)");
        } else {
          //如果是Middle，则追加到scratch中
          scratch->append(fragment.data(), fragment.size());
        }
        break;
      case kLastType:
        if (!in_fragmented_record) {
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(2)");
        } else {
          //如果是Last，则追加后包装在slice里并返回。
          scratch->append(fragment.data(), fragment.size());
          *record = Slice(*scratch);
          last_record_offset_ = prospective_record_offset;
          return true;
        }
        break;
      case kEof:
        //如果达到文件末尾，则返回空
        if (in_fragmented_record) {
          // This can be caused by the writer dying immediately after
          // writing a physical record but before completing the next; don't
          // treat it as a corruption, just ignore the entire logical record.
          scratch->clear();
        }
        return false;
      case kBadRecord:
        //碰到异常记录，则停止read。
        if (in_fragmented_record) {
          ReportCorruption(scratch->size(), "error in middle of record");
          in_fragmented_record = false;
          scratch->clear();
        }
        break;
      default: {
        //碰到未知类型，则停止read。
        char buf[40];
        snprintf(buf, sizeof(buf), "unknown record type %u", record_type);
        ReportCorruption(
            (fragment.size() + (in_fragmented_record ? scratch->size() : 0)),
            buf);
        in_fragmented_record = false;
        scratch->clear();
        break;
      }
    }
  }
  return false;
}
```

## 技巧

1、读取底层数据跟上层的合并分离，ReadPhysicalRecord只从物理文件中读取，ReadRecord则根据定义的格式合并底层数据。