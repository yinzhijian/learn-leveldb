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

接着看下BGThread怎么执行任务的，不断循环从队列里取任务，如果为空则wait，队列里提取任务后，执行它。

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

