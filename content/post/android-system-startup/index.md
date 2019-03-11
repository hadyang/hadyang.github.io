---
title: Android系统开机过程分析
date: 2016-05-25
categories:
  - Android

tags:
    - 源码
---

当我们开机时，首先是启动Linux内核，在Linux内核中首先启动的是init进程，这个进程会去读取配置文件system\core\rootdir\init.rc配置文件，这个文件中配置了Android系统中第一个进程Zygote进程。Android中其他所有进程都是通过fork Zygote进程得来的。

<!--more-->

在这里我只是先简单的分析代码中Android系统启动的大体过程：）

- Step1 -- init.rc

```
...
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    socket zygote stream 666
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media

...
```

- Step2 -- init.c

```C
void service_start(struct service *svc, const char *dynamic_args)
{
    pid_t pid;

    ...

    pid = fork();//创建子进程

    if (pid == 0) {//进入这个分支说明函数是在子进程中返回的，新的子进程就是zygote
        struct socketinfo *si;

        ...

        //创建通信使用的socket
        for (si = svc->sockets; si; si = si->next) {
            int s = create_socket(si->name,
                                  !strcmp(si->type, "dgram") ?
                                  SOCK_DGRAM : SOCK_STREAM,
                                  si->perm, si->uid, si->gid);
            if (s >= 0) {
                publish_socket(si->name, s);//发布到系统中
            }
        }

        ...

        //检查启动参数，如果没有就加载zygote应用程序文件
        if (!dynamic_args) {
            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
            }
        } else {
        //否则就将参数合并到arg_ptrs数组中，在加载zygote应用程序文件
            char *arg_ptrs[SVC_MAXARGS+1];
            int arg_idx = svc->nargs;
            char *tmp = strdup(dynamic_args);
            char *next = tmp;
            char *bword;

            /* Copy the static arguments */
            memcpy(arg_ptrs, svc->args, (svc->nargs * sizeof(char *)));

            while((bword = strsep(&next, " "))) {
                arg_ptrs[arg_idx++] = bword;
                if (arg_idx == SVC_MAXARGS)
                    break;
            }
            arg_ptrs[arg_idx] = '\0';
            //Step3
            execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
        }
        _exit(127);
    }
}
```

- Step3 -- App_main.cpp

```C++
int main(int argc, char* const argv[])
{
    ...

    //创建AppRuntime，android运行时环境
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));

    ...
    int i;
    ...
    ++i;  // Skip unused "parent dir" argument.

    //检查启动参数
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;

            //zygote进程名字
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    ...

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        //设置zygote进程名
        set_process_name(niceName.string());
    }

    //zygote进程进入第一个分支
    if (zygote) {
        InitializeNativeLoader();
        //Step3
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

- Step3 -- AndroidRuntime.start

AppRuntime.start继承自AndroidRuntime.start

```C++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{

    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    //启动虚拟机
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    //在虚拟机中注册JNI方法
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    ...
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
        ALOGE("JavaVM unable to find main() in '%s'\n", className);
    } else {
        //进入ZygoteInit.main
        //Step4
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
    }

}
```

- Step4 -- ZygoteInit.main

```Java
//com.android.internal.os.ZygoteInit.java
public static void main(String argv[]) {
        try {
            ...
            //创建Server端socket，用来等待AMS请求
            //Step4.1
            registerZygoteSocket(socketName);

            ...
            //这里startSystemServer==true
            if (startSystemServer) {
                //Step4.2
                startSystemServer(abiList, socketName);
            }
            ...
            //等待AMS请求
            //Step4.3
            runSelectLoop(abiList);
            ...
        }
    }
```

- Step4.1 -- ZygoteInit.registerZygoteSocket

```Java
private static final String ANDROID_SOCKET_PREFIX = "ANDROID_SOCKET_";

private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            //获取socket描述字符串
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                //获取socket文件描述符
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                //根据文件描述符创建server端socket
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```

- STep4.2 -- ZygoteInit.startSystemServer

```Java
private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        ...
        /* Hardcoded command line to start the system server */
        //服务启动参数
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;
        try {
            ...
            /* Request to fork the system server process */
            //通过fork创建服务
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
            //Step5
            handleSystemServerProcess(parsedArgs);
        }
        return true;
    }
```

- Step4.3 -- ZygoteInit.runSelectLoop

```Java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        //循环等待请求
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    //收到请求创建进程
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```

- Step5 ZygoteInit.handleSystemServerProcess

```Java
private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {

        //由于system进程复制了zygote进程的地址空间，因此它也会获得zygote进程在启动过程中创建的socket，
        //由于system进程不需要这个socket，所以关闭
        closeServerSocket();
        ...
        //Step6
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
}
```

- Step6 -- RuntimeInit.zygoteInit

```Java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        //初始化时区，键盘布局等通用信息      
        commonInit();
        //在system进程中启动一个binder线程池
        nativeZygoteInit();
        //Step7
        applicationInit(targetSdkVersion, argv, classLoader);
}
```

- Step7 -- RuntimeInit.applicationInit

```Java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        ...
        // Remaining arguments are passed to the start class's static main
        //进入com.android.server.SystemServer.main
        //Step8
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```

- Step8 -- SystemServer.main

```Java
//com.android.server.SystemServer.java
public static void main(String[] args) {
        //Step9
        new SystemServer().run();
}
```

- Step9 -- SystemServer.run

```Java
private void run() {
        try {
            ...

            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");

            ...

            // Initialize native services.
            System.loadLibrary("android_servers");

            ...

            // Initialize the system context.
            createSystemContext();

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);

            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }

        // Start services.
        try {
            //Step9.1
            startBootstrapServices();
            //Step9.2
            startCoreServices();
            //启动其他服务，包括网络、彩信等等，还启动了Launcher
            startOtherServices();
        } catch (Throwable ex) {
            ...
            throw ex;
        } finally {
            ...
        }
        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

- Step9.1 -- SystemServer.startBootstrapServices

```Java
private void startBootstrapServices() {
        //启动Installer Server
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // 启动AMS
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        //启动PMS
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        ...

        // Manages LEDs and display backlight so we need it to bring up the display.
        mSystemServiceManager.startService(LightsService.class);

        // Display manager is needed to provide display metrics before package manager
        // starts up.
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // Start the package manager.
        traceBeginAndSlog("StartPackageManagerService");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);

        // Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();

        //启动传感器服务
        startSensorService();
    }

```

- Step9.2 -- SystemServer.startCoreServices

```Java
private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);

        // Tracks application usage stats.
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        // Update after UsageStatsService is available, needed before performBootDexOpt.
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```
