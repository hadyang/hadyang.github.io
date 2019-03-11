---
title: 启动Launcher
date: 2016-05-27
categories:
  - Android

tags:
    - 源码
---

在上一篇文章《Android系统开机过程分析》中，我们分析了从init进程到系统服务启动的整个过程，虽然写的很简单（很多东西了解的不是很全面，后面会慢慢修改），但是整个启动过程还是很复杂，涉及的东西很多。今天我们来分析一下Android系统中用户第一个看见的App--Launcher的启动过程。

<!--more-->

从《Android系统开机过程分析》这篇文章中的Step9开始，接着分析。

- Step0 -- SystemServer.run

```Java
private void run() {
    try {
        //Step9.1
        startBootstrapServices();
        //Step9.2
        startCoreServices();
        //启动其他服务，包括网络、彩信等等，还启动了Launcher
        //Step1
        startOtherServices();
    } catch (Throwable ex) {
        ...
        throw ex;
    } finally {
        ...
    }
}
```

- Step1 -- SystemServer.startOtherServices

```Java
private void startOtherServices() {
        ...
        //Step2
        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                ...
                try {
		    //启动SystemUIService
                    startSystemUi(context);
                } catch (Throwable e) {
                    reportWtf("starting System UI", e);
                }
                ...
        });
    }
```

- Step2 -- ActivityManagerService.systemReady

```Java
//com.android.server.am.ActivityManagerService.java
public void systemReady(final Runnable goingCallback) {
        synchronized(this) {
            if (mSystemReady) {
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }

            ...
           //Step3
           startHomeActivityLocked(mCurrentUserId, "systemReady");
}
```

- Step3 -- ActivityManagerService.startHomeActivityLocked

```Java
boolean startHomeActivityLocked(int userId, String reason) {
        ...
        //Step3.1
        Intent intent = getHomeIntent();
        //通过上一步获取的intent来解析有哪些Launcher应用
        ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            ...
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                //Step4
                mStackSupervisor.startHomeActivity(intent, aInfo, reason);
            }
        }
        return true;
    }
```

- Step3.1 -- ActivityManagerService.getHomeIntent

```Java
Intent getHomeIntent() {
        //mTopAction==action.main
        //mTopData==null
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
             //在Launcher的主Activity有`android.intent.category.HOME`属性
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
}
```

- Step4 -- ActivityStackSupervisor.startHomeActivity

```Java
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason) {
        //将Home所在的Activity栈移动到前台
        moveHomeStackTaskToTop(HOME_ACTIVITY_TYPE, reason);

        //下面就进入到Activity的启动过程
        startActivityLocked(null /* caller */, intent, null /* resolvedType */, aInfo,
                null /* voiceSession */, null /* voiceInteractor */, null /* resultTo */,
                null /* resultWho */, 0 /* requestCode */, 0 /* callingPid */, 0 /* callingUid */,
                null /* callingPackage */, 0 /* realCallingPid */, 0 /* realCallingUid */,
                0 /* startFlags */, null /* options */, false /* ignoreTargetSecurity */,
                false /* componentSpecified */,
                null /* outActivity */, null /* container */,  null /* inTask */);

          ...   
}
```
