---
title: Activity启动过程
date: 2016-05-28
categories:
  - Android
tags:
  - 源码
---

我们进行Android开发的时候，第一个例子就接触过`startActivity`这个函数，已经使用了很长时间了。今天我们就来看看Activity的启动到底是怎么回事。

本文略长...

<!--more-->

- Step1 -- ContextWrapper.startActivity()

```Java
//andrid.content.ContextWrapper.java
public void startActivity(Intent intent, Bundle options) {
       //Step2
        mBase.startActivity(intent, options);//mBase为Activity.attach里设置的ContextImpl
    }
```

- Step2 -- ContextImpl.startActivity

```Java
//android.app.ContextImpl.java
public void startActivity(Intent intent, Bundle options) {
    ...
    //mMainThread为ActivityThread的实例，Step3
    mMainThread.getInstrumentation().execStartActivity(
    getOuterContext(), mMainThread.getApplicationThread(), null,
        (Activity) null, intent, -1, options);
    }

```

- Step3 -- Instrumentation.execStartActivity

```Java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();

            //Step4
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

- Step4 -- ActivityManagerService.startActivity

```Java
//com.android.server.am.ActivityManagerService.java;

public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {

        //Step5
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }

```

- Step5 -- ActivityManagerService.startActivityAsUser

```Java
//com.android.server.am.ActivityManagerService.java;
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);

        //Step6
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```

- Step6 -- ActivityStackSupervisor.startActivityMayWait

```Java
//com.android.server.am.ActivityStackSupervisor.java

final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
        ...

        boolean componentSpecified = intent.getComponent() != null;

        // Don't modify the client's object!
        intent = new Intent(intent);

        // Collect information about the target of the Intent.
        //获取目标Activity的信息
        ActivityInfo aInfo =
                resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);

        ActivityContainer container = (ActivityContainer)iContainer;
        synchronized (mService) {
            ...
            //Step7
            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);

            ...
            return res;
        }
    }

```

- Step7 -- ActivityStackSupervisor.startActivityLocked

```Java
final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
        ...

        //Step8
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);

        ...
        return err;
    }
```

- Step8 -- ActivityStackSupervisor.startActivityUncheckedLocked

```Java
final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        ...


        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        //Step9
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // Don't set focus on an activity that's going to the back.
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
    }

```

- Step9 -- ActivityStackSupervisor.startActivityLocked

```Java
final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        TaskRecord rTask = r.task;
        ...
        //Step10
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

- Step10 -- ActivityStackSupervisor.resumeTopActivityInnerLocked

resumeTopActivitiesLocked会调用resumeTopActivityInnerLocked

```Java

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        ...
        // We need to start pausing the current activity so the top one
        // can be resumed...
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);

            //Step11
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }

        ...

        // We are starting up the next activity, so tell the window manager
        // that the previous one will be hidden soon.  This way it can know
        // to ignore it when computing the desired screen orientation.
```

- Step11 -- ActivityStackSupervisor.startPausingLocked

```
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ...
        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);

                //Step12
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
            ...
    }
```

- Step12 -- ActivityThread.handlePauseActivity

通过Binder进程间通信，最终调用ActivityThread.handlePauseActivity

```Java
private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
            ...
            //将当前Activity暂停
            performPauseActivity(token, finished, r.isPreHoneycomb());
            ...

            // Tell the activity manager we have paused.
            //前一个Activity已经暂停，现在通知ActivityManager，可以启动新的Activity
            //Step13
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
    }
```

- Step13 -- ActivityManagerService.activityPaused

`ActivityManagerNative.getDefault().activityPaused(token)``通过Binder调用最终调用`ActivityManagerService.activityPaused`

```Java
public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                //Step14
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```

- Step14 -- ActivityStack.activityPausedLocked

```Java
final void activityPausedLocked(IBinder token, boolean timeout) {
        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                        + (timeout ? " (due to timeout)" : " (pause complete)"));
                //Step15
                completePauseLocked(true);
            }
            ...
        }
    }

```

- Step15 -- ActivityStack.completePauseLocked

```Java
private void completePauseLocked(boolean resumeNext) {
        ActivityRecord prev = mPausingActivity;

        ...
        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                mStackSupervisor.checkReadyForSleepLocked();
                ActivityRecord top = topStack.topRunningActivityLocked(null);
                if (top == null || (prev != null && top != prev)) {
                    // If there are no more activities available to run,
                    // do resume anyway to start something.  Also if the top
                    // activity on the stack is not the just paused activity,
                    // we need to go ahead and resume it to ensure we complete
                    // an in-flight app switch.
                    //Step16
                    mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
                }
            }
        }
        ...
    }
```

- Step16 -- ActivityStackSupervisor.resumeTopActivitiesLocked

```Java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            //Step17
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }

```

- Step17 -- ActivityStackSupervisor.resumeTopActivityLocked

```Java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            //Step18
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```

- Step18 -- ActivityStack.resumeTopActivityInnerLocked

```Java

  ...
  //Step19
  mStackSupervisor.startSpecificActivityLocked(next, true, false);
  ...

```

- Step19 -- ActivityStackSupervisor.startSpecificActivityLocked

```Java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);

        //如果App已存在
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                //Step19.1
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        //如果App未启动
        //Step19.2
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

- Step19.1 -- realStartActivityLocked

```Java
final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig){

      ...
      app.forceProcessStateUpTo(mService.mTopProcessState);
      //启动Activity
      app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
      ...

}

```

- Step19.2 -- ActivityManagerService.startProcessLocked

```Java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
        boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {

        if (entryPoint == null)
            entryPoint = "android.app.ActivityThread";
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +app.processName);
        checkTime(startTime, "startProcess: asking zygote to start proc");

        //进入ActivityThread.main函数
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                      app.processName, uid, uid, gids, debugFlags, mountExternal,
                      app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                      app.info.dataDir, entryPointArgs);
        checkTime(startTime, "startProcess: returned from zygote!");
}
```
