# Binder通讯原理

*基于Android 11的源码剖析，笔记记录binder通讯原理的实现过程*；

根据网络上的各路大神案例，都是从典型的mediaserver进程开始分析，binder服务的注册注册过程；

mediaserver进程启动的main函数开始分析，至于init.rc的注册启动进程就跳过了；main函数代码如下：

```c
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    //创建与binder驱动交互和binder线程池的管理者
    sp<ProcessState> proc(ProcessState::self());
    //获取ServiceManager的客户端BpServiceManager
    sp<IServiceManager> sm(defaultServiceManager());
    ALOGI("ServiceManager: %p", sm.get());
    //创建MediaPlayerService服务，和向ServiceManager注册服务
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    registerExtensions();
    ::android::hardware::configureRpcThreadpool(16, false);
    //启动binder线程池
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
    ::android::hardware::joinRpcThreadpool();
}
```

## Binder线程池的注册

每个采用 Binder 的进程会有一个或多个用于处理接收数据的线程，位于 Binder 线程池。采用 Binder 机制的进程最典型的就是应用程序进程了。那应用程序进程的 Binder 线程池是在什么时候启动的呢？

### `ProcessState`

源码位置：frameworks/native/libs/binder/ProcessState.cpp

**`ProcessState` 是 Binder 机制核心之一**，它是 Binder 通信的基础，负责与 Binder 驱动的交互与 Binder 线程池的管理。它实现了单例模式，通过 `self()` 函数获取实例，每个进程仅有一个。

#### `ProcessState`创建

实现了单例模式，通过 `self()` 函数获取实例。来看看它的构造函数，如下：

```c
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))//访问binder设备，并与binder驱动交互
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{

// TODO(b/139016109): enforce in build system
#if defined(__ANDROID_APEX__)
    LOG_ALWAYS_FATAL("Cannot use libbinder in APEX (only system.img libbinder) since it is not stable.");
#endif

    if (mDriverFD >= 0) {
        //映射binder驱动，提供通讯的虚拟空间
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

#ifdef __ANDROID__
    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver '%s' could not be opened.  Terminating.", driver);
#endif
}
```

ProcessState创建是主要步骤是：

1. 访问binder设备，并与binder驱动交互；
2. 映射binder驱动，提供通讯基础的虚拟空间；

**其中提供通讯基础的虚拟空间默认大小是由`BINDER_VM_SIZE`这个宏来决定的，宏定义如下：**

```c
//binder分配的默认内存大小为1M-8k
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
```

下面来主要看看`open_driver`函数，函数内容如下：

```c
static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);//访问binder设备
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);//进行版本比对
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
          ALOGE("Binder driver protocol(%d) does not match user space protocol(%d)! ioctl() return value: %d",
                vers, BINDER_CURRENT_PROTOCOL_VERSION, result);
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);//设置binder线程池最大线程数
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}
```

在`open_driver`函数中主要处理：

1. 访问binder设备，通过open函数来实现，具体现在不做详细说明；

2. 通过ioctl进行binder的版本比较

3. 通知binder驱动binder线程池的默认最大线程数，而这个最大线程数由`DEFAULT_MAX_BINDER_THREADS`宏来决定；宏定义如下：

   ```c
   //默认binder线程池的最大线程数,那加上本身binder默认的最大可并发访问的线程数为16
   #define DEFAULT_MAX_BINDER_THREADS 15
   ```

#### 启动binder线程池

`ProcessState` 实例后调用其 `startThreadPool()` 函数，以启动进程的 Binder 线程池。

```c
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);//创建线程
        t->run(name.string());//启动线程
    }
}
```

`mThreadPoolStarted` 用于标识线程池是否已经启动过，以确保 Binder 线程池仅初始化一次。`spawnPooledThread()` 函数启动了一个 Binder 线程，类型为 `PoolThread`，函数参数表示这是 Binder 线程池中的第一线程。

```c
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
  //PoolThread继承Thread类。t->run()方法最终调用内部类 PoolThread的threadLoop()方法。
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};
```

`PoolThread`继承`Thread`类。`t->run()`方法最终调用内部类 `PoolThread`的`threadLoop`()方法。在主要创建了`IPCThreadState`和执行了`IPCThreadState`的`joinThreadPool`函数；

在mediaserver的main函数中后面又执行了一次`IPCThreadState`的`joinThreadPool`函数，这两次的区别是一个在子线程执行，一个是在进程主线程执行，**mediaserver默认binder的事件监听线程数是2吗？**这样binder线程池算基本完成！

### `IPCThreadState`

源码位置：frameworks/native/libs/binder/IPCThreadState.cpp

`IPCThreadState` 同样是 Binder 机制的核心之一，它用于管理与 Binder 通信相关线程的状态，每个 Binder 线程都会通过此将自己注册到 Binder 驱动。一个具有多个线程的进程里应该会有多个IPCThreadState对象了，只不过每个线程只需一个IPCThreadState对象而已。所以要放在binder线程池中统一管理。

#### `IPCThreadState`创建

`IPCThreadState`同样是通过 `self()` 函数获取实例的。

```c
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS.load(std::memory_order_acquire)) {
restart:
        const pthread_key_t k = gTLS;
        //获取当前线程是否创建了IPCThreadState，如果创建了直接返回,类似Looper里的ThreadLocal
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;
    }

    // Racey, heuristic test for simultaneous shutdown.
    if (gShutdown.load(std::memory_order_relaxed)) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
        return nullptr;
    }

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS.load(std::memory_order_relaxed)) {
        //创建线程唯一的标签
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
                    strerror(key_create_value));
            return nullptr;
        }
        gHaveTLS.store(true, std::memory_order_release);
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;//回到开始根据线程唯一标记创建IPCThreadState
}
```

`self()` 函数是一个工厂函数，用于获取 `IPCThreadState` 实例。`self()` 根据 `pthread_getspecific()` 管理每个参与 Binder 通信线程的实例，类似`Looper`里的`ThreadLocal`，每个参与 Binder 通信的线程其 `IPCThreadState` 对象都是相互独立的，保证了后续操作的线程安全。构造函数内容其实，很简单主要是绑定线程唯一标记和初始化输入输出缓冲区；

```c
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mServingStackPointer(nullptr),
      mWorkSource(kUnsetWorkSource),
      mPropagateWorkSource(false),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0),
      mCallRestriction(mProcess->mCallRestriction)
{
    //将线程唯一标签保存的内容设置为自身
    pthread_setspecific(gTLS, this);
    //获取当前进程的pid和uid信息
    clearCaller();
    //设置输入缓冲区大小，默认256
    mIn.setDataCapacity(256);
    //设置输出缓冲区大小，默认256
    mOut.setDataCapacity(256);
}
```

#### `IPCThreadState::joinThreadPool`函数

`joinThreadPool`函数就是一个死循环，不断从驱动获取数据;

```c
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        //清除上一次通讯的输入缓冲区
        processPendingDerefs();
        //处理下一条信息或者等待
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            LOG_ALWAYS_FATAL("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```

如此看来`IPCThreadState`是通过`getAndExecuteCommand`来不断获取通讯数据的；

```c
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
    //从bender驱动中获取数据
    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();//读取命令字段
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount++;
        if (mProcess->mExecutingThreadsCount >= mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs == 0) {
            mProcess->mStarvationStartTimeMs = uptimeMillis();
        }
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
        //进行binder命令解析
        result = executeCommand(cmd);

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount--;
        if (mProcess->mExecutingThreadsCount < mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs != 0) {
            int64_t starvationTimeMs = uptimeMillis() - mProcess->mStarvationStartTimeMs;
            if (starvationTimeMs > 100) {
                ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
                      mProcess->mMaxThreads, starvationTimeMs);
            }
            mProcess->mStarvationStartTimeMs = 0;
        }
        pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
    }

    return result;
}
```

`getAndExecuteCommand`的执行步骤：

1. 通过`talkWithDriver`向binder驱动获取通讯数据；
2. 读取命令字段，并通过`executeCommand`函数进行不同命令字段的解析和处理

## 获取`ServiceManager`

获取Service Manager是通过`defaultServiceManager()`方法来完成，当进程注册服务(addService)或 获取服务(getService)的过程之前，都需要先调用defaultServiceManager()方法来获取`gDefaultServiceManager`对象。

大概流程图如下：

<img src="./img/defaultServiceManager.png" style="zoom:50%;" />

`defaultServiceManager`函数代码如下：

```c
sp<IServiceManager> defaultServiceManager()
{
    std::call_once(gSmOnce, []() {
        sp<AidlServiceManager> sm = nullptr;
        //避免ServiceManager未启动完成，重复请求
        while (sm == nullptr) {
            //获取BpServiceManager
            sm = interface_cast<AidlServiceManager>(ProcessState::self()->getContextObject(nullptr));
            if (sm == nullptr) {
                ALOGE("Waiting 1s on context object on %s.", ProcessState::self()->getDriverName().c_str());
                sleep(1);
            }
        }
        //创建BpServiceManager代理对象
        gDefaultServiceManager = new ServiceManagerShim(sm);
    });

    return gDefaultServiceManager;
}
```

