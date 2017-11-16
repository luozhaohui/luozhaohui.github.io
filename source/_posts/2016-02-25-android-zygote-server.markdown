---
layout: post
title: "Android5 Zygote 与 SystemServer 启动流程分析"
date: 2016-02-25 22:13:20 +0800
comments: true
categories: [软件开发]
tags: [android]
description: Android5 Zygote 与 SystemServer 启动流程分析
keywords: android, zygote, system server
---

## 前言
<code>Android5.0.1</code> 的启动流程与之前的版本相比变化并不大，OK，变化虽然还是有：<code>SystemServer</code> 启动过程的 <code>init1()</code>, <code>init2()</code>没有了，但主干流程依然不变：<code>Linux</code> 内核加载完毕之后，首先启动 <code>init</code> 进程，然后解析 <code>init.rc</code>，并根据其内容由 <code>init</code> 进程装载 <code>Android</code> 文件系统、创建系统目录、初始化属性系统、启动一些守护进程，其中最重要的守护进程就是 <code>Zygote</code> 进程。<code>Zygote</code> 进程初始化时会创建 <code>Dalvik</code> 虚拟机、预装载系统的资源和 <code>Java</code> 类。所有从 <code>Zygote</code> 进程 <code>fork</code> 出来的用户进程都将继承和共享这些预加载的资源。<code>init</code> 进程是 <code>Android</code> 的第一个进程，而 <code>Zygote</code> 进程则是所有用户进程的根进程。<code>SystemServer</code> 是 <code>Zygote</code> 进程 <code>fork</code> 出的第一个进程，也是整个 <code>Android</code> 系统的核心进程。

<!--more-->

## zygote 进程

### 解析 zygote.rc
在文件中 /system/core/rootdir/init.rc 中包含了 zygote.rc:
```
import /init.${ro.zygote}.rc
```

<code>${ro.zygote}</code>是平台相关的参数，实际可对应到 <code>init.zygote32.rc</code>， <code>init.zygote64.rc</code>， <code>init.zygote64_32.rc</code>， <code>init.zygote32_64.rc</code>，前两个只会启动单一<code>app_process(64)</code> 进程，而后两个则会启动两个<code>app_process</code>进程：第二个<code>app_process</code>进程称为 <code>secondary</code>，在后面的代码中可以看到相应 <code>secondary socket</code> 的创建过程。为简化起见，在这里就不考虑这种创建两个<code>app_process</code>进程的情形。

以 <code>/system/core/rootdir/init.zygote32.rc</code> 为例：
```
    service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

第一行创建了名为 <code>zygote</code> 的进程，这个进程是通过 <code>app_process</code> 的 <code>main</code> 启动并以<code>-Xzygote /system/bin --zygote --start-system-server</code>作为<code>main</code>的入口参数。

<code>app_process</code> 对应代码为 <code>framework/base/cmds/app_process/app_main.cpp</code>。在这个文件的 <code>main</code> 函数中：

```java
AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args);
    }
```

根据入口参数，我们知道 <code>zygote</code> 为<code>true</code>，<code>args</code>参数中包含了<code>"start-system-server"</code>。

<code>AppRuntime</code> 继承自 <code>AndroidRuntime</code>，因此下一步就执行到 <code>AndroidRuntime</code> 的 <code>start</code> 函数。

```java
void AndroidRuntime::start(const char* className, const Vector<String8>& options)
{
    /* start the virtual machine */ // 创建虚拟机
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env) != 0) {
        return;
    }
    onVmCreated(env);

    ...
    //调用className对应类的静态main()函数
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
    ...
}
```

<code>start</code>函数主要做两件事：创建虚拟机和调用传入类名对应类的 <code>main</code> 函数。因此下一步就执行到 <code>com.android.internal.os.ZygoteInit</code> 的 <code>main</code> 函数。
```
    public static void main(String argv[]) {
        try {
            boolean startSystemServer = false;
            String socketName = "zygote";
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                }
                ...
            }

            registerZygoteSocket(socketName);
            ...
            preload();
            ...

            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }

            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```

它主要做了三件事情:
1. 调用 <code>registerZygoteSocket</code> 函数创建了一个 <code>socket</code> 接口，用来和 <code>ActivityManagerService</code> 通讯；
2. 调用 <code>startSystemServer</code> 函数来启动 <code>SystemServer</code>;
3. 调用 <code>runSelectLoop</code> 函数进入一个无限循环在前面创建的 <code>socket</code> 接口上等待 <code>ActivityManagerService</code> 请求创建新的应用程序进程。

这里要留意 <code>catch (MethodAndArgsCaller caller)</code> 这一行，<code>android</code> 在这里通过抛出一个异常来处理正常的业务逻辑。

### 创建 zygote socket
```java
    static final String ANDROID_SOCKET_PREFIX = "ANDROID_SOCKET_";

    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                sServerSocket = new LocalServerSocket(
                        createFileDescriptor(fileDesc));
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
在这里根据传入参数<code>"zygote"</code>拼成名为 <code>fullSocketName</code> 的<code>key</code>，该 <code>key</code> 的值为：<code>ANDROID_SOCKET_zygote</code>，然后查找它对应的环境变量值，解析该值得到文件描述符号，再根据该文件描述符创建 <code>socket</code>。那么这个文件描述符是什么地方创建并写到环境变量里去的呢？还记得前面 <code>init.zygote32.rc</code> 的第二行么？
```
    socket zygote stream 660 root system
```

系统启动脚本文件 <code>init.rc</code> 是由 <code>init</code> 进程来解释执行的，而 <code>init</code> 进程的源代码位于 <code>system/core/init</code> 目录中，在 <code>init.c</code> 文件中，是由 <code>service_start</code> 函数来解释 <code>init.zygote32.rc</code> 文件中的 <code>service</code> 命令的：
```cpp
    void service_start(struct service *svc, const char *dynamic_args)
    {
        ...
        pid = fork();

        if (pid == 0) {
            struct socketinfo *si;
            ...

            for (si = svc->sockets; si; si = si->next) {
                int socket_type = (
                        !strcmp(si->type, "stream") ? SOCK_STREAM :
                            (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
                int s = create_socket(si->name, socket_type,
                                      si->perm, si->uid, si->gid, si->socketcon ?: scon);
                if (s >= 0) {
                    publish_socket(si->name, s);
                }
            }
            ...
        }
        ...
    }
```

每一个 service 命令都会促使 init 进程调用 fork 函数来创建一个新的进程，在新的进程里面，会分析里面的 socket 选项，对于每一个 socket 选项，都会通过 <code>create_socket</code> 函数来在 <code>/dev/socket</code> 目录下创建一个文件，在 zygote 进程中 socket 选项为<code>socket zygote stream 660 root system</code>，因此这个文件便是 zygote了，然后得到的文件描述符通过 <code>publish_socket</code> 函数写入到环境变量中去：

```cpp
    static void publish_socket(const char *name, int fd)
    {
        char key[64] = ANDROID_SOCKET_ENV_PREFIX;
        char val[64];

        strlcpy(key + sizeof(ANDROID_SOCKET_ENV_PREFIX) - 1,
                name,
                sizeof(key) - sizeof(ANDROID_SOCKET_ENV_PREFIX));
        snprintf(val, sizeof(val), "%d", fd);
        add_environment(key, val);

        /* make sure we don't close-on-exec */
        fcntl(fd, F_SETFD, 0);
    }
```

这里传进来的参数name值为"zygote"，而 <code>ANDROID_SOCKET_ENV_PREFIX</code> 在 <code>system/core/include/cutils/sockets.h</code> 定义为：

```cpp
#define ANDROID_SOCKET_ENV_PREFIX   "ANDROID_SOCKET_"
#define ANDROID_SOCKET_DIR          "/dev/socket"
```

因此，这里就把上面得到的文件描述符写入到以 <code>ANDROID_SOCKET_zygote</code> 为 key 值的环境变量中。又因为上面的 <code>ZygoteInit.registerZygoteSocket</code> 函数与这里创建 socket 文件的 create_socket 函数是运行在同一个进程中，因此，上面的 <code>ZygoteInit.registerZygoteSocket</code> 函数可以直接使用这个文件描述符来创建一个 Java 层的 LocalServerSocket 对象。如果其它进程也需要打开这个 <code>/dev/socket/zygote</code> 文件来和 zygote 进程进行通信，那就必须要通过文件名来连接这个 LocalServerSocket了。也就是说创建 zygote socket 之后，ActivityManagerService 就能够通过该 socket 与 zygote 进程通信从而 fork 创建新进程，android 中的所有应用进程都是通过这种方式 fork zygote 进程创建的。在 ActivityManagerService中 的 <code>startProcessLocked</code> 中调用了 <code>Process.start()</code> 方法，进而调用 <code>Process.startViaZygote</code> 和 <code>Process.openZygoteSocketIfNeeded</code>。

### 启动 SystemServer
socket 创建完成之后，紧接着就通过 <code>startSystemServer</code> 函数来启动 <code>SystemServer</code> 进程。

```java
    private static boolean startSystemServer(String abiList, String socketName)
    {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ...

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

这里我们可以从参数推测出：创建名为<code>“system_server"</code>的进程，其入口是： <code>com.android.server.SystemServer</code> 的 <code>main</code> 函数。<code>zygote</code> 进程通过 <code>Zygote.forkSystemServer</code> 函数来创建一个新的进程来启动 <code>SystemServer</code> 组件，返回值 <code>pid</code> 等 0 的地方就是新的进程要执行的路径，即新创建的进程会执行 <code>handleSystemServerProcess</code> 函数。<code>hasSecondZygote</code> 是针对 <code>init.zygote64_32.rc</code>， <code>init.zygote32_64.rc</code> 这两者情况的，在这里跳过不谈。接下来来看 <code>handleSystemServerProcess</code>：
```java
    /**
     * Finish remaining work for the newly forked system server process.
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller
    {
        closeServerSocket();

        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");

        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
            Thread.currentThread().setContextClassLoader(cl);
        }

        /*
         * Pass the remaining arguments to SystemServer.
         */
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);

        /* should never reach here */
    }
```

<code>handleSystemServerProcess</code> 会抛出 <code>MethodAndArgsCaller</code> 异常，前面提到这个异常其实是处理正常业务逻辑的，相当于一个回调。由于由 <code>zygote</code> 进程创建的子进程会继承 <code>zygote</code> 进程在前面创建的 <code>socket</code> 文件描述符，而这里的子进程又不会用到它，因此，这里就调用 <code>closeServerSocket</code> 函数来关闭它。<code>SYSTEMSERVERCLASSPATH</code> 是包含 <code>/system/framework/framework.jar</code> 的环境变量，它定义在 <code>system/core/rootdir/init.environ.rc.in</code> 中：

```
    on init
        export PATH /sbin:/vendor/bin:/system/sbin:/system/bin:/system/xbin
        export ANDROID_BOOTLOGO 1
        export ANDROID_ROOT /system
        export SYSTEMSERVERCLASSPATH %SYSTEMSERVERCLASSPATH%
        export LD_PRELOAD libsigchain.so
```

<code>handleSystemServerProcess</code> 函数接着调用 <code>RuntimeInit.zygoteInit</code> 函数来进一步执行启动 <code>SystemServer</code> 组件的操作。
``` java
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {

        commonInit();
        nativeZygoteInit();

        applicationInit(targetSdkVersion, argv, classLoader);
    }
```

<code>commonInit</code> 设置线程未处理异常 <code>handler</code>，时区等，<code>JNI</code> 方法 <code>nativeZygoteInit</code> 实现在 <code>frameworks/base/core/jni/AndroidRuntime.cpp</code> 中：
```cpp
static AndroidRuntime* gCurRuntime = NULL;

static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

<code>AndroidRuntime</code> 是个带虚函数的基类，真正的实现是在 <code>app_main.cpp</code> 中的 <code>AppRuntime</code>:
```cpp
class AppRuntime : public AndroidRuntime
{
    virtual void onStarted()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();

        AndroidRuntime* ar = AndroidRuntime::getRuntime();
        ar->callMain(mClassName, mClass, mArgs);

        IPCThreadState::self()->stopProcess();
    }

    virtual void onZygoteInit()
    {
        // Re-enable tracing now that we're no longer in Zygote.
        atrace_set_tracing_enabled(true);

        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }

    virtual void onExit(int code)
    {
        if (mClassName.isEmpty()) {
            // if zygote
            IPCThreadState::self()->stopProcess();
        }

        AndroidRuntime::onExit(code);
    }
};
```

通过执行 <code>AppRuntime::onZygoteInit</code> 函数，这个进程的 <code>Binder</code> 进程间通信机制基础设施就准备好了，参考代码 <code>frameworks/native/libs/binder/ProcessState.cpp</code>。

接下来，看 applicationInit ：
```java
    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {

        final Arguments args;
        try {
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```

<code>applicationInit</code> 仅仅是转调 <code>invokeStaticMain</code>：
```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller
    {
        Class cl;
        cl = Class.forName(className, true, classLoader);
        Method m;
        m = cl.getMethod("main", new Class[] { String[].class });


        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

<code>invokeStaticMain</code> 也很简单，通过反射找到参数 <code>className</code> 对应的类的静态 <code>main</code> 方法，然后将该方法与参数生成 <code>ZygoteInit.MethodAndArgsCaller</code> 对象当做异常抛出，这个异常对象在 <code>ZygoteInit</code> 的 <code>main</code> 函数被捕获并执行该对象的 <code>run</code> 方法。
```java
    /**
     * Helper exception class which holds a method and arguments and
     * can call them. This is used as part of a trampoline to get rid of
     * the initial process setup stack frames.
     */
    public static class MethodAndArgsCaller extends Exception
            implements Runnable {

        public void run() {
            ...
            mMethod.invoke(null, new Object[] { mArgs });
            ...
        }
    }
```

这么复杂的跳转，其实就做了一件简单的事情：根据 <code>className </code>反射调用该类的静态 <code>main</code> 方法。这个类名是 <code>ZygoteInit.startSystemServer</code> 方法中写死的 <code>com.android.server.SystemServer</code>。 从而进入 <code>SystemServer</code> 类的 <code>main()</code>方法。

### 执行 ZygoteInit.runSelectLoop
在 <code>startSystemServer</code> 函数中，创建 <code>system_server</code> 进程之后，<code>pid</code> 等于 0 时在该新进程中执行 <code>SystemServer.main</code>，否则回到 <code>zygote</code> 进程进行执行 <code>ZygoteInit.runSelectLoop</code>：

```java
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        FileDescriptor[] fdArray = new FileDescriptor[4];

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        int loopCount = GC_LOOP_COUNT;
        while (true) {
            int index;

            /*
             * Call gc() before we block in select().
             * It's work that has to be done anyway, and it's better
             * to avoid making every child do it.  It will also
             * madvise() any free memory as a side-effect.
             *
             * Don't call it every time, because walking the entire
             * heap is a lot of overhead to free a few hundred bytes.
             */
            if (loopCount <= 0) {
                gc();
                loopCount = GC_LOOP_COUNT;
            } else {
                loopCount--;
            }

            try {
                fdArray = fds.toArray(fdArray);
                index = selectReadable(fdArray);
            } catch (IOException ex) {
                throw new RuntimeException("Error in select()", ex);
            }

            if (index < 0) {
                throw new RuntimeException("Error in select()");
            } else if (index == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDescriptor());
            } else {
                boolean done;
                done = peers.get(index).runOnce();

                if (done) {
                    peers.remove(index);
                    fds.remove(index);
                }
            }
        }
    }
```

<code>runSelectLoop</code> 函数的逻辑比较简单，主要有两点：
1、 处理客户端的连接和请求。前面创建的 <code>LocalServerSocket</code> 对象保存 <code>sServerSocket</code>，这个 <code>socket</code> 通过 <code>selectReadable</code> 等待 <code>ActivityManagerService</code>(简写 <code>AMS</code>) 与之通信。<code>selectReadable</code>是一个<code>native</code>函数，内部调用<code>select</code>等待 <code>AMS</code> 连接，<code>AMS</code> 连接上之后就会返回: 返回值 < 0：内部发生错误；返回值 = 0：第一次连接到服务端 ；返回值 > 0：与服务端已经建立连接，并开始发送数据。每一个链接在 <code>zygote</code> 进程中使用 <code>ZygoteConnection</code> 对象表示。

2、 客户端的请求由 <code>ZygoteConnection.runOnce</code> 来处理，这个方法也抛出 <code>MethodAndArgsCaller</code> 异常，从而进入 <code>MethodAndArgsCaller.run</code> 中调用根据客户请求数据反射出的类的 <code>main</code> 方法。

```java
    private String[] readArgumentList()
    {
        int argc;

        try {
            String s = mSocketReader.readLine();

            if (s == null) {
                // EOF reached.
                return null;
            }
            argc = Integer.parseInt(s);
        } catch (NumberFormatException ex) {
            Log.e(TAG, "invalid Zygote wire format: non-int at argc");
            throw new IOException("invalid wire format");
        }

        String[] result = new String[argc];
        for (int i = 0; i < argc; i++) {
            result[i] = mSocketReader.readLine();
            if (result[i] == null) {
                // We got an unexpected EOF.
                throw new IOException("truncated request");
            }
        }

        return result;
    }

    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
        String args[];
        Arguments parsedArgs = null;
        args = readArgumentList();
        parsedArgs = new Arguments(args);

        ...
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                parsedArgs.appDataDir);
        ...
    }
```

## SystemServer 启动过程
在前面启动 SystemServer一节讲到，通过反射调用类 <code>com.android.server.SystemServer</code> main() 函数，从而开始执行 SystemServer 的初始化流程。

SystemServer.main()
```java
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```

main 函数创建一个 SystemServer 对象，调用其 run() 方法。
```java
    private void run() {
        // If a device's clock is before 1970 (before 0), a lot of
        // APIs crash dealing with negative numbers, notably
        // java.io.File#setLastModified, so instead we fake it and
        // hope that time from cell towers or NTP fixes it shortly.
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        } // 检测时间设置

        // Here we go!
        Slog.i(TAG, "Entered the Android system server!");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

        // In case the runtime switched since last boot (such as when
        // the old runtime was removed in an OTA), set the system
        // property so that it is in sync. We can't do this in
        // libnativehelper's JniInvocation::Init code where we already
        // had to fallback to a different runtime because it is
        // running as root and we need to be the system user to set
        // the property. http://b/11463182
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

        // Enable the sampling profiler.
        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            mProfilerSnapshotTimer = new Timer();
            mProfilerSnapshotTimer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
                }
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        } // 启动性能分析采样

        // Mmmmmm... more memory!
        VMRuntime.getRuntime().clearGrowthLimit();

        // The system server has to run all of the time, so it needs to be
        // as efficient as possible with its memory usage.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

        // Some devices rely on runtime fingerprint generation, so make sure
        // we've defined it before booting further.
        Build.ensureFingerprintProperty();

        // Within the system server, it is an error to access Environment paths without
        // explicitly specifying a user.
        Environment.setUserRequired(true);

        // Ensure binder calls into the system always run at foreground priority.
        BinderInternal.disableBackgroundScheduling(true);

        // Prepare the main looper thread (this thread).
        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper(); // 准备主线程循环

        // Initialize native services.
        System.loadLibrary("android_servers");
        nativeInit();

        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();

        // Initialize the system context.
        createSystemContext();

        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        // Start services.  // 启动服务
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        // For debug builds, log event loop stalls to dropbox for analysis.
        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        // Loop forever.
        Looper.loop();  // 启动线程循环，等待消息处理
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

在这个 run 方法中，主要完成三件事情，创建 system context 和 system service manager，启动一些系统服务，进入主线程消息循环。

## Zygote 的 fork 本地方法分析
接下来我们仔细分析 <code>Zygote.forkSystemServer</code> 与 <code>Zygote.forkAndSpecialize</code> 两个方法。

### forkSystemServer
```java
    private static final ZygoteHooks VM_HOOKS = new ZygoteHooks();

    public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        VM_HOOKS.postForkCommon();
        return pid;
    }
```

在调用 <code>nativeForkSystemServer</code> 创建 <code>system_server</code> 进程之前与之后，都会调用 ZygoteHooks 进行一些前置与后置处理。

#### ZygoteHooks.preFork
前置处理 <code>ZygoteHooks.preFork</code>：
```java
    public void preFork() {
        Daemons.stop();
        waitUntilAllThreadsStopped();
        token = nativePreFork();
    }
```

<code>Daemons.stop()</code>; 停止虚拟机中一些守护线程操作：如引用队列、终接器、GC等
```java
    public static void stop() {
        ReferenceQueueDaemon.INSTANCE.stop();
        FinalizerDaemon.INSTANCE.stop();
        FinalizerWatchdogDaemon.INSTANCE.stop();
        HeapTrimmerDaemon.INSTANCE.stop();
        GCDaemon.INSTANCE.stop();
    }
```

<code>waitUntilAllThreadsStopped</code> 保证被 <code>fork</code> 的进程是单线程，这样可以确保通过 <code>copyonwrite fork</code> 出来的进程也是单线程，从而节省资源。与前面提到的在新建 <code>system_server</code> 进程中调用 <code>closeServerSocket</code> 关闭 <code>sockect</code> 有异曲同工之妙。
```java
    /**
     * We must not fork until we're single-threaded again. Wait until /proc shows we're
     * down to just one thread.
     */
    private static void waitUntilAllThreadsStopped() {
        File tasks = new File("/proc/self/task");
        while (tasks.list().length > 1) {
            try {
                // Experimentally, booting and playing about with a stingray, I never saw us
                // go round this loop more than once with a 10ms sleep.
                Thread.sleep(10);
            } catch (InterruptedException ignored) {
            }
        }
    }
```

本地方法 <code>nativePreFork</code> 实现在 <code>art/runtime/native/dalvik_system_ZygoteHooks.cc</code> 中。
```cpp
    static jlong ZygoteHooks_nativePreFork(JNIEnv* env, jclass) {
      Runtime* runtime = Runtime::Current();
      CHECK(runtime->IsZygote()) << "runtime instance not started with -Xzygote";

      runtime->PreZygoteFork();

      // Grab thread before fork potentially makes Thread::pthread_key_self_ unusable.
      Thread* self = Thread::Current();
      return reinterpret_cast<jlong>(self);
    }
```

<code>ZygoteHooks_nativePreFork</code> 通过调用 <code>Runtime::PreZygoteFork</code> 来完成 <code>gc</code> 堆的一些初始化，这部分代码在 <code>art/runtime/runtime.cc</code> 中：
```cpp
    heap_ = new gc::Heap(...);
    void Runtime::PreZygoteFork() {
        heap_->PreZygoteFork();
    }
```

#### 创建 system_server 进程：
<code>nativeForkSystemServer</code> 实现在 <code>framework/base/core/jni/com_android_internal_os_Zygote.cpp</code> 中：

```cpp
    static jint com_android_internal_os_Zygote_nativeForkSystemServer(
            JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
            jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
            jlong effectiveCapabilities) {
        pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                        debug_flags, rlimits,
                        permittedCapabilities, effectiveCapabilities,
                        MOUNT_EXTERNAL_NONE, NULL, NULL, true, NULL,
                        NULL, NULL);
        if (pid > 0) {
            // The zygote process checks whether the child process has died or not.
            ALOGI("System server process %d has been created", pid);
            gSystemServerPid = pid;
            // There is a slight window that the system server process has crashed
            // but it went unnoticed because we haven't published its pid yet. So
            // we recheck here just to make sure that all is well.
            int status;
            if (waitpid(pid, &status, WNOHANG) == pid) {
                ALOGE("System server process %d has died. Restarting Zygote!", pid);
                RuntimeAbort(env);
            }
        }
        return pid;
    }
```

它转调 <code>ForkAndSpecializeCommon</code> 来创建新进程，并确保 <code>system_server</code> 创建成功，若不成功便成仁：重启 <code>zygote</code>，因为没有 <code>system_server</code> 就干不了什么事情。<code>ForkAndSpecializeCommon</code> 实现如下：

```cpp
    static const char kZygoteClassName[] = "com/android/internal/os/Zygote";
    gZygoteClass = (jclass) env->NewGlobalRef(env->FindClass(kZygoteClassName));
    gCallPostForkChildHooks = env->GetStaticMethodID(gZygoteClass, "callPostForkChildHooks",
                                           "(ILjava/lang/String;)V");

    // Utility routine to fork zygote and specialize the child process.
    static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                     jint debug_flags, jobjectArray javaRlimits,
                     jlong permittedCapabilities, jlong effectiveCapabilities,
                     jint mount_external,
                     jstring java_se_info, jstring java_se_name,
                     bool is_system_server, jintArray fdsToClose,
                     jstring instructionSet, jstring dataDir)
    {
        SetSigChldHandler();

        pid_t pid = fork();

        if (pid == 0) {
            // The child process.
            ...
            rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
            ...
            UnsetSigChldHandler();
            ...
            env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                          is_system_server ? NULL : instructionSet);
        }
        else if (pid > 0) {
            // the parent process
        }

        return pid;
    }
```

<code>ForkAndSpecializeCommon</code> 首先设置子进程异常处理<code>handler</code>，然后 </ode>fork</code> 新进程，在新进程中设置 <code>SELinux</code>，并清除它的子进程异常处理 <code>handler</code>，然后调用 <code>Zygote.callPostForkChildHooks</code> 方法。

```java
    private static void callPostForkChildHooks(int debugFlags, String instructionSet) {
        long startTime = SystemClock.elapsedRealtime();
        VM_HOOKS.postForkChild(debugFlags, instructionSet);
        checkTime(startTime, "Zygote.callPostForkChildHooks");
    }
```

<code>callPostForkChildHooks</code> 又转调 <code>ZygoteHooks.postForkChild</code> :
```java
    public void postForkChild(int debugFlags, String instructionSet) {
        nativePostForkChild(token, debugFlags, instructionSet);
    }
```

本地方法 <code>nativePostForkChild</code> 又进到 <code>dalvik_system_ZygoteHooks.cc</code> 中：
```cpp
    static void ZygoteHooks_nativePostForkChild(JNIEnv* env, jclass, jlong token, jint debug_flags,
                                            jstring instruction_set) {
        Thread* thread = reinterpret_cast<Thread*>(token);
        // Our system thread ID, etc, has changed so reset Thread state.
        thread->InitAfterFork();
        EnableDebugFeatures(debug_flags);

        if (instruction_set != nullptr) {
            ScopedUtfChars isa_string(env, instruction_set);
            InstructionSet isa = GetInstructionSetFromString(isa_string.c_str());
            Runtime::NativeBridgeAction action = Runtime::NativeBridgeAction::kUnload;
            if (isa != kNone && isa != kRuntimeISA) {
                action = Runtime::NativeBridgeAction::kInitialize;
            }
            Runtime::Current()->DidForkFromZygote(env, action, isa_string.c_str());
        } else {
            Runtime::Current()->DidForkFromZygote(env, Runtime::NativeBridgeAction::kUnload, nullptr);
        }
    }
```

<code>thread->InitAfterFork()</code>; 实现在 <code>art/runtime/thread.cc</code> 中，设置新进程主线程的线程<code>id</code>： <code>tid</code>。<code>DidForkFromZygote</code> 实现在 <code>Runtime.cc</code> 中：
```cpp
    void Runtime::DidForkFromZygote(JNIEnv* env, NativeBridgeAction action, const char* isa) {
        is_zygote_ = false;

        switch (action) {
        case NativeBridgeAction::kUnload:
            UnloadNativeBridge();
            break;

        case NativeBridgeAction::kInitialize:
            InitializeNativeBridge(env, isa);
            break;
        }

        // Create the thread pool.
        heap_->CreateThreadPool();

        StartSignalCatcher();

        // Start the JDWP thread. If the command-line debugger flags specified "suspend=y",
        // this will pause the runtime, so we probably want this to come last.
        Dbg::StartJdwp();
    }
```

首先根据 <code>action</code> 参数来卸载或转载用于跨平台桥接用的库。然后启动 <code>gc</code> 堆的线程池。<code>StartSignalCatcher</code> 设置信号处理 <code>handler</code>，其代码在 <code>signal_catcher.cc</code> 中。

#### ZygoteHooks.postForkCommon
后置处理 <code>ZygoteHooks.postForkCommon</code>：
```java
    public void postForkCommon() {
        Daemons.start();
    }
```

<code>postForkCommon</code> 转调 <code>Daemons.start</code>，以初始化虚拟机中引用队列、终接器以及 <code>gc</code> 的守护线程。
```java
    public static void start() {
        ReferenceQueueDaemon.INSTANCE.start();
        FinalizerDaemon.INSTANCE.start();
        FinalizerWatchdogDaemon.INSTANCE.start();
        HeapTrimmerDaemon.INSTANCE.start();
        GCDaemon.INSTANCE.start();
    }
```

### forkAndSpecialize
<code>Zygote.forkAndSpecialize</code> 方法
```java
    public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
          String instructionSet, String appDataDir) {
        long startTime = SystemClock.elapsedRealtime();
        VM_HOOKS.preFork();
        checkTime(startTime, "Zygote.preFork");
        int pid = nativeForkAndSpecialize(
                  uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                  instructionSet, appDataDir);
        checkTime(startTime, "Zygote.nativeForkAndSpecialize");
        VM_HOOKS.postForkCommon();
        checkTime(startTime, "Zygote.postForkCommon");
        return pid;
    }
```

前置处理与后置处理与 <code>forkSystemServer</code> 中一样的，这里就跳过不讲了。本地方法 <code>nativeForkAndSpecialize</code> 实现在 <code>framework/base/core/jni/com_android_internal_os_Zygote.cpp</code> 中：

```cpp
static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits,
        jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jstring instructionSet, jstring appDataDir) {
    // Grant CAP_WAKE_ALARM to the Bluetooth process.
    jlong capabilities = 0;
    if (uid == AID_BLUETOOTH) {
        capabilities |= (1LL << CAP_WAKE_ALARM);
    }

    return ForkAndSpecializeCommon(env, uid, gid, gids, debug_flags,
            rlimits, capabilities, capabilities, mount_external, se_info,
            se_name, false, fdsToClose, instructionSet, appDataDir);
}
```

这个函数与 <code>com_android_internal_os_Zygote_nativeForkSystemServer</code> 非常类似，只不过少了一个确保子进程创建成功的步骤。
