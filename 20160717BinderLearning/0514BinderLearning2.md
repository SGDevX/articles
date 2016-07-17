---
layout: post
title:  "Binder学习笔记（二）—— defaultServiceManager()返回了什么？"
date:   2016-05-07 00:39:48 +0800
categories: Android
tags:   binder
toc: true
comments: true
---
不管是客户端还是服务端，头部都要先调用
``` c++
sp < IServiceManager > sm = defaultServiceManager();
```
defaultServiceManager()都干了什么，它返回的是什么实例呢？
该函数定义在frameworks/native/libs/binder/IserviceManager.cpp:33
``` c++
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
   
        ... ...
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));  
        ... ...
    
    return gDefaultServiceManager;
}
```
关键步骤可以分解为几步：
1、ProcessState::self()
2、ProcessState::getContextObject(NULL)
3、interface_cast&lt;IserviceManager&gt;(ProcessState::self()->getContextObject(NULL))

# ProcessState::self()
frameworks/native/libs/binder/ProcessState.cpp:70 
``` c++
sp<ProcessState> ProcessState::self()  // 又是一个进程单体
{
    ... ...
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState;  // 首次创建在这里
    return gProcess;
}
```
ProcessState的构造函数很简单，frameworks/native/libs/binder/ProcessState.cpp:339
``` c++
ProcessState::ProcessState()
    : mDriverFD(open_driver())  // 这里打开了/dev/binder文件，并返回文件描述符
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    ... ...
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
    ... ...
}
```
ProcessState的构造函数主要完成两件事：
1、初始化列表里调用opern_driver()，打开了文件/dev/binder；
2、将文件映射到内存。

ProcessState::self()返回单体实例。

# ProcessState::getContextObject(NULL)
接下来是defaultServiceManager()的第二步调用，该函数定义在frameworks/native/libs/binder/ProcessState.cpp:85
``` c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}
```
继续深入，frameworks/native/libs/binder/ProcessState/cpp:179
``` c++
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{   // handle = NULL
    sp<IBinder> result;
    ... ...
    handle_entry* e = lookupHandleLocked(handle);  //正常情况下总会返回一个非空实例
        ... ...
        IBinder* b = e->binder;
        ... ...
            if (handle == 0) {  // 首次创建b为NULL，handle为0
                ... ...
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = new BpBinder(handle); 
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;  // 返回的是BpBinder(0)
        ... ...

    return result;
}
```
因此getStrongProxyForHandle(0)返回的就是new BpBinder(0)。有几处细节可以再回头来关注一下：
## ProcessState::lookupHandleLocked(int32_t handle)
该函数定义在frameworks/native/libs/binder/ProcessState.cpp:166
``` c++
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```
成员变量mHandleToObject是一个数组：
``` c++
Vector<handle_entry>    mHandleToObject;
```
该数组以handle为索引下标，这个handle是驱动层为每一个进程分配的进程内唯一的整形数，用来标识一个binder的引用，这在后面还会讲到。
该函数遍历数组查找handle，如果没找到则会向该数组中插入一个新元素。新元素的binder、refs成员默认均为NULL，在getStrongProxyForHandle(…)中会被赋值。

# interface_cast&lt;IserviceManager&gt;(ProcessState::self()->getContextObject(NULL))
defaultServiceManager()函数最终返回的是这一大坨，函数interface_cast(…)在binder体系中非常常用，后面还会不断遇见。该函数定义在frameworks/native/include/binder/IInterface.h:41
``` c++
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{   // obj=ProcessState::self()->getContextObject(NULL)，即
    // new BpBinder(0)
    return INTERFACE::asInterface(obj);
}
```
代入模板参数及实参后为：
``` c++
IServiceManager::asInterface(new BpBinder(0));
```
该函数藏在宏IMPLEMENT_META_INTERFACE中，frameworks/native/libs/binder/IServiceManager.cpp:185
``` c++
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");
```
展开后为：
``` c++
android::sp< IServiceManager > IServiceManager::asInterface(
            const android::sp<android::IBinder>& obj)
    {   // obj=new BpBinder(0)
        android::sp< IServiceManager > intr;
        if (obj != NULL) {
            intr = static_cast< IServiceManager *>( 
                obj->queryLocalInterface(IServiceManager::descriptor).get());
            if (intr == NULL) {  // 首次会走这里
                intr = new BpServiceManager(obj);
            }
        }
        return intr;
    }
```
因此它返回的就是new BpServiceManager(new BpBinder(0))。
> 经过层层抽丝剥茧之后，defaultServiceManager()的返回值即为：
> new BpServiceManager(new BpBinder(0))。
> 它表示handle为0的binder引用，即对ServiceManager的引用。
> 

我们再顺道看一下BpServiceManager的继承关系以及构造函数，frameworks/native/libs/binder/IServiceManager.cpp:126
``` c++
class BpServiceManager : public BpInterface<IServiceManager>
```
frameworks/native/libs/binder/IInterface.h:62
``` c++
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
```
BpServiceManager继承自BpInterface，后者继承自BpRefBase。
frameworks/native/libs/binder/IServiceManager.cpp:129
``` c++
    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {
    }
```
frameworks/native/include/binder/IInterface.h:134
``` c++
template<typename INTERFACE>
inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote)
    : BpRefBase(remote)
{
}
```
frameworks/native/libs/binder/Binder.cpp:241
``` c++
BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(NULL), mState(0)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);

    if (mRemote) {
        mRemote->incStrong(this);           // Removed on first IncStrong().
        mRefs = mRemote->createWeak(this);  // Held for our entire lifetime.
    }
}
```
BpServiceManager通过构造函数，沿着继承关系一路将impl参数传递给基类BpRefBase，基类将它赋给数据成员mRemote。在defaultServiceManager()中传给BpServiceManager构造函数的参数是new BpBinder(0)。

