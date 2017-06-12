# Env 分析2

## 具体实现

### 线程相关

Env在线程方面定义了两个辅助方法，**PthreadCall**用来统一提示，**BGThreadWrapper**则用来间接执行**BGThread**。

**BGItem**则用来存储调用方法及参数。

Env自身维护了一个双向队列组成的任务队列，用来暂存任务，以供BGThread消费。

```cpp
  void PthreadCall(const char* label, int result) {
    if (result != 0) {
      fprintf(stderr, "pthread %s: %s\n", label, strerror(result));
      abort();
    }
  }

  // BGThread() is the body of the background thread
  void BGThread();
  static void* BGThreadWrapper(void* arg) {
    reinterpret_cast<PosixEnv*>(arg)->BGThread();
    return NULL;
  }
  pthread_mutex_t mu_;
  pthread_cond_t bgsignal_;
  pthread_t bgthread_;
  bool started_bgthread_;
  // Entry per Schedule() call
  struct BGItem { void* arg; void (*function)(void*); };
  typedef std::deque<BGItem> BGQueue;
  BGQueue queue_;

  PosixLockTable locks_;
  Limiter mmap_limit_;
  Limiter fd_limit_;
```

posix env的构造函数初始化了mutex以及bgsignal两个锁。

```cpp
PosixEnv::PosixEnv()
    : started_bgthread_(false),
      mmap_limit_(MaxMmaps()),
      fd_limit_(MaxOpenFiles()) {
  PthreadCall("mutex_init", pthread_mutex_init(&mu_, NULL));
  PthreadCall("cvar_init", pthread_cond_init(&bgsignal_, NULL));
}
```

下面看下调度函数，整个过程都加了排它锁。首先检查是否已经有后台线程，如果没有则创建一个，也就是说一个env只有一个后台线程。接着检查任务队列是否为空，如果为空则试着先唤醒BG线程，接着讲任务放进队列。

```cpp
void PosixEnv::Schedule(void (*function)(void*), void* arg) {
  PthreadCall("lock", pthread_mutex_lock(&mu_));
  // Start background thread if necessary
  if (!started_bgthread_) {
    started_bgthread_ = true;
    PthreadCall(
        "create thread",
        pthread_create(&bgthread_, NULL,  &PosixEnv::BGThreadWrapper, this));
  }
  // If the queue is currently empty, the background thread may currently be
  // waiting.
  if (queue_.empty()) {
    PthreadCall("signal", pthread_cond_signal(&bgsignal_));
  }
  // Add to priority queue
  queue_.push_back(BGItem());
  queue_.back().function = function;
  queue_.back().arg = arg;
  PthreadCall("unlock", pthread_mutex_unlock(&mu_));
}
```

接着看下BGThread怎么执行任务的，不断循环从队列里取任务，如果为空则wait，	队列里提取任务后，执行它。

```cpp
void PosixEnv::BGThread() {
  while (true) {
    // Wait until there is an item that is ready to run
    PthreadCall("lock", pthread_mutex_lock(&mu_));
    while (queue_.empty()) {
      PthreadCall("wait", pthread_cond_wait(&bgsignal_, &mu_));
    }

    void (*function)(void*) = queue_.front().function;
    void* arg = queue_.front().arg;
    queue_.pop_front();

    PthreadCall("unlock", pthread_mutex_unlock(&mu_));
    (*function)(arg);
  }
}
```

env还提供了新建线程执行任务的方法，pthread_create只能传递一个参数，就新建了一个StartThreadState来保存方法及参数，再用wrapper函数包装后再执行。

```cpp
namespace {
struct StartThreadState {
  void (*user_function)(void*);
  void* arg;
};
}
static void* StartThreadWrapper(void* arg) {
  StartThreadState* state = reinterpret_cast<StartThreadState*>(arg);
  state->user_function(state->arg);
  delete state;
  return NULL;
}

void PosixEnv::StartThread(void (*function)(void* arg), void* arg) {
  pthread_t t;
  StartThreadState* state = new StartThreadState;
  state->user_function = function;
  state->arg = arg;
  PthreadCall("start thread",
              pthread_create(&t, NULL,  &StartThreadWrapper, state));
}
```

### 文件相关

封装了文件的打开及关闭，使调用者只需要关心读或者写即可。

env构造函数设置了mmap及openfiles的限制，mmap的限制在64位为1000个，小于的则为0。openfiles则从系统调用中获取进程最大文件描述符个数，如果获取失败则默认50个，成功且是无限，则取int得最大数，否则设置为系统配置的五分之一。

```cpp
static int mmap_limit = -1;//全局变量
static int open_read_only_file_limit = -1;//全局变量

// Return the maximum number of concurrent mmaps.
static int MaxMmaps() {
  if (mmap_limit >= 0) {
    return mmap_limit;
  }
  // Up to 1000 mmaps for 64-bit binaries; none for smaller pointer sizes.
  mmap_limit = sizeof(void*) >= 8 ? 1000 : 0;
  return mmap_limit;
}
// Return the maximum number of read-only files to keep open.
static intptr_t MaxOpenFiles() {
  if (open_read_only_file_limit >= 0) {
    return open_read_only_file_limit;
  }
  struct rlimit rlim;
  if (getrlimit(RLIMIT_NOFILE, &rlim)) {
    // getrlimit failed, fallback to hard-coded default.
    open_read_only_file_limit = 50;
  } else if (rlim.rlim_cur == RLIM_INFINITY) {
    open_read_only_file_limit = std::numeric_limits<int>::max();
  } else {
    // Allow use of 20% of available file descriptors for read-only files.
    open_read_only_file_limit = rlim.rlim_cur / 5;
  }
  return open_read_only_file_limit;
}
```

创建新SequentialFile实例，封装了实例化方法，以便后续调整新实例化策略。以读的方式打开文件，并实例化PosixSequentialFile。

```cpp
  virtual Status NewSequentialFile(const std::string& fname,
                                   SequentialFile** result) {
    FILE* f = fopen(fname.c_str(), "r");
    if (f == NULL) {
      *result = NULL;
      return IOError(fname, errno);
    } else {
      *result = new PosixSequentialFile(fname, f);
      return Status::OK();
    }
  }
```

NewRandomAccessFile封装了实例化随机读文件类的方法，首先试着打开文件，如果打不开说明有问题，就不用继续了。接着尝试获取mmap的资格，如果获取到了，则以mmap的方式将该fd映射到内存，成功后实例化PosixMmapReadableFile；如果没有获取到则实例化PosixRandomAccessFile。

封装的好处就是，可以根据策略实例化不同的子类。

```cpp
 virtual Status NewRandomAccessFile(const std::string& fname,
                                     RandomAccessFile** result) {
    *result = NULL;
    Status s;
    int fd = open(fname.c_str(), O_RDONLY);
    if (fd < 0) {
      s = IOError(fname, errno);
    } else if (mmap_limit_.Acquire()) {
      uint64_t size;
      s = GetFileSize(fname, &size);
      if (s.ok()) {
        void* base = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
        if (base != MAP_FAILED) {
          *result = new PosixMmapReadableFile(fname, base, size, &mmap_limit_);
        } else {
          s = IOError(fname, errno);
        }
      }
      close(fd);
      if (!s.ok()) {
        mmap_limit_.Release();
      }
    } else {
      *result = new PosixRandomAccessFile(fname, fd, &fd_limit_);
    }
    return s;
 }
```

实例化写入文件类，以写的方式打开文件，并实例化PosixWritableFile实现

```cpp
 virtual Status NewWritableFile(const std::string& fname,
                                 WritableFile** result) {
    Status s;
    FILE* f = fopen(fname.c_str(), "w");
    if (f == NULL) {
      *result = NULL;
      s = IOError(fname, errno);
    } else {
      *result = new PosixWritableFile(fname, f);
    }
    return s;
  }
```

实例化追加写文件类，跟writable的区别是以追加的方式打开文件。

```cpp
virtual Status NewAppendableFile(const std::string& fname,
                                   WritableFile** result) {
    Status s;
    FILE* f = fopen(fname.c_str(), "a");
    if (f == NULL) {
      *result = NULL;
      s = IOError(fname, errno);
    } else {
      *result = new PosixWritableFile(fname, f);
    }
    return s;
  }
```

PosixLockTable提供的作用是相同进程如果已经锁定了file，则可以判断不能再次锁定。其通过set类重复insert相同值时，返回false来实现判断。而且由于用了互斥scoped锁，所以多进程也是可以的。

```cpp
// Set of locked files.  We keep a separate set instead of just
// relying on fcntrl(F_SETLK) since fcntl(F_SETLK) does not provide
// any protection against multiple uses from the same process.
class PosixLockTable {
 private:
  port::Mutex mu_;
  std::set<std::string> locked_files_;
 public:
  bool Insert(const std::string& fname) {
    MutexLock l(&mu_);
    return locked_files_.insert(fname).second;
  }
  void Remove(const std::string& fname) {
    MutexLock l(&mu_);
    locked_files_.erase(fname);
  }
};
```

我们接着看下对文件lock的操作。

fcntl是对fd操作的一个系统调用。解释如下，可以理解为防止多个操作程序同时访问文件的某部分。

> The remaining `fcntl` commands are used to support *record locking*, which permits multiple cooperating programs to prevent each other from simultaneously accessing parts of a file in error-prone ways.
>
> An *exclusive* or *write* lock gives a process exclusive access for writing to the specified part of the file. While a write lock is in place, no other process can lock that part of the file.
>
> A *shared* or *read* lock prohibits any other process from requesting a write lock on the specified part of the file. However, other processes can request read locks.

```cpp
static int LockOrUnlock(int fd, bool lock) {
  errno = 0;
  struct flock f;
  memset(&f, 0, sizeof(f));
  f.l_type = (lock ? F_WRLCK : F_UNLCK);
  f.l_whence = SEEK_SET;
  f.l_start = 0;
  f.l_len = 0;        // Lock/unlock entire file
  return fcntl(fd, F_SETLK, &f);
}
```

LockFile提供了锁定某个文件的方法。具体方式为，通过多个if语句使整个异常判断实现简单；首先打开文件名所对应的fd，接着看是否已经在locks里【防止同一个进程对同一文件获取多次锁】，接着通过LockOrUnlock防止多个进程获取同一个锁，最后实例化FileLock子类PosixFileLock。

```cpp
  virtual Status LockFile(const std::string& fname, FileLock** lock) {
    *lock = NULL;
    Status result;
    int fd = open(fname.c_str(), O_RDWR | O_CREAT, 0644);
    if (fd < 0) {
      result = IOError(fname, errno);
    } else if (!locks_.Insert(fname)) {
      close(fd);
      result = Status::IOError("lock " + fname, "already held by process");
    } else if (LockOrUnlock(fd, true) == -1) {
      result = IOError("lock " + fname, errno);
      close(fd);
      locks_.Remove(fname);
    } else {
      PosixFileLock* my_lock = new PosixFileLock;
      my_lock->fd_ = fd;
      my_lock->name_ = fname;
      *lock = my_lock;
    }
    return result;
  }
```

## 技巧

1. 通过多个if，使异常判断一气呵成。
2. 通过将实例化父类的方法封装，从而可以通过不同策略，实例化不同子类。 封装的好处。