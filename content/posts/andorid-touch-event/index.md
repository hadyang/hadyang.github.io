---
title: 事件分发过程
date: 2016-06-04
categories: Android
type: post
tags:
    - 源码
---

这篇文章我们继续分析Activity的事件分发过程。

<!--more-->

- Step1 -- ViewRootImpl.dispatchInputEvent

```Java
public void dispatchInputEvent(InputEvent event, InputEventReceiver receiver) {
        SomeArgs args = SomeArgs.obtain();
        args.arg1 = event;
        args.arg2 = receiver;
        Message msg = mHandler.obtainMessage(MSG_DISPATCH_INPUT_EVENT, args);
        msg.setAsynchronous(true);

        //Step2
        mHandler.sendMessage(msg);
    }
```

- Step2 -- ViewRootImpl.doProcessInputEvents

```Java
void doProcessInputEvents() {
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            ...
            //Step3
            deliverInputEvent(q);
        }
}
```

- Step3 -- ViewRootImpl.deliverInputEvent

```Java
private void deliverInputEvent(QueuedInputEvent q) {
        ...

        if (stage != null) {
            //Step4
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
}

```

- Step4 -- ViewPostImeInputStage.onProcess

`stage.deliver(q)`最终调用到ViewPostImeInputStage.onProcess

```Java
protected int onProcess(QueuedInputEvent q) {
        final int source = q.mEvent.getSource();
        if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
        //Step5
        return processPointerEvent(q);
        } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
            return processTrackballEvent(q);
        } else {
            return processGenericMotionEvent(q);
        }
    }
```

- Step5 -- ViewRootImpl.processPointerEvent

```Java
private int processPointerEvent(QueuedInputEvent q) {
      ...
      //Step6 mView为DecorView
      boolean handled = mView.dispatchPointerEvent(event);
      if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
          mUnbufferedInputDispatch = true;
      if (mConsumeBatchedInputScheduled) {
          scheduleConsumeBatchedInputImmediately();
        }
    }
    return handled ? FINISH_HANDLED : FORWARD;
}

```

- Step6 -- View.dispatchPointerEvent

```Java
public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

- Step6.1 -- DecorView.dispatchTouchEvent

```Java
public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();
            //cb为Activity，Step7
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
}
```

- Step7 -- Activity.dispatchTouchEvent

```Java
public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        //Step8
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

- Step8 -- DecorView.superDispatchTouchEvent

PhoneWindow.superDispatchTouchEvent(ev)最终调用DecorView.superDispatchTouchEvent

```Java
public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
}
```

- Step9 -- ViewGroup.dispatchTouchEvent

```Java
public boolean dispatchTouchEvent(MotionEvent ev) {

        ...
        if (onFilterTouchEventForSecurity(ev)) {
            ...
            // Check for interception.
            //检查是否拦截该事件，只能拦截ACTION_DOWN
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0){
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);
                            ...

                            //事件分发 Step10
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();

                                //添加下一个子View
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }
                    ...
                }
            }

            // Dispatch to touch targets.
            //如果ViewGroup拦截了这个事件，mFirstTouchTarget则为NULL，即进入这个分支，调用自己的dispatchTouchEvent
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            }
        return handled;
}
```

- Step10 -- ViewGroup.dispatchTransformedTouchEvent

```Java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                //自己进行事件分发
                handled = super.dispatchTouchEvent(event);
            } else {
                //对子View进行事件分发
                //Step11
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
}
```

- Step11 -- View.dispatchTouchEvent

```Java
public boolean dispatchTouchEvent(MotionEvent event) {
        boolean result = false;

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    //onTouch方法比onTouchEvent方法优先级高，如果onTouch返回为true，即result==true，则不走onTouchEvent
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            //Step12
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        return result;
}
```

- Step12 -- View.onTouchEvent

```Java
public boolean onTouchEvent(MotionEvent event) {
        //View不可见，还是会消耗事件
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }

        //如果设置的有代理，则会执行代理
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {

                        ...
                            if (!focusTaken) {
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                //如果onClickListener不为空，执行onClick
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        ...
                    break;
                case MotionEvent.ACTION_DOWN:
                    if (isInScrollingContainer) {
                        ...
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        //如果onLongClickListener不为空，则执行onLongClick
                        checkForLongClick(0);
                    }
                    break;
                    ...
            }
            return true;
        }
        return false;
}
```
