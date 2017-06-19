#coding分析
##功能
coding是用来将各种类型编码到char*或string中，以及从char*和string中解码，其中需要注意encoding为little还是big，如果是little，则0x76543210,则变成char*时,则为0x10 0x32 0x54 0x76。big则相反为0x76 0x54 0x32 0x10。
###设计详情
fixed int部分，主要有针对char*的函数名以encode开头；针对string的则以put开头，实际是封装了encode，先encode到buf，在append到string中，复制了两次。

从little encoding 转成big encoding 通过硬编码的方式转换。

```cpp
void EncodeFixed32(char* buf, uint32_t value) {
  if (port::kLittleEndian) {
    memcpy(buf, &value, sizeof(value));
  } else {
    buf[0] = value & 0xff;
    buf[1] = (value >> 8) & 0xff;
    buf[2] = (value >> 16) & 0xff;
    buf[3] = (value >> 24) & 0xff;
  }
}
void EncodeFixed64(char* buf, uint64_t value) {
  if (port::kLittleEndian) {
    memcpy(buf, &value, sizeof(value));
  } else {
    buf[0] = value & 0xff;
    buf[1] = (value >> 8) & 0xff;
    buf[2] = (value >> 16) & 0xff;
    buf[3] = (value >> 24) & 0xff;
    buf[4] = (value >> 32) & 0xff;
    buf[5] = (value >> 40) & 0xff;
    buf[6] = (value >> 48) & 0xff;
    buf[7] = (value >> 56) & 0xff;
  }
}
void PutFixed32(std::string* dst, uint32_t value) {
  char buf[sizeof(value)];
  EncodeFixed32(buf, value);
  dst->append(buf, sizeof(buf));
}

void PutFixed64(std::string* dst, uint64_t value) {
  char buf[sizeof(value)];
  EncodeFixed64(buf, value);
  dst->append(buf, sizeof(buf));
}
```

var int部分，由于占用字节数根据值的不同而不同，因此这里返回了最新的目的指针，以供后续接着编码。var编码规则为，如果字节的第8位为1，则说明下一个还属于本int。因此按这个硬编码即可。比如如果大于128且小于128^2，则需要编成2字节。比默认的4字节节省两字节的开销。

```cpp
// 可以替换成varint64的while方式
char* EncodeVarint32(char* dst, uint32_t v) {
  // Operate on characters as unsigneds
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  static const int B = 128;
  if (v < (1<<7)) {
    *(ptr++) = v;
  } else if (v < (1<<14)) {
    *(ptr++) = v | B;
    *(ptr++) = v>>7;
  } else if (v < (1<<21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)) {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}
// 继续封装encode，通过返回的最新指针计算编码长度。
void PutVarint32(std::string* dst, uint32_t v) {
  //这里为什么是5呢，因为正常是4个char，但var int从每个字节中占用了第八位作为标识符，因此4字节的最大数编码成var int后少占用了4字节的4个bit位，因此会被编码成5个字节。
  char buf[5];
  char* ptr = EncodeVarint32(buf, v);
  dst->append(buf, ptr - buf);
}

char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  while (v >= B) {
    *(ptr++) = (v & (B-1)) | B;
    v >>= 7;
  }
  *(ptr++) = static_cast<unsigned char>(v);
  return reinterpret_cast<char*>(ptr);
}

void PutVarint64(std::string* dst, uint64_t v) {
  //跟32位的同理，但这里多占用了4个bit为，共8个bit位，因此需要10个字节，多的8个bit位需要占用2个字节，因为还需要其第8位又是标志位。
  char buf[10];
  char* ptr = EncodeVarint64(buf, v);
  dst->append(buf, ptr - buf);
}
```

slice编码，将内容长度为编码为var int 32，最大支持2^32即4GB长度的内容，内存则直接append到string中，不需要额外编码。

```cpp
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}
```

辅助函数，计算64位的int值会被编码成多少位。

```cpp
int VarintLength(uint64_t v) {
  int len = 1;
  while (v >= 128) {
    v >>= 7;
    len++;
  }
  return len;
}
```

Get部分就是Put的反向，则不再详细介绍

get fixed貌似不需要额外函数，直接转换即可。

有点特殊的是PtrFallback，有最大长度限制，以避免超过p本身的内存大小。

get var int 32 和64都实际使用的是PtrFallback，以避免超过p本身大小。

```cpp
const char* GetVarint32PtrFallback(const char* p,
                                   const char* limit,
                                   uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28 && p < limit; shift += 7) {
    uint32_t byte = *(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```

