---
layout: post
title: "Android多线程分析之三：Handler，Looper的实现"
date: 2014-07-12 20:53:20 +0800
comments: true
categories: [软件开发]
tags: [android, thread]
description: Android多线程分析之三：Handler，Looper的实现
keywords: android, handler, looper
---

## 引言
在前文[Android多线程分析之二：Thread的实现](https://luozhaohui.github.io/blog/2014/07/10/android-thread-tutoria02/)中已经详细分析了<code>Android Thread</code> 是如何创建，运行以及销毁的，其重点是对相应 <code>native</code> 方法进行分析，今天我将聚焦于 <code>Android Framework</code> 层多线程相关的类：<code>Handler</code>, <code>Looper</code>, <code>MessageQueue</code>, <code>Message</code> 以及它们与<code>Thread</code> 之间的关系。可以用一个不太妥当的比喻来形容它们之间的关联：如果把 <code>Thread</code> 比作生产车间，那么 <code>Looper</code> 就是放在这车间里的生产线，这条生产线源源不断地从 <code>MessageQueue</code> 中获取材料 <code>Messsage</code>，并分发处理 <code>Message</code> (由于 <code>Message</code> 通常是完备的，所以 <code>Looper</code> 大多数情况下只是调度让 <code>Message</code> 的 <code>Handler</code> 去处理 <code>Message</code>)。正是因为消息需要在 <code>Looper</code> 中处理，而 <code>Looper</code> 又需运行在 <code>Thread</code> 中，所以不能随随便便在非 <code>UI</code> 线程中进行 <code>UI</code> 操作。 <code>UI</code> 操作通常会通过投递消息来实现，只有往正确的 <code>Looper</code> 投递消息才能得到处理，对于 <code>UI</code> 来说，这个 <code>Looper</code> 一定是运行在 <code>UI</code> 线程中。

<!--more-->

## 示例
在编写 <code>app</code> 的过程中，我们常常会这样来使用 <code>Handler</code>：

``` java
Handler mHandler = new Handler();
mHandler.post(new Runnable(){
	@Override
	public void run() {
		// do somework
	}
});
```

或者如这系列文章[Android多线程分析之一：使用Thread异步下载图像](https://luozhaohui.github.io/blog/2014/07/08/android-thread-tutoria01/)中的示例那样：

``` java
    private Handler mHandler= new Handler(){
        @Override
        public void handleMessage(Message msg) {
        	Log.i("UI thread", " >> handleMessage()");

            switch(msg.what){
            case MSG_LOAD_SUCCESS:
            	Bitmap bitmap = (Bitmap) msg.obj;
                mImageView.setImageBitmap(bitmap);

                mProgressBar.setProgress(100);
                mProgressBar.setMessage("Image downloading success!");
                mProgressBar.dismiss();
                break;

            case MSG_LOAD_FAILURE:
                mProgressBar.setMessage("Image downloading failure!");
                mProgressBar.dismiss();
            	break;
            }
        }
    };

    Message msg = mHandler.obtainMessage(MSG_LOAD_FAILURE, null);
    mHandler.sendMessage(msg);
```

## Handler
那么，在 <code>Handler</code> 的 <code>post/sendMessage</code> 背后到底发生了什么事情呢？下面就来解开这个谜团。
首先我们从 <code>Handler</code> 的构造函数开始分析：

``` java
    final MessageQueue mQueue; 
    final Looper mLooper; 
    final Callback mCallback; 
    final boolean mAsynchronous;

    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    public Handler() {
        this(null, false);
    }
```

上面列出了 <code>Handler</code> 的一些成员变量：

> <code>mLooper</code>：线程的消息处理循环，注意：并非每一个线程都有消息处理循环，因此 <code>Framework</code> 中线程可以分为两种：有 <code>Looper</code> 的和无 <code>Looper</code> 的。为了方便 <code>app</code> 开发，<code>Framework</code> 提供了一个有 <code>Looper</code> 的 <code>Thread</code> 实现：<code>HandlerThread</code>。在前一篇[Android多线程分析之二：Thread的实现](https://luozhaohui.github.io/blog/2014/07/10/android-thread-tutoria02/)中也提到了两种不同 <code>Thread</code> 的 <code>run()</code> 方法的区别。  

``` java
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
    Looper mLooper;
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }

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

    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}
```

## Looper
这个 <code>HandlerThread</code> 与 <code>Thread</code> 相比，多了一个类型为 <code>Looper</code> 成员变量 <code>mLooper</code>，它是在 <code>run()</code> 函数中由 <code>Looper::prepare()</code> 创建的，并保存在 <code>TLS</code> 中：

``` java
/** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

然后通过 <code>Looper::myLooper()</code> 获取创建的 <code>Looper</code>：

``` java
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```

最后通过 <code>Looper::Loop()</code> 方法运行消息处理循环：从 <code>MessageQueue</code> 中取出消息，并分发处理该消息，不断地循环这个过程。这个方法将在后面介绍。

> <code>Handler</code> 的成员变量 <code>mQueue</code> 是其成员变量 <code>mLooper</code> 的成员变量，这里只是为了简化书写，单独拿出来作为 <code>Handler</code> 的成员变量；成员变量 <code>mCallback</code> 提供了另一种使用<code>Handler</code> 的简便途径：只需实现回调接口 <code>Callback</code>，而无需子类化<code>Handler</code>，下面会讲到的：  

``` java
    /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```

成员变量 <code>mAsynchronous</code> 是标识是否异步处理消息，如果是的话，通过该 <code>Handler obtain</code> 得到的消息都被强制设置为异步的。
同是否有无 <code>Looper</code> 来区分 <code>Thread</code> 一样，<code>Handler</code> 的构造函数也分为自带 <code>Looper</code> 和外部 <code>Looper</code> 两大类：如果提供了 <code>Looper</code>，在消息会在该 <code>Looper</code> 中处理，否则消息就会在当前线程的 <code>Looper</code> 中处理，注意这里要确保当前线程一定有 <code>Looper</code>。所有的 <code>UI thread</code> 都是有 <code>Looper</code> 的，因为 <code>view/widget</code> 的实现中大量使用了消息，需要 <code>UI thread</code> 提供 <code>Looper</code> 来处理，可以参考<code>view.java</code>：

view.java
``` java
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        // Assume that post will succeed later
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
```

ViewRootImpl.java

``` java
    private void performTraversals() {
        ....
        // Execute enqueued actions on every traversal in case a detached view enqueued an action
        getRunQueue().executeActions(attachInfo.mHandler);
      ...
    }

    static RunQueue getRunQueue() {
        RunQueue rq = sRunQueues.get();
        if (rq != null) {
            return rq;
        }
        rq = new RunQueue();
        sRunQueues.set(rq);
        return rq;
    }

    /**
     * The run queue is used to enqueue pending work from Views when no Handler is
     * attached.  The work is executed during the next call to performTraversals on
     * the thread.
     * @hide
     */
    static final class RunQueue {
    ...
        void executeActions(Handler handler) {
            synchronized (mActions) {
                final ArrayList<HandlerAction> actions = mActions;
                final int count = actions.size();

                for (int i = 0; i < count; i++) {
                    final HandlerAction handlerAction = actions.get(i);
                    handler.postDelayed(handlerAction.action, handlerAction.delay);
                }

                actions.clear();
            }
        }
    }
```

从上面的代码可以看出，作为所有控件基类的 <code>view</code> 提供了 <code>post</code> 方法，用于向 <code>UI thread</code> 发送消息，并在 <code>UI thread</code> 的 <code>Looper</code> 中处理这些消息，而 <code>UI thread</code> 一定有 <code>Looper</code> 这是由 <code>ActivityThread</code> 来保证的：

``` java
public final class ActivityThread {
...
    final Looper mLooper = Looper.myLooper();
}
```

<code>UI</code> 操作需要向 <code>UI</code> 线程发送消息并在其 <code>Looper</code> 中处理这些消息。这就是为什么我们不能在非 <code>UI</code> 线程中更新 <code>UI</code> 的原因，在控件在非 <code>UI</code> 线程中构造 <code>Handler</code> 时，要么由于非 <code>UI</code> 线程没有 <code>Looper</code> 使得获取 <code>myLooper</code> 失败而抛出 <code>RunTimeException</code>，要么即便提供了 <code>Looper</code>，但这个 <code>Looper</code> 并非 <code>UI</code> 线程的 <code>Looper</code> 而不能处理控件消息。为此在 <code>ViewRootImpl</code> 中有一个强制检测 <code>UI</code> 操作是否是在 <code>UI</code> 线程中处理的方法 <code>checkThread()</code>：该方法中的 <code>mThread</code> 是在 <code>ViewRootImpl</code> 的构造函数中赋值的，它就是 <code>UI</code> 线程；该方法中的 <code>Thread.currentThread()</code> 是当前进行 <code>UI</code> 操作的线程，如果这个线程不是非 <code>UI</code> 线程就会抛出异常<code>CalledFromWrongThreadException</code>。

``` java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

如果修改[Android多线程分析之一：使用Thread异步下载图像](https://luozhaohui.github.io/blog/2014/07/08/android-thread-tutoria01/)中示例，下载完图像 <code>bitmap</code> 之后，在 <code>Thread</code> 的 <code>run()</code> 函数中设置 <code>ImageView</code> 的图像为该 <code>bitmap</code>，即会抛出上面提到的异常：

```
W/dalvikvm(796): threadid=11: thread exiting with uncaught exception (group=0x40a71930)
E/AndroidRuntime(796): FATAL EXCEPTION: Thread-75
E/AndroidRuntime(796): android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
E/AndroidRuntime(796): 	at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:4746)
E/AndroidRuntime(796): 	at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:823)
E/AndroidRuntime(796): 	at android.view.View.requestLayout(View.java:15473)
E/AndroidRuntime(796): 	at android.view.View.requestLayout(View.java:15473)
E/AndroidRuntime(796): 	at android.view.View.requestLayout(View.java:15473)
E/AndroidRuntime(796): 	at android.view.View.requestLayout(View.java:15473)
E/AndroidRuntime(796): 	at android.view.View.requestLayout(View.java:15473)
E/AndroidRuntime(796): 	at android.widget.ImageView.setImageDrawable(ImageView.java:406)
E/AndroidRuntime(796): 	at android.widget.ImageView.setImageBitmap(ImageView.java:421)
E/AndroidRuntime(796): 	at com.example.thread01.MainActivity$2$1.run(MainActivity.java:80)
```

## 处理消息
<code>Handler</code> 的构造函数暂且介绍到这里，接下来介绍：<code>handleMessage</code> 和 <code>dispatchMessage</code>：

``` java
/**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }

    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

前面提到有两种方式来设置处理消息的代码：一种是设置 <code>Callback</code> 回调，一种是子类化 <code>Handler</code>。而子类化 <code>Handler</code> 其子类就要实现 <code>handleMessage</code> 来处理自定义的消息，如前面的匿名子类示例一样。<code>dispatchMessage</code> 是在 <code>Looper::Loop()</code> 中被调用，即它是在线程的消息处理循环中被调用，这样就能让 <code>Handler</code> 不断地处理各种消息。在 <code>dispatchMessage</code> 的实现中可以看到，如果 <code>Message</code> 有自己的消息处理回调，那么就优先调用消息自己的消息处理回调：

``` java
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

否则看 <code>Handler</code> 是否有消息处理回调 <code>mCallback</code>，如果有且 <code>mCallback</code> 成功处理了这个消息就返回了，否则就调用 <code>handleMessage</code>（通常是子类的实现） 来处理消息。

在分析 <code>Looper::Loop()</code> 这个关键函数之前，先来理一理 <code>Thread，Looper，Handler，MessageQueue</code> 的关系：<code>Thread</code> 需要有 <code>Looper</code> 才能处理消息（也就是说 <code>Looper</code> 是运行在 <code>Thread</code> 中），这是通过在自定义 <code>Thread</code> 的 <code>run()</code> 函数中调用 <code>Looper::prepare()</code> 和 <code>Looper::loop()</code> 来实现，然后在 <code>Looper::loop()</code> 中不断地从 <code>MessageQueue</code> 获取由 <code>Handler</code> 投递到其中的 <code>Message</code>，并调用 <code>Message</code> 的成员变量 <code>Handler</code> 的 <code>dispatchMessage</code> 来处理消息。

下面先来看看 <code>Looper</code> 的构造函数：

``` java
    final MessageQueue mQueue;
    final Thread mThread;
    volatile boolean mRun;

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mRun = true;
        mThread = Thread.currentThread();
    }
```

<code>Looper</code> 的构造函数很简单，创建 <code>MessageQueue</code>，保存当前线程到 <code>mThread</code> 中。但它是私有的，只能通过两个静态函数 <code>prepare()</code>/<code>prepareMainLooper()</code> 来调用，前面已经介绍了 <code>prepare()</code>,下面来介绍 <code>prepareMainLooper()</code>:

``` java
/**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

<code>prepareMainLooper</code> 是通过调用 <code>prepare</code> 实现的，不过传入的参数为 <code>false</code>，表示 <code>main Looper</code> 不允许中途被中止，创建之后将 <code>looper</code> 保持在静态变量 <code>sMainLooper</code> 中。整个 <code>Framework</code> 框架只有两个地方调用了 <code>prepareMainLooper</code> 方法：

### ServerThread
<code>SystemServer.java</code> 中的 <code>ServerThread</code>，<code>ServerThread</code> 的重要性就不用说了，绝大部分 <code>Android Service</code> 都是这个线程中初始化的。这个线程是在 <code>Android</code> 启动过程中的 <code>init2()</code> 方法启动的：

``` java
    public static final void init2() {
        Slog.i(TAG, "Entered the Android system server!");
        Thread thr = new ServerThread();
        thr.setName("android.server.ServerThread");
        thr.start();
    }

class ServerThread extends Thread {
    @Override
    public void run() {
        ...
        Looper.prepareMainLooper();
        ...
        Looper.loop();
        Slog.d(TAG, "System ServerThread is exiting!");
    }
}
```

### ActivityThread
以及 <code>ActivityThread.java</code> 的 <code>main()</code> 方法：

``` java
public static void main(String[] args) {
        ....
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

<code>ActivityThread</code> 的重要性也不言而喻，它是 <code>Activity</code> 的主线程，也就是 <code>UI</code> 线程。注意这里的 <code>AsyncTask.init()</code>，在后面介绍 <code>AsyncTask</code> 时会详细介绍的，这里只提一下：<code>AsyncTask</code> 能够进行 <code>UI</code> 操作正是由于在这里调用了 <code>init()</code>。

## Looper::Loop()

有了前面的铺垫，这下我们就可以来分析 <code>Looper::Loop()</code> 这个关键函数了：

``` java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
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

<code>loop()</code> 的实现非常简单，一如前面一再说过的那样：不断地从 <code>MessageQueue</code> 中获取消息，分发消息，回收消息。从上面的代码可以看出: <code>loop()</code> 仅仅是一个不断循环作业的生产流水线，而 <code>MessageQueue</code> 则为它提供原材料 <code>Message</code>，让它去分发处理。至于 <code>Handler</code> 是怎么提交消息到 <code>MessageQueue</code> 中，<code>MessageQueue</code> 又是怎么管理消息的，且待下文分解。
