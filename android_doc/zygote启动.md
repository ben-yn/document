# Zygote进程启动过程
zygote进程是由init进程启动的,在android中所有应用进程以及系统服务进程都是由zygote fork()出来的,并且system_server是zygote的嫡长子,
当zygote完成启动立启动system_server,这是在zygote.rc中的参数`--start-system-serve`决定的.  
在`aosp/frameworks/base/cmds/app_process/app_main.cpp`的main函数里`runtime.start("com.android.internal.os.ZygoteInit", args, zygote);`,
进入java层的`aosp/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`的main函数  
在进行分析之前,先分享一张图片,图片来自:


```cpp
    public static void main(String argv[]) {
        /// M: GMO Zygote64 on demand @{
        DEBUG_ZYGOTE_ON_DEMAND =
            SystemProperties.get(PROP_ZYGOTE_ON_DEMAND_DEBUG).equals("1");
        if (DEBUG_ZYGOTE_ON_DEMAND) {
            Log.d(TAG, "ZygoteOnDemand: Zygote ready = " + sZygoteReady);
        }
        //socketName用于区分不预加载类操作,zygote为一类,其他为另一类
        String socketName = "zygote";
        /// M: GMO Zygote64 on demand @}

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        try {
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygoteInit");
            RuntimeInit.enableDdms();
            /// M: Added for BOOTPROF
            //MTPROF_DISABLE = "1".equals(SystemProperties.get("ro.mtprof.disable"));
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            boolean startSystemServer = false;
            String abiList = null;
            //将配置文件参数转换为代码里的对应的变量值
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    //zygote的rc文件中配置了该参数,表示zygote需要启动systemserver
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    //可能传的参数有修改socketName的值
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
            //创建一个socket接口,用于接受AMS的fork新进程的请求
            registerZygoteSocket(socketName);
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            /// M: Added for BOOTPROF
            addBootEvent("Zygote:Preload Start");

            /// M: GMO Zygote64 on demand @{
            /// M: Added for Zygote preload control @{
            preloadByName(socketName);
            /// @}
            /// M: GMO Zygote64 on demand @}

            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PostZygoteInitGC");
            gcAndFinalize();
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            // Zygote process unmounts root storage spaces.
            Zygote.nativeUnmountStorageOnInit();

            /// M: Added for BOOTPROF
            addBootEvent("Zygote:Preload End");

            ZygoteHooks.stopZygoteNoThreadCreation();

            /// N: Move MPlugin init after preloading due to "Zygote: No Thread Creation Issues". @{
            boolean preloadMPlugin = false;
            if (!sZygoteOnDemandEnabled) {
                preloadMPlugin = true;
            } else {
                if ("zygote".equals(socketName)) {
                    preloadMPlugin = true;
                }
            }
            if (preloadMPlugin) {
                Log.i(TAG1, "preloadMappingTable() -- start ");
                PluginLoader.preloadPluginInfo();
                Log.i(TAG1, "preloadMappingTable() -- end ");
            }
            /// @}

            if (startSystemServer) {
                //启动systemserver组件
                startSystemServer(abiList, socketName);
            }

            /// M: GMO Zygote64 on demand @{
            sZygoteReady = true;
            if (DEBUG_ZYGOTE_ON_DEMAND) {
                Log.d(TAG, "ZygoteOnDemand: Zygote ready = " + sZygoteReady +
                    ", socket name: " + socketName);
            }
            /// M: GMO Zygote64 on demand @}

            Log.i(TAG, "Accepting command socket connections");
            //进入一个无限循环,在前面创建的socket上等待AMS的请求
            runSelectLoop(abiList);

            /// M: GMO Zygote64 on demand @{
            zygoteStopping("ZygoteOnDemand: End of runSelectLoop", socketName);
            /// M: GMO Zygote64 on demand @}

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {//通过抓取MethodAndArgsCaller异常去启动systenserver
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }

        /// M: GMO Zygote64 on demand @{
        zygoteStopping("ZygoteOnDemand: End of main function", socketName);
        /// M: GMO Zygote64 on demand @}
    }
```
zygote main()里的主要工作:  

- 创建用于接受AMS请求的socket  
- 加载类和资源  
- 启动systemserver组件  
- 通过runSelectLoop()进入无限循环,用于等待socket端口  

接下来就是分四大部分来分析,这部分里面启动systemserver组件和后续的文章相关.

***
分析创建socket的函数registerZygoteSocket:  
```cpp
    /**
     * Registers a server socket for zygote command connections
     *
     * @throws RuntimeException when open fails
     */
    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            //这里的socketName默认是zygote,也可以通过传参数修改,在main函数里有先关的说明
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                //这就是在获取zygote在native端在/dev/socket下创建的zygote这个socket文件的位置
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
***
分析加载类和资源,在zygote.java的main函数里:  
```
        preloadByName(socketName);
```
preloadByName:  
```
    /// M: GMO Zygote64 on demand @{
    /// M: Added for Zygote preload control @{
    static void preloadByName(String name) {
        if (sZygoteOnDemandEnabled) {//sZygoteOnDemandEnabled是MTK平台的参数,可以忽略不管,一般都是1
            if ("zygote".equals(name)) {//是zygote进程
                preload();
            } else {//如果socketName的值不是默认的zygote,由于传了参数,去修改它走走这个分支
                preloadSecondary();
            }
        } else {
            preload();
        }
    }
```
preload():  
```
    static void preload() {
        Log.d(TAG, "begin preload");
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "BeginIcuCachePinning");
        //才疏学浅...
        beginIcuCachePinning();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadClasses");
        //加载preloaded-classes资源,文件位置:frameworks/base/preloaded-classes
        preloadClasses();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadResources");
        //加载共享资源
        preloadResources();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadOpenGL");
        //加载OpenGL()资源
        preloadOpenGL();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        endIcuCachePinning();
        warmUpJcaProviders();
        Log.d(TAG, "end preload");
    }
```
preload方法会加载三个部分，preloadClasses，preloadResources和preloadOpenGL。简单分析preloadClasses,其他两个不熟悉...  
**加载preloaded-classes资源:**  
```cpp
    /**
     * Performs Zygote process initialization. Loads and initializes
     * commonly used classes.
     *
     * Most classes only cause a few hundred bytes to be allocated, but
     * a few will allocate a dozen Kbytes (in one case, 500+K).
     */
    private static void preloadClasses() {
        final VMRuntime runtime = VMRuntime.getRuntime();

        InputStream is;
        try {
            //预加载类的信息存储在PRELOADED_CLASSES文件中
            //打开PRELOADED_CLASSES文件
            is = new FileInputStream(PRELOADED_CLASSES);
        } catch (FileNotFoundException e) {
            Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");
            return;
        }

        Log.i(TAG, "Preloading classes...");
        long startTime = SystemClock.uptimeMillis();

        // Drop root perms while running static initializers.
        final int reuid = Os.getuid();
        final int regid = Os.getgid();

        // We need to drop root perms only if we're already root. In the case of "wrapped"
        // processes (see WrapperInit), this function is called from an unprivileged uid
        // and gid.
        boolean droppedPriviliges = false;
        if (reuid == ROOT_UID && regid == ROOT_GID) {
            try {
                Os.setregid(ROOT_GID, UNPRIVILEGED_GID);
                Os.setreuid(ROOT_UID, UNPRIVILEGED_UID);
            } catch (ErrnoException ex) {
                throw new RuntimeException("Failed to drop root", ex);
            }

            droppedPriviliges = true;
        }

        // Alter the target heap utilization.  With explicit GCs this
        // is not likely to have any effect.
        float defaultUtilization = runtime.getTargetHeapUtilization();
        runtime.setTargetHeapUtilization(0.8f);

        /// M: Added for BOOTPROF
        int count = 0;
        try {
            BufferedReader br
                = new BufferedReader(new InputStreamReader(is), 256);

            String line;
            while ((line = br.readLine()) != null) {
                // Skip comments and blank lines.\
                //是按行解析PRELOADED_CLASSES
                line = line.trim();
                if (line.startsWith("#") || line.equals("")) {//跳过#开头或者空行
                    continue;
                }

                Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadClass " + line);
                try {
                    if (false) {
                        Log.v(TAG, "Preloading " + line + "...");
                    }
                    // Load and explicitly initialize the given class. Use
                    // Class.forName(String, boolean, ClassLoader) to avoid repeated stack lookups
                    // (to derive the caller's class-loader). Use true to force initialization, and
                    // null for the boot classpath class-loader (could as well cache the
                    // class-loader of this class in a variable).
                    //通过反射加载类
                    Class.forName(line, true, null);
                    count++;
                } catch (ClassNotFoundException e) {
                    Log.w(TAG, "Class not found for preloading: " + line);
                } catch (UnsatisfiedLinkError e) {
                    Log.w(TAG, "Problem preloading " + line + ": " + e);
                } catch (Throwable t) {
                    Log.e(TAG, "Error preloading " + line + ".", t);
                    if (t instanceof Error) {
                        throw (Error) t;
                    }
                    if (t instanceof RuntimeException) {
                        throw (RuntimeException) t;
                    }
                    throw new RuntimeException(t);
                }
                Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
            }

            Log.i(TAG, "...preloaded " + count + " classes in "
                    + (SystemClock.uptimeMillis()-startTime) + "ms.");
        } catch (IOException e) {
            Log.e(TAG, "Error reading " + PRELOADED_CLASSES + ".", e);
        } finally {
            IoUtils.closeQuietly(is);
            // Restore default.
            runtime.setTargetHeapUtilization(defaultUtilization);

            // Fill in dex caches with classes, fields, and methods brought in by preloading.
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadDexCaches");
            runtime.preloadDexCaches();
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            // Bring back root. We'll need it later if we're in the zygote.
            if (droppedPriviliges) {
                try {
                    Os.setreuid(ROOT_UID, ROOT_UID);
                    Os.setregid(ROOT_GID, ROOT_GID);
                } catch (ErrnoException ex) {
                    throw new RuntimeException("Failed to restore root", ex);
                }
            }
            /// M: Added for BOOTPROF @{
            addBootEvent("Zygote:Preload " + count + " classes in " +
            (SystemClock.uptimeMillis() - startTime) + "ms");
            /// @}
        }
    }
```
PRELOADED_CLASSES文件在源码中位于frameworks/base/preloaded-classes,手机中位于system/etc/preloaded-classes
函数的作用就是按行加载preloaded-classes里面的类,preloaded-classes有4000多行,还是很耗时间的,在android启动中
这也是一个耗时操作.关于preloaded-classes的产生以及深入学习,以后再做总结.
**加载共享资源:**  
```cpp
    /**
     * Load in commonly used resources, so they can be shared across
     * processes.
     *
     * These tend to be a few Kbytes, but are frequently in the 20-40K
     * range, and occasionally even larger.
     */
    private static void preloadResources() {
        final VMRuntime runtime = VMRuntime.getRuntime();

        try {
            mResources = Resources.getSystem();
            mResources.startPreloading();
            if (PRELOAD_RESOURCES) {
                Log.i(TAG, "Preloading resources...");

                long startTime = SystemClock.uptimeMillis();
                TypedArray ar = mResources.obtainTypedArray(
                        com.android.internal.R.array.preloaded_drawables);
                int N = preloadDrawables(ar);
                ar.recycle();
                Log.i(TAG, "...preloaded " + N + " resources in "
                        + (SystemClock.uptimeMillis()-startTime) + "ms.");
                addBootEvent("Zygote:Preload " + N + " obtain resources in " +
                                (SystemClock.uptimeMillis() - startTime) + "ms");

                startTime = SystemClock.uptimeMillis();
                ar = mResources.obtainTypedArray(
                        com.android.internal.R.array.preloaded_color_state_lists);
                N = preloadColorStateLists(ar);
                ar.recycle();
                Log.i(TAG, "...preloaded " + N + " resources in "
                        + (SystemClock.uptimeMillis()-startTime) + "ms.");

                if (mResources.getBoolean(
                        com.android.internal.R.bool.config_freeformWindowManagement)) {
                    startTime = SystemClock.uptimeMillis();
                    ar = mResources.obtainTypedArray(
                            com.android.internal.R.array.preloaded_freeform_multi_window_drawables);
                    N = preloadDrawables(ar);
                    ar.recycle();
                    Log.i(TAG, "...preloaded " + N + " resource in "
                            + (SystemClock.uptimeMillis() - startTime) + "ms.");
                }

                /// M: Added for BOOTPROF @{
                addBootEvent("Zygote:Preload " + N + " resources in " +
                (SystemClock.uptimeMillis() - startTime) + "ms");
                /// @}
            }
            mResources.finishPreloading();
        } catch (RuntimeException e) {
            Log.w(TAG, "Failure preloading resources", e);
        }
    }
```
**加载OpenGL()资源:**  
```cpp
    private static void preloadOpenGL() {
        /// N: Added for Boot time profiling @{
        Log.i(TAG, "Preloading OpenGL...");
        /// @}
        if (!SystemProperties.getBoolean(PROPERTY_DISABLE_OPENGL_PRELOADING, false)) {
            EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
        }
        /// N: Added for Boot time profiling @{
        Log.i(TAG, "Preloading OpenGL --- End");
        /// @}
    }
```
总之,这是preload是一个启动耗时操作.那么启动时预加载这些类,是为了在开机后使用到这些类的时候速度更快,更加流畅.  
***
启动systemserver进程的操作不是直接执行systemserver main函数,而是通过抛出异常,抓取异常的方式实现的,这样做的好处就是:
清理应用程序栈中ZygoteInit.main以上的函数栈帧，以实现当相应的main函数退出时，能直接退出整个应用程序。
当当前的main退出后，就会退回到MethodAndArgsCaller.run而这个函数直接就退回到ZygoteInit.main函数，
而ZygoteInit.main也无其他的操作，直接退出了函数，这样整个应用程序将会完全退出。
分析startSystemServer(abiList, socketName):  
```cpp

    /**
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_IPC_LOCK,
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
        /* Containers run without this capability, so avoid setting it in that case */
        if (!SystemProperties.getBoolean(PROPERTY_RUNNING_IN_CONTAINER, false)) {
            capabilities |= posixCapabilitiesAsBits(OsConstants.CAP_BLOCK_SUSPEND);
        }
        /* Hardcoded command line to start the system server */
        //该数组中包含了启动该进程的相关信息
        String args[] = {
            //uid,gid,用户组id设置参数
            "--setuid=1000",
            "--setgid=1000",
            /// M: ANR mechanism for system_server add shell(2000) group to access
            ///    /sys/kernel/debug/tracing/tracing_on
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,2000," +
                "3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            //最后用于设置SystemServer进程名字
            "--nice-name=system_server",
            "--runtime-args",
            //装载的第一个Java类
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            //通过native方法forkSystemServer,最终会调用fork(作用类似其他进程的fork函数:forAndSpecilize(),感受到SystemServer的不同待遇了吧~~~)
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
        if (pid == 0) {//进入子进程,从这开始就运行在system_server进程了
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            //进程产生了,接着干活
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }

```
handleSystemServerProcess主要干的事:关闭Socket;抛出ZygoteInit.MethodAndArgsCaller异常
handleSystemServerProcess(parsedArgs):  
```cpp
    /**
     * Finish remaining work for the newly forked system server process.
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
         throws ZygoteInit.MethodAndArgsCaller {//原来是这里抛出了ZygoteInit.MethodAndArgsCaller异常
        //由于由Zygote进程创建的子进程会继承Zygote进程创建的Socket文件描述符，而这里的子进程又不会用到它，因此，这里就调用closeServerSocket函数来关闭它
        closeServerSocket();
        //英语注释解释的很清楚~~~
        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            performSystemServerDexOpt(systemServerClasspath);
        }

        if (parsedArgs.invokeWith != null) {
            String[] args = parsedArgs.remainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
            }

            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = createSystemServerClassLoader(systemServerClasspath,
                                                   parsedArgs.targetSdkVersion);

                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
            //进一步执行systemserver组件启动
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
    }
```
进入`aosp/frameworks/base/core/java/com/android/internal/os/RuntimeInit.java`,RuntimeInit.zygoteInit:  
```cpp
    /**
     * The main function called when started through the zygote process. This
     * could be unified with main(), if the native code in nativeFinishInit()
     * were rationalized with Zygote startup.<p>
     *
     * Current recognized args:
     * <ul>
     *   <li> <code> [--] &lt;start class name&gt;  &lt;args&gt;
     * </ul>
     *
     * @param targetSdkVersion target SDK version
     * @param argv arg strings
     */
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();

        commonInit();
        //Binder进程间通信机制初始化，完成后,进程中的Binder对象就可以进行进程间通信了
        nativeZygoteInit();
        //接着初始化,异常就是该这函数抛出
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```
applicationInit:  
```cpp
    private staticZygoteInit.MethodAndArgsCaller void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //真正抛出ZygoteInit.MethodAndArgsCaller的函数露脸了~
        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```
invokeStaticMain:  
```cpp
    /**
     * Invokes a static "main(argv[]) method on class "className".
     * Converts various failing exceptions into RuntimeExceptions, with
     * the assumption that they will then cause the VM instance to exit.
     *
     * @param className Fully-qualified class name
     * @param argv Argument vector for main()
     * @param classLoader the classLoader to load {@className} with
     */
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            //....
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            //...
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        //多么清楚的一段注释
        //注意将m,argv作为参数传入
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```
注释都说明这个ZygoteInit.MethodAndArgsCaller是让 ZygoteInit.main()去catch的,那么就回到ZygoteInit.main()代码中catch的位置:  
```cpp
        } catch (MethodAndArgsCaller caller) {//通过抓取MethodAndArgsCaller异常去启动systenserver
            caller.run();
        }
```
执行了caller.run(),这个caller就是MethodAndArgsCaller类,MethodAndArgsCaller在ZygoteInit.javaZygoteInit中,看看MethodAndArgsCaller.run()的实现:  
```cpp
        public void run() {
            try {
                //进入aosp/frameworks/base/services/java/com/android/server/SystemServer.java main()方法
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
```
终于到了SystemServer.java里面,来看看这个main函数长什么样子:  
```cpp
    static void main(String[] args) {
        new SystemServer().run();
    }
```
这里就到此位置,接下来在main()方法之后的分析,在另一篇文章中分析.
***
分析完启动systemserver组件,接着分析zygote的下一个工作,创建无线循环,在socket上等待AMS的请求.分析runSelectLoop(abiList):  
```cpp
    /**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     *
     * @throws MethodAndArgsCaller in a child process when a main() should
     * be executed.
     */
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

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
                    //acceptCommandPeer函数会和客户端建立一个socket连接
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    //peers.get(index)得到的是一个ZygoteConnection对象,表示一个Socket连接
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
由于代码太长.就不全贴出来,接着就会在ZygoteConnection.runOnce()方法中创建进程,
```cpp
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
```
除了system_server这样的进程外,其他的应用进程启动都是使用该方法Zygote.forkAndSpecialize fork出新进程.   
***









