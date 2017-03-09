---
title: 记一次需求的开发--在ViewPager中嵌套WebView
categories: Android
tags: Android
---

###  前言
 ViewPager是可以左右滑动的组件，我们经常会用到，当ViewPager中嵌套了WebView且WebView中的内容也需要左右滑动的时候就会出现事件冲突：因为ViewPager需要消费左右滑动的事件，WebView也需要消费左右滑动的事件，那么该如何解决这样的问题？首先我们来回顾下Android事件传递的机制。
 
### Android事件传递机制
其实网上有很多讲解Android事件传递机制的文章，但感觉讲的都不是很清楚，因为网上大部分都是切割成三部分来讲，分别为：Activity、ViewGroup、View，但其实Android事件传递本质上是一个递归，如果单纯的切割开来，会忽略很多的内部实现细节，所以都不如自己看源代码来的实在。

我们从事件开始传递的最初入口开始讲起
### Activity
事件是从Activity开始传递，具体代码如下：
`Activity.java`
```
    public boolean dispatchTouchEvent(MotionEvent ev) {
    	//如果事件为ACTION_DOWN，则调用onUserInteraction(),我们可以看到这个函数的实现为空
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        //调用window的dispatchTouchEvent，如果返回true，即下层的消费了此事件，则直接return true，否则返回false，即下层没有消费此事件，就掉用Activity的onTouchEvent消费本次事件
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

在看Window里的superDispatchTouchEvent是怎么实现的
这里的getWindow返回的winow是PhoneWindow的实例
`PhoneWindow.java`
```
    @Override
    public boolean superDispatchKeyEvent(KeyEvent event) {
    	//这里调用的是mDecor的superDispatchKeyEvent
        return mDecor.superDispatchKeyEvent(event);
    }
```

mDecor是DecorView的实例
`DecorView.java`
```
public boolean superDispatchTouchEvent(MotionEvent event) {
       //这里是调用的父类的dispatchKeyShortcutEvent方法
        return super.dispatchTouchEvent(event);
    }
```
DecoreView的父类是FrameLayout,FrameLayout没有实现dispatchTouchEvent，因此调用的是FrameLayout的父类ViewGroup的dispatchTouchEvent的方法。
InfiniteData
第一部分讲到这就结束了，因为我们都知道Activity的最外层包裹的是DecorView，DecoreView里面在包裹我们自己定义实现的Activity的view，而DecoreView继承自FrameLayout，本质上也是一个view，所以总结来说就是Activity获得了事件，随即就抛给了view来处理。

## ViewGroup
这里就进入到递归的主要逻辑里面，只要把这里搞懂，那其实就是搞懂了事件的传递机制，同样从代码入手，从上面第一节我们知道Activity里面的事件最后调用的是ViewGroup的dispatchTouchEvent事件，所以我们从ViewGroup 的dispatchTouchEvent看起，这里代码很多，我们一行一行来分析
`dispatchTouchEvent`

```
public boolean dispatchTouchEvent(MotionEvent ev) {
	//此处mInputEventConsistencyVerifier是用于测试用的，很容易被迷惑，因为这也有onTouchEvent方法，会误以为在这里就消费了事件，其实没有，只是测试用的
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
        //获取当前事件的action
            final int action = ev.getAction();
            //获取当前事件是哪个action
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            //如果事件为down事件
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            //intercept这个是用来标记是否拦截此次事件的，true：拦截，自己消费，false：不拦截，接着往下传递
            //mFirstTouchTarget:是一个链表，用来保存这个viewgroup下面可以接受事件的view，这里如果为null，则代表这个viewgroup没有子view，所以不用分发事件，直接自己消费，这里是为了提高运行效率的。当然你第一次进来的时候，这个肯定为null，但是却满足了第一个条件，即事件为down事件，所以会走下去
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

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
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

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
