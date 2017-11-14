---
layout: post
title: "Android多线程分析之四：MessageQueue的实现"
date: 2014-07-12 20:53:20 +0800
comments: true
categories: [软件开发]
tags: [android, thread]
description: Android多线程分析之四：MessageQueue的实现
keywords: android, thread
---

## 引言
在前面两篇文章[Android多线程分析之二：Thread的实现](https://luozhaohui.github.io/blog/2014/07/10/android-thread-tutoria02/)，[Android多线程分析之三：Handler，Looper的实现](https://luozhaohui.github.io/blog/2014/07/12/android-thread-tutoria03/)中分别介绍了 <code>Thread</code> 的创建，运行，销毁的过程以及 <code>Thread</code>与 <code>Handler</code>，<code>Looper</code> 之间的关联：<code>Thread</code> 在其 <code>run()</code> 方法中创建和运行消息处理循环 <code>Looper</code>，而 <code>Looper::loop()</code> 方法不断地从 <code>MessageQueue</code> 中获取消息，并由 <code>Handler</code> 分发处理该消息。接下来就来介绍 <code>MessageQueue</code> 的运作机制。

## 参考源码：
```
android/framework/base/core/java/android/os/MessageQueue.java
android/framework/base/core/java/android/os/Message.java
android/frameworks/base/core/jni/android_os_MessageQueue.h
android/frameworks/base/core/jni/android_os_MessageQueue.cpp
```

<!--more-->

## 构造函数及成员变量
先来看 <code>MessageQueue</code> 的构造函数以及重要的成员变量：

``` java
    // True if the message queue can be quit.
    private final boolean mQuitAllowed;
    private int mPtr; // used by native code
    Message mMessages;
    private boolean mQuiting;
    // Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
    private boolean mBlocked;
```

<code>mQuitAllowed</code>: 其含义与 <code>Looper.prepare(boolean quitAllowed)</code> 中参数含义一直，是否允许中止；
<code>mPtr</code>：<code>Android MessageQueue</code> 是通过调用 <code>C++ native MessageQueue</code> 实现的，这个 <code>mPtr</code> 就是指向 <code>native MessageQueue</code>；
<code>mMessages</code>：<code>Message</code> 是链表结构的，因此这个变量就代表 <code>Message</code> 链表；
<code>mQuiting</code>：是否终止了；
<code>mBlocked</code>：是否正在等待被激活以获取消息；

<code>MessageQueue</code> 的构造函数很简单：

``` java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        nativeInit();
    }
```

它通过转调 <code>native</code> 方法 <code>nativeInit()</code> 实现的，后者是定义在 <code>android_os_MessageQueue.cpp</code>中：

``` cpp
static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return;
    }

    nativeMessageQueue->incStrong(env);
    android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
}
static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv* env, jobject messageQueueObj,
        NativeMessageQueue* nativeMessageQueue) {
    env->SetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr,
             reinterpret_cast<jint>(nativeMessageQueue));
}
```

<code>nativeInit()</code> 方法创建 <code>NativeMessageQueue</code> 对象，并将这个对象的指针复制给 <code>Android MessageQueue</code> 的 <code>mPtr</code>。<code>NativeMessageQueue</code> 的定义如下：

``` java
class MessageQueue : public RefBase {
public:
    /* Gets the message queue's looper. */
    inline sp<Looper> getLooper() const {
        return mLooper;
    }

    bool raiseAndClearException(JNIEnv* env, const char* msg);
    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj) = 0;
protected:
    MessageQueue();
    virtual ~MessageQueue();
protected:
    sp<Looper> mLooper;
};

class NativeMessageQueue : public MessageQueue {
public:
    NativeMessageQueue();
    virtual ~NativeMessageQueue();
    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj);
    void pollOnce(JNIEnv* env, int timeoutMillis);
    void wake();

private:
    bool mInCallback;
    jthrowable mExceptionObj;
};
```

## 消息同步
其中值得关注的是 <code>NativeMessageQueue</code> 的构造以及<code>pollOnce</code>，<code>wake</code> 两个方法，它们是<code>Java MessageQueue</code> 中 <code>nativePollOnce</code> 和 <code>nativeWake</code> 的 <code>native</code> 方法：

``` cpp
NativeMessageQueue::NativeMessageQueue() : mInCallback(false), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}

void NativeMessageQueue::pollOnce(JNIEnv* env, int timeoutMillis) {
    mInCallback = true;
    mLooper->pollOnce(timeoutMillis);
    mInCallback = false;
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

在 <code>NativeMessageQueue</code> 的构造函数中，会获取当前线程的 <code>Looper</code>(注意这是 <code>C++ Looper</code>,定义在<code>frameworks/native/libs/utils/Looper.h</code> 中)，如果当前线程还没有 <code>Looper</code>，就创建一个，并保存在线程的 <code>TLS</code> 中。<code>pollOnce</code> 和 <code>wake</code> 最终都是通过 <code>Linux</code> 的 <code>epoll</code> 模型来实现的。<code>pollOnce()</code> 通过等待被激活，然后从消息队列中获取消息；<code>wake()</code> 则是激活处于等待状态的消息队列，通知它有消息到达了。这是典型的生产者-消费者模型。

对于<code>Android MessageQueue</code> 来说，其主要的工作就是：接收投递进来的消息，获取下一个需要处理的消息。这两个功能是通过 <code>enqueueMessage()</code> 和 <code>next()</code> 方法实现的。<code>next()</code> 在前一篇文章介绍 <code>Looper.loop()</code> 时提到过。

## Message
在分析这两个函数之前，先来介绍一下 <code>Message</code>：前面说过 <code>Message</code> 是完备的，即它同时带有消息内容和处理消息的 <code>Handler</code> 或 <code>callback</code>。下面列出它的主要成员变量：

``` cpp
public int what;     // 消息 id
public int arg1;     // 消息参数
public int arg2;     // 消息参数
public Object obj;   // 消息参数
long when;           // 处理延迟时间，由 Handler 的 sendMessageDelayed/postDelayed 设置
Handler target;    // 处理消息的 Handler
Runnable callback;   // 处理消息的回调
Message next;    // 链表结构，指向下一个消息
```

<code>Message</code> 有一些名为 <code>obtain</code> 的静态方法用于创建 <code>Message</code>，通常我们都是通过 <code>Handler</code> 的 <code>obtain</code> 静态方法转调 <code>Message</code> 的静态方法来创建新的 <code>Message</code>。

接下来分析 <code>enqueueMessage()</code>enqueueMessage：
``` java
    final boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }
```

首先检测消息的合法性：是否已经在处理中和是否有处理它的 <code>Handler</code>，然后判断 <code>mQuiting</code> 是否中止了，如果没有则根据消息处理时间排序将消息插入链表中的合适位置。在这其中作了一些减少同步操作的优化，即使当前消息队列已经处于 <code>Blocked</code> 状态，且队首是一个消息屏障(和内存屏障的理念一样，这里是通过 <code>p.target == null</code> 来判断队首是否是消息屏障)，并且要插入的消息是所有异步消息中最早要处理的才会 <code>needwake</code> 激活消息队列去获取下一个消息。<code>Handler</code> 的 <code>post/sendMessage</code> 系列方法最后都是通过转调 <code>MessageQueue</code> 的 <code>enqueueMessage</code> 来实现的，比如：

``` java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

其实 <code>Handler</code> 中与 <code>Message</code> 相关的静态方法都是通过 <code>MessageQueue</code> 的对应的静态方法实现的，比如 <code>removeMessages</code>, <code>hasMessages</code>, <code>hasCallbacks</code> 等等，这里就不一一详述了。至此，已经完整地分析了如何通过 <code>Handler</code> 提交消息到 <code>MessageQueue</code> 中了。

## 获取消息
下面来分析如何从 <code>MessageQueue</code> 中获取合适的消息, 这是 <code>next()</code> 要做的最主要的事情，<code>next()</code> 方法还做了其他一些事情，这些其它事情是为了提高系统效果，利用消息队列在空闲时通过 <code>idle handler</code> 做一些事情，比如 <code>gc</code> 等等。但它们和获取消息关系不大，所以这部分将从略介绍。

``` java
   final Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;

        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(mPtr, nextPollTimeoutMillis);

            synchronized (this) {
                if (mQuiting) {
                    return null;
                }

                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

队列被激活之后，首先判断队首是不是消息屏障，如果是则跳过所有的同步消息，查找最先要处理的异步消息。如果第一个待处理的消息还没有到要处理的时机则设置激活等待时间；否则这个消息就是需要处理的消息，将该消息设置为 <code>inuse</code>，并将队列设置为非  <code>blocked</code> 状态，然后返回该消息。<code>next()</code> 方法是在  <code>Looper.loop()</code> 中被调用的，<code>Looper</code> 在获得要处理的消息之后就会调用和消息关联的 <code>Handler</code> 来分发消息，这里再回顾一下：

``` java
  public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        ...
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            msg.target.dispatchMessage(msg);

            msg.recycle();
        }
    }
```

如果队列中没有消息或者第一个待处理的消息时机未到，且也没有其他利用队列空闲要处理的事务，则将队列设置为设置 <code>blocked</code> 状态，进入等待状态；否则就利用队列空闲处理其它事务。

至此，已经对 <code>Android</code> 多线程相关的主要概念 <code>Thread</code>, <code>HandlerThread</code>, <code>Handler</code>, <code>Looper</code>, <code>Message</code>, <code>MessageQueue</code> 作了一番介绍，下一篇就要讲讲 <code>AsyncTask</code>，这是为了简化 <code>UI</code> 多线程编程为提供的一个便利工具类。
