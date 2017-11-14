---
layout: post
title: "Android多线程分析之二：Thread的实现"
date: 2014-07-10 21:21:40 +0800
comments: true
categories: [软件开发]
tags: [android, thread]
description: Android多线程分析之二：Thread的实现
keywords: android, thread
---

## 引言
在前文[Android多线程分析之一：使用Thread异步下载图像](https://luozhaohui.github.io/blog/2014/07/08/android-thread-tutoria01/)中演示了如何使用<code>Thread</code>处理异步事务。示例中这个<code>Java Thread</code>类都是位于<code>Framework</code>层的类，它自身是通过<code>JNI</code>转调<code>dalvik</code>里面的<code>Thread</code>相关方法实现的。因此要分析<code>Andriod</code> 中的线程，就需要分析这两层中的与线程相关的代码，这就是本文要探讨的主题。本文将把<code>Framework</code>层中的<code>Java Thread</code>称为<code>Android Thread</code>，而把<code>dalvik</code>中的 <code>Thread</code>成为<code>dalvik Thread</code>。 

<!--more-->

## 源码路径
本文涉及到的 <code>Android</code> 源码路径：
```
android/libcore/luni/src/main/java/java/lang/Runnable.java
android/libcore/luni/src/main/java/java/lang/Thread.java
android/libcore/luni/src/main/java/java/lang/ThreadGroup.java
android/libcore/luni/src/main/java/java/lang/VMThread.java
android/dalvik/vm/native/java_lang_VMThread.cpp
android/dalvik/vm/Thread.cpp
```

## Android Thread
首先来分析<code>Android Thread</code>，这个类的源码在<code>android/libcore/luni/src/main/java/java/lang/Thread.java</code>，它实现了<code>Runnable</code>接口。<code>Runnable</code>只有一个无参无返回值的<code>void run()</code>的接口：

``` java
/**
 * Represents a command that can be executed. Often used to run code in a
 * different {@link Thread}.
 */
public interface Runnable {

    /**
     * Starts executing the active part of the class' code. This method is
     * called when a thread is started that has been created with a class which
     * implements {@code Runnable}.
     */
    public void run();
}
```

<code>Android Thread</code>存在六种状态，这些状态定义在枚举<code>State</code>中，源码注释写的很清晰，在这里就不罗嗦了：
``` java
    /**
     * A representation of a thread's state. A given thread may only be in one
     * state at a time.
     */
    public enum State {
        /**
         * The thread has been created, but has never been started.
         */
        NEW,
        /**
         * The thread may be run.
         */
        RUNNABLE,
        /**
         * The thread is blocked and waiting for a lock.
         */
        BLOCKED,
        /**
         * The thread is waiting.
         */
        WAITING,
        /**
         * The thread is waiting for a specified amount of time.
         */
        TIMED_WAITING,
        /**
         * The thread has been terminated.
         */
        TERMINATED
    }
```

<code>Android Thread</code>类中一些关键成员变量如下：
```
    volatile VMThread vmThread;
    volatile ThreadGroup group;
    volatile boolean daemon;    
    volatile String name;
    volatile int priority;
    volatile long stackSize;
    Runnable target;
    private static int count = 0;
    private long id;
    ThreadLocal.Values localValues;
```

> <code>vmThread</code>：可视为对<code>dalvik thread</code>的简单封装，<code>Thread</code>类通过<code>VMThread</code>里面的<code>JNI</code>方法来调用<code>dalvik</code>中操作线程的方法，通过它的成员变量<code>thread</code>和<code>vmata</code>，我们可以将<code>Android Thread</code>和<code>dalvik Thread</code>的关联起来；  
> <code>group</code>：每一个线程都属于一个<code>group</code>，当线程被创建时就会加入一个特定的<code>group</code>，当线程运行结束，会从这个<code>group</code>中移除；  
> <code>daemon</code>：当前线程是不是守护线程，守护线程只会在没有非守护线程运行的情况下才会运行；  
> <code>priority</code>：线程优先级，<code>Java Thread</code>类的线程优先级取值范围为 [1, 10]，默认优先级为 5；  
> <code>stackSize</code>：线程栈大小，默认为 0，即使用默认的线程栈大小（由<code>dalvik<</code>中的全局变量<code>gDvm.stackSize</code>决定）；  
> <code>target</code>：一个<code>Runnable</code>对象，<code>Thread</code>的<code>run()</code>方法中会转调该<code>target</code>的<code>run()</code>方法，这是线程真正处理事务的地方；  
> <code>id</code>：线程<code>id</code>，通过递增<code>count</code>得到该<code>id</code>，如果没有显式给线程设置名字，那么就会使用<code>Thread+id</code>当作线程的名字。注意这不是真正意义上的线程<code>id</code>，即在<code>logcat</code>中打印的<code>tid</code>并不是这个<code>id</code>，那<code>tid</code>是指<code>dalvik</code>线程的<code>id</code>；
> <code>localValues</code>：线程本地存储（<code>TLS</code>）数据；  

接下来，我们来看<code>Android Thread</code>的构造函数，大部分构造函数都是通过转调静态函数<code>create</code>实现的，下面来详细分析<code>create</code>这个关键函数：

``` java
    private void create(ThreadGroup group, Runnable runnable, String threadName, long stackSize) {
        Thread currentThread = Thread.currentThread();
        if (group == null) {
            group = currentThread.getThreadGroup();
        }

        if (group.isDestroyed()) {
            throw new IllegalThreadStateException("Group already destroyed");
        }

        this.group = group;

        synchronized (Thread.class) {
            id = ++Thread.count;
        }

        if (threadName == null) {
            this.name = "Thread-" + id;
        } else {
            this.name = threadName;
        }

        this.target = runnable;
        this.stackSize = stackSize;

        this.priority = currentThread.getPriority();

        this.contextClassLoader = currentThread.contextClassLoader;

        // Transfer over InheritableThreadLocals.
        if (currentThread.inheritableValues != null) {
            inheritableValues = new ThreadLocal.Values(currentThread.inheritableValues);
        }

        // add ourselves to our ThreadGroup of choice
        this.group.addThread(this);
    }
```

首先，通过静态函数<code>currentThread</code>获取创建线程所在的当前线程，然后将当前线程的一些属性传递给即将创建的新线程。这是通过<code>VMThread</code>转调<code>dalvik</code>中的代码实现的。

``` java
    public static Thread currentThread() {
        return VMThread.currentThread();
    }
```

## dalvik Thread

<code>VMThread</code>的<code>currentThread</code>是一个<code>native</code>方法，其<code>JNI</code>实现为<code>android/dalvik/vm/native/java_lang_VMThread.cpp</code>中的<code>Dalvik_java_lang_VMThread_currentThread</code>方法：

``` cpp
static void Dalvik_java_lang_VMThread_currentThread(const u4* args,
    JValue* pResult)
{
    UNUSED_PARAMETER(args);

    RETURN_PTR(dvmThreadSelf()->threadObj);
}
```

该方法里的<code>dvmThreadSelf()</code>方法定义在<code>android/dalvik/vm/Thread.cpp</code>中：

``` cpp
Thread* dvmThreadSelf()
{
    return (Thread*) pthread_getspecific(gDvm.pthreadKeySelf);
}
```

从上面的调用栈可以看到，每一个<code>dalvik</code>线程都会将自身存放在<code>key</code>为<code>pthreadKeySelf</code> 的线程本地存储中，获取当前线程时，只需要根据这个<code>key</code>查询获取即可，<code>dalvik Thread</code>有一个名为<code>threadObj</code> 的成员变量：

``` cpp
    /* the java/lang/Thread that we are associated with */
    Object*     threadObj;
```

<code>dalvik Thread</code>这个成员变量<code>threadObj</code>关联的就是对应的<code>Android Thread</code>对象，所以通过<code>native</code>方法<code>VMThread.currentThread()</code>返回的是存储在<code>TLS</code>中的当前<code>dalvik</code>线程对应的<code>Android Thread</code>。

接着分析上面的代码，如果没有给新线程指定<code>group</code>那么就会指定<code>group</code>为当前线程所在的<code>group</code>中，然后给新线程设置<code>name</code>，<code>priority</code>等。最后通过调用<code>ThreadGroup</code>的<code>addThread</code>方法将新线程添加到<code>group</code>中：

``` java
    /**
     * Called by the Thread constructor.
     */
    final void addThread(Thread thread) throws IllegalThreadStateException {
        synchronized (threadRefs) {
            if (isDestroyed) {
                throw new IllegalThreadStateException();
            }
            threadRefs.add(new WeakReference<Thread>(thread));
        }
    }
```

<code>ThreadGroup</code>的代码相对简单，它有一个名为<code>threadRefs</code>的列表，持有属于同一组的<code>thread</code>引用，可以对一组<code>thread</code>进行一些线程操作。

上面分析的是<code>Android Thread</code>的构造过程，从上面的分析可以看出，<code>Android Thread</code>的构造方法仅仅是设置了一些线程属性，并没有真正去创建一个新的<code>dalvik Thread</code>，<code>dalvik Thread</code>创建过程要等到客户代码调用<code>Android Thread</code>的<code>start()</code>方法才会进行。下面我们来分析<code>Java Thread</code>的<code>start()</code>方法：

``` java
public synchronized void start() {

        if (hasBeenStarted) {
            throw new IllegalThreadStateException("Thread already started."); // TODO Externalize?
        }

        hasBeenStarted = true;

        VMThread.create(this, stackSize);
    }
}
```

<code>Android Thread</code>的 <code>start</code> 方法很简单，仅仅是转调 <code>VMThread</code> 的 <code>native</code> 方法 <code>create</code>，其 <code>JNI</code> 实现为 <code>android/dalvik/vm/native/java_lang_VMThread.cpp</code> 中的 <code>Dalvik_java_lang_VMThread_create</code> 方法：

``` cpp
static void Dalvik_java_lang_VMThread_create(const u4* args, JValue* pResult)
{
    Object* threadObj = (Object*) args[0];
    s8 stackSize = GET_ARG_LONG(args, 1);

    /* copying collector will pin threadObj for us since it was an argument */
    dvmCreateInterpThread(threadObj, (int) stackSize);
    RETURN_VOID();
}
```

## Create 线程
<code>dvmCreateInterpThread</code> 的实现在 <code>Thread.cpp</code> 中，由于这个函数的内容很长，在这里只列出关键的地方：

``` cpp
bool dvmCreateInterpThread(Object* threadObj, int reqStackSize)
{
    Thread* self = dvmThreadSelf();
    ...
    Thread* newThread = allocThread(stackSize); 
    newThread->threadObj = threadObj;
    ...
    Object* vmThreadObj = dvmAllocObject(gDvm.classJavaLangVMThread, ALLOC_DEFAULT);
    dvmSetFieldInt(vmThreadObj, gDvm.offJavaLangVMThread_vmData, (u4)newThread);
    dvmSetFieldObject(threadObj, gDvm.offJavaLangThread_vmThread, vmThreadObj);
    ...
    pthread_t threadHandle;
    int cc = pthread_create(&threadHandle, &threadAttr, interpThreadStart, newThread);

    /*
     * Tell the new thread to start.
     *
     * We must hold the thread list lock before messing with another thread.
     * In the general case we would also need to verify that newThread was
     * still in the thread list, but in our case the thread has not started
     * executing user code and therefore has not had a chance to exit.
     *
     * We move it to VMWAIT, and it then shifts itself to RUNNING, which
     * comes with a suspend-pending check.
     */
    dvmLockThreadList(self);

    assert(newThread->status == THREAD_STARTING);
    newThread->status = THREAD_VMWAIT;
    pthread_cond_broadcast(&gDvm.threadStartCond);

    dvmUnlockThreadList();
    ...
}

/*
 * Alloc and initialize a Thread struct.
 *
 * Does not create any objects, just stuff on the system (malloc) heap.
 */
static Thread* allocThread(int interpStackSize)
{
    Thread* thread;
    thread = (Thread*) calloc(1, sizeof(Thread));
    ...
    thread->status = THREAD_INITIALIZING;
}
```

首先，通过调用 <code>allocThread</code> 创建一个名为 <code>newThread</code> 的<code>dalvik Thread</code> 并设置一些属性，将设置其成员变量 <code>threadObj</code> 为传入的 <code>Android Thread</code>，这样<code>dalvik Thread</code>就与<code>Android Thread</code>关联起来了；然后创建一个名为 <code>vmThreadObj</code> 的 <code>VMThread</code> 对象，设置其成员变量 <code>vmData</code> 为 <code>newThread</code>，设置<code>Android Thread threadObj</code>的成员变量 <code>vmThread</code> 为这个 <code>vmThreadObj</code>，这样<code>Android Thread</code>通过 <code>VMThread</code> 的成员变量 <code>vmData</code> 就和<code>dalvik Thread</code>关联起来了。

## Start 线程
然后，通过 <code>pthread_create</code> 创建 <code>pthread</code> 线程，并让这个线程 <code>start</code>，这样就会进入该线程的 <code>thread entry</code> 运行，下来我们来看新线程的 <code>thread entry</code> 方法 <code>interpThreadStart</code>，同样只列出关键的地方：

``` cpp
/*
 * pthread entry function for threads started from interpreted code.
 */
static void* interpThreadStart(void* arg)
{
    Thread* self = (Thread*) arg;

    std::string threadName(dvmGetThreadName(self));
    setThreadName(threadName.c_str());

    /*
     * Finish initializing the Thread struct.
     */
    dvmLockThreadList(self);
    prepareThread(self);

    while (self->status != THREAD_VMWAIT)
        pthread_cond_wait(&gDvm.threadStartCond, &gDvm.threadListLock);

    dvmUnlockThreadList();

    /*
     * Add a JNI context.
     */
    self->jniEnv = dvmCreateJNIEnv(self);

    /*
     * Change our state so the GC will wait for us from now on.  If a GC is
     * in progress this call will suspend us.
     */
    dvmChangeStatus(self, THREAD_RUNNING);

    /*
     * Execute the "run" method.
     *
     * At this point our stack is empty, so somebody who comes looking for
     * stack traces right now won't have much to look at.  This is normal.
     */
    Method* run = self->threadObj->clazz->vtable[gDvm.voffJavaLangThread_run];
    JValue unused;

    ALOGV("threadid=%d: calling run()", self->threadId);
    assert(strcmp(run->name, "run") == 0);
    dvmCallMethod(self, run, self->threadObj, &unused);
    ALOGV("threadid=%d: exiting", self->threadId);

    /*
     * Remove the thread from various lists, report its death, and free
     * its resources.
     */
    dvmDetachCurrentThread();

    return NULL;
}

/*
 * Finish initialization of a Thread struct.
 *
 * This must be called while executing in the new thread, but before the
 * thread is added to the thread list.
 *
 * NOTE: The threadListLock must be held by the caller (needed for
 * assignThreadId()).
 */
static bool prepareThread(Thread* thread)
{
    assignThreadId(thread);
    thread->handle = pthread_self();
    thread->systemTid = dvmGetSysThreadId();

    setThreadSelf(thread);
    ...

    return true;
}

/*
 * Explore our sense of self.  Stuffs the thread pointer into TLS.
 */
static void setThreadSelf(Thread* thread)
{
    int cc;

    cc = pthread_setspecific(gDvm.pthreadKeySelf, thread);
    ...
}
```

在新线程的 <code>thread entry</code> 方法 <code>interpThreadStart</code> 中，首先设置线程的名字，然后通过调用 <code>prepareThread</code> 设置线程 <code>id</code> 以及其它一些属性，并调用 <code>setThreadSelf</code> 将新<code>dalvik Thread</code>自身保存在 <code>TLS</code> 中，这样之后就能通过 <code>dvmThreadSelf</code> 方法从 <code>TLS</code> 中获取它。然后修改状态为 <code>THREAD_RUNNING</code>，并调用对应<code>Android Thread</code>的 <code>run</code> 方法，运行客户代码：

``` java
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

对于继承自<code>Android Thread</code>带有 <code>Looper</code> 的 <code>Android HandlerThread</code> 来说，会调用它覆写 <code>run()</code> 方法：（关于 <code>Looper</code> 的话题下一篇会讲到，这里暂且略过）

``` java
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

## Detach 线程
<code>target</code> 在前面已经做了介绍，它是线程真正处理逻辑事务的地方。一旦逻辑事务处理完毕从 <code>run</code> 中返回，线程就会回到 <code>interpThreadStart</code> 方法中，继续执行 <code>dvmDetachCurrentThread</code> 方法：

``` cpp
/*
 * Detach the thread from the various data structures, notify other threads
 * that are waiting to "join" it, and free up all heap-allocated storage.
 * /
void dvmDetachCurrentThread()
{
    Thread* self = dvmThreadSelf();
    Object* vmThread;
    Object* group;
    ...
    group = dvmGetFieldObject(self->threadObj, gDvm.offJavaLangThread_group);

    /*
     * Remove the thread from the thread group.
     */
    if (group != NULL) {
        Method* removeThread =
            group->clazz->vtable[gDvm.voffJavaLangThreadGroup_removeThread];
        JValue unused;
        dvmCallMethod(self, removeThread, group, &unused, self->threadObj);
    }

    /*
     * Clear the vmThread reference in the Thread object.  Interpreted code
     * will now see that this Thread is not running.  As this may be the
     * only reference to the VMThread object that the VM knows about, we
     * have to create an internal reference to it first.
     */
    vmThread = dvmGetFieldObject(self->threadObj,
                    gDvm.offJavaLangThread_vmThread);
    dvmAddTrackedAlloc(vmThread, self);
    dvmSetFieldObject(self->threadObj, gDvm.offJavaLangThread_vmThread, NULL);

    /* clear out our struct Thread pointer, since it's going away */
    dvmSetFieldObject(vmThread, gDvm.offJavaLangVMThread_vmData, NULL);

    ...

    /*
     * Thread.join() is implemented as an Object.wait() on the VMThread
     * object.  Signal anyone who is waiting.
     */
    dvmLockObject(self, vmThread);
    dvmObjectNotifyAll(self, vmThread);
    dvmUnlockObject(self, vmThread);

    dvmReleaseTrackedAlloc(vmThread, self);
    vmThread = NULL;

    ...

    dvmLockThreadList(self);

    /*
     * Lose the JNI context.
     */
    dvmDestroyJNIEnv(self->jniEnv);
    self->jniEnv = NULL;

    self->status = THREAD_ZOMBIE;

    /*
     * Remove ourselves from the internal thread list.
     */
    unlinkThread(self);

    ...

    releaseThreadId(self);
    dvmUnlockThreadList();

    setThreadSelf(NULL);

    freeThread(self);
}

/*
 * Free a Thread struct, and all the stuff allocated within.
 */
static void freeThread(Thread* thread)
{
    ...
    free(thread);
}
```

在 <code>dvmDetachCurrentThread</code> 函数里，首先获取当前线程 <code>self</code>，这里获得的就是当前执行 <code>thread entry</code> 的新线程，然后通过其对应的<code>Android Thread</code>对象 <code>threadObj</code> 获取该对象所在 <code>group</code>，然后将 <code>threadObj</code> 这个<code>Android Thread</code>对象从 <code>group</code> 中移除；接着清除 <code>Android</code> 与 <code>dalvik</code> 线程之间的关联关系，并通知 <code>join</code> 该线程的其它线程；最后，设置线程状态为 <code>THREAD_ZOMBIE</code>，清除 <code>TLS</code> 中存储的线程值，并通过调用 <code>freeThread</code> 释放内存，至此线程就终结了。
