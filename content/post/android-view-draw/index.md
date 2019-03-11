---
title: View的绘制过程
date: 2016-06-01
categories:
  - Android

tags:
    - 源码
---

上次我们分析了Activity的启动过程，我们已经分析到ActivityThread.main函数，接下来两篇文章我们分析Activity的绘制、时间分发过程。以前，Activity的绘制在我的印象中就measure、lyaout、draw三个过程，对于这三个过程具体的调用过程和细节知之甚少，下面我们就来分析一下Activity的绘制过程。

<!--more-->

- Step1 -- ActivityThread.main()

```Java
//android.app.ActivityThread.java
 public static final void main(String[] args) {
        ...
        //初始化主线程消息循环
        Looper.prepareMainLooper();

        //由于ApplicationThread是ActivityThread的成员变量，所以同时也new ApplicationThread
        ActivityThread thread = new ActivityThread();

        //step2
        thread.attach(false);

        Looper.loop();

        ...
        thread.detach();
        ...
    }
}
```

- Step2 -- attach(false)

```Java
//android.app.ActivityThread.java
private final void attach(boolean system) {
        sThreadLocal.set(this);
        mSystemThread = system;
        //非系统应用
        if (!system) {
            ViewRoot.addFirstDrawHandler(new Runnable() {
                public void run() {
                    ensureJitEnabled();//确保开启JNI
                }
            });
           ...
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            IActivityManager mgr = ActivityManagerNative.getDefault();//获取ActivityManagerService的Binder代理
            try {
                //Step3
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
            }
        }
        //系统应用
        ...

        //添加OnLowMemory等事件回调
        ViewRoot.addConfigCallback(...);
```

- Step3 -- attachApplication(mAppThread)

```Java
//com.android.server.am.ActivityManagerService.java
public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();//获取调用进程PID
            final long origId = Binder.clearCallingIdentity();
            //Step4
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

- Step4 -- attachApplicationLocked()

```Java
//com.android.server.am.ActivityManagerService.java
private final boolean attachApplicationLocked(IApplicationThread thread,int pid) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }

        ...
        // 初始化ProcessRecord，这个变量是存储在AMS中的
        app.thread = thread;
        app.curAdj = app.setAdj = -100;
        app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;
        app.setSchedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;
        app.forcingToForeground = null;
        app.foregroundServices = false;
        app.debugging = false;

        ...
          //命令客户进程运行指定的Activity所在的APK文件，这个函数最终会调用ActivityThread.handleBindApplication
          //Step5
          thread.bindApplication(...)
        ...

        //Step6
        if (realStartActivityLocked(hr, app, true, true)) {
                        didSomething = true;
         }

        // Find any services that should be running in this process...
        ...

        // Check if the next broadcast receiver is in this process...
        ...

        // Check whether the next backup agent is in this process...
        ...
```
- Step5 -- handleBindApplication(AppBindData)

```Java
//android.os.ActivityThread.java
private final void handleBindApplication(AppBindData data) {
        ...

        // 新建Application，Step5.1
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
```

- Step5.1 -- makeApplication(boolean forceDefaultAppClass,Instrumentation instrumentation)

```Java
//android.os.ActivityThread.java
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {
            if (mApplication != null) {
                return mApplication;
            }

            Application app = null;

            String appClass = mApplicationInfo.className;

            try {
                java.lang.ClassLoader cl = getClassLoader();
                ContextImpl appContext = new ContextImpl();
                appContext.init(this, null, mActivityThread);
                //创建Application对象，并将appContext attach到Application对象上
                app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
                appContext.setOuterContext(app);
            }
            mActivityThread.mAllApplications.add(app);
            mApplication = app;

            return app;
        }

```

- Step6 -- realStartActivityLocked()

```Java
//com.android.server.am.ActivityManagerService.java
private final boolean realStartActivityLocked(HistoryRecord r,ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
        ...
        r.app = app;
        ...
        //Step7
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r,
                    System.identityHashCode(r),
                    r.info, r.icicle, results, newIntents, !andResume,
                    isNextTransitionForward());
        ...
```
- Step7 -- scheduleLaunchActivity

通过消息循环最终会调用handleLaunchActivity

```Java
//android.os.ActivityThread.java
private final void handleLaunchActivity(ActivityRecord r, Intent customIntent) {
        ...
        //Step8
        Activity a = performLaunchActivity(r, customIntent);
        ...

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            //Step9
            handleResumeActivity(r.token, false, r.isForward);
         }
        ...
}
```

- Step8 -- performLaunchActivity

```Java
private final Activity performLaunchActivity(ActivityRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        ...
        //设置启动Activity信息
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        Activity activity = null;
       ...
        //通过反射生成Activity
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        r.intent.setExtrasClassLoader(cl);
        ...

         Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                ContextImpl appContext = new ContextImpl();
                appContext.init(r.packageInfo, r.token, this);
                appContext.setOuterContext(activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mConfiguration);

                //Step8.1
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstance,
                        r.lastNonConfigurationChildInstances, config);

               //调用Activity.OnCreate函数
                mInstrumentation.callActivityOnCreate(activity, r.state);

                if (!r.activity.mFinished) {
                    //调用Activity.OnStart函数
                    activity.performStart();
                    r.stopped = false;
                }
        return activity;
    }
```

- Step8.1 -- attach

```Java
//android.app.Activity.java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            Object lastNonConfigurationInstance,
            HashMap<String,Object> lastNonConfigurationChildInstances,
            Configuration config) {

        //将Context attach到activity
        attachBaseContext(context);

        //新建PhoneWindow对象
        mWindow = new PhoneWindow(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);

       ...       
       //设置UI线程
       mUiThread = Thread.currentThread();

        //设置Activity相关属性
        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstance = lastNonConfigurationInstance;
        mLastNonConfigurationChildInstances = lastNonConfigurationChildInstances;

       ...    
}

```

- Step9 -- handleResumeActivity

```Java
//android.os.ActivityThread.java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward) {

        //这里会调用performRestart，通过状态来判断是否回调Activity.OnRestart()
        ActivityRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            final Activity a = r.activity;
            ...

            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;

                    //Step10
                    wm.addView(decor, l);
                }
                ...
    }
```

- Step10 -- WindowManagerGlobal.addView

WindowManagerImpl.addView最终调用WindowManagerGlobal.addView

```Java
//android.view.WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...

        ViewRootImpl root;

        获取ViewRootImpl
        ...

         root = new ViewRootImpl(view.getContext(), display);
          view.setLayoutParams(wparams);
          mViews.add(view);
          mRoots.add(root);
          mParams.add(wparams);

        // do this last because it fires off messages to start doing things
         root.setView(view, wparams, panelParentView);

    }
```

- Step11 -- ViewRootImpl .setView

```Java
//android.view.ViewRootImpl .java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {

            ...
           // Schedule the first layout -before- adding to the window
           // manager, to make sure we do the relayout before receiving
           // any other events from the system.
           //Step12
            requestLayout();
            ...
    }

```

- Step12 -- requestLayout

```Java
//android.view.ViewRootImpl .java
public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            //检查当前线程是否是UI线程
            checkThread();
            mLayoutRequested = true;
            //Step 13
            scheduleTraversals();
        }
    }
```

- Step13 -- ViewRootImpl.scheduleTraversals

```Java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            ...

            //这里通过Runable调用ViewRootImpl.doTraversal Step14
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
           ...    
}
```

- Step14 -- ViewRootImpl.doTraversal

```Java
void doTraversal() {
        if (mTraversalScheduled) {
            //Step15
            performTraversals();
    }
```

- Step15 -- ViewRootImpl.performTraversals

```Java
private void performTraversals() {
        ...
         // Ask host how big it wants to be
         //开始measure过程
         windowSizeMayChange |= measureHierarchy(host, lp, res,desiredWindowWidth, desiredWindowHeight);

        ...   

        if (didLayout) {
            //开始Layout过程
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);
         }
          ...
        //开始Draw过程
        //Step16
         performDraw();
         ...
    }
```

- Step16 -- ViewRootImpl.drawSoftware

performDraw会调用ViewRootImpl.draw然后会调用ViewRootImpl.drawSoftware。

```Java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        // Draw with software renderer.
        ...

        final Canvas canvas;
        try {
                canvas.translate(-xoff, -yoff);
                if (mTranslator != null) {
                    mTranslator.translateCanvas(canvas);
                }
                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                attachInfo.mSetIgnoreDirtyState = false;
               //进入Draw过程
                mView.draw(canvas);

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                if (!attachInfo.mSetIgnoreDirtyState) {
                    // Only clear the flag if it was not set during the mView.draw() call
                    attachInfo.mIgnoreDirtyState = false;
                }
            }
         ....
        return true;
    }
```
