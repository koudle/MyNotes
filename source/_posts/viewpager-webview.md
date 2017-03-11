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
这里就进入到递归的主要逻辑里面，只要把这里搞懂，就搞懂了事件的传递机制，同样从代码入手，从上面第一节我们知道Activity里面的事件最后调用的是ViewGroup的dispatchTouchEvent事件，所以我们从ViewGroup 的dispatchTouchEvent看起，这里代码很多，我们一行一行来分析
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
        //这里有一个方法 onFilterTouchEventForSecurity，是用来做安全校验的，通过校验true，则开始分发事件，否则将直接返回，看下面的具体代码
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
                    //这里有一个flag，是通过requestDisallowInterceptTouchEvent来设置的，如果不拦截，则分发事件，否则自己处理，这里有点绕，详细解释下
                    //首先，(mGroupFlags & FLAG_DISALLOW_INTERCEPT) ，因为是&操作，如果结果不等于0,则disallowIntercept为true，代表有FLAG_DISALLOW_INTERCEPT这个设置，这个viewgroup不执行onInterceptTouchEvent方法，对事件进行分发
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
            //这里intercepted如果为true，表示事件已经被viewgroup拦截，viewgroup会自己消费事件，如果为false，表示viewgroup暂时不消费此事件，需要对事件进行分发
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
            //这里如果canceld和intercept都为false，才会分发事件
            if (!canceled && !intercepted) {

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

		//这里actionMasked如果符合这些条件
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

			//mChildrenCount是全局变量，会记录viewgroup里子view的数量
                    final int childrenCount = mChildrenCount;
                    //从前面看到newTouchTarget为前面几行定义的局部变量，且初始值为null，所以这里只要childrenCount不为0,一定会走下面的逻辑，如果childrenCount为0,则viewgroup子view为0,也就不需要分发事件了
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        //buildOrderedChildList这个方法会把viewgroup的所有子view按照前后顺序排序，以便决定接收事件的顺序
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                                //mChildren为全局变量，储存viewgroup的子view
                        final View[] children = mChildren;
                        //遍历子view
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
                            //dispatchTransformedTouchEvent就是分发事件的函数，我们可以往下看它具体的代码，这里如果返回true，则代表有子view消费了此次事件，那么分发到此为止，如果为false，则继续分发
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
            //mFirstTouchTarget是viewgroup可接受事件的子view的缓存
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                 //mFirstTouchTarget为null，代表没有可接受事件的子view
                //所以dispatchTransformedTouchEvent中child的变量为null，意思是自己来消费事件
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
        //handled的值为dispatchTransformedTouchEvent的值，也就是onDispatchTouchEvent的值，也就是onTouchEvent的值，handled的初始值为false，
        return handled;
    }
```


`onFilterTouchEventForSecurity`
```
根据这个方法的注释，很容易理解这个方法的含义
/**
     * Filter the touch event to apply security policies.
     *
     * @param event The motion event to be filtered.
     * @return True if the event should be dispatched, false if the event should be dropped.
     *
     * @see #getFilterTouchesWhenObscured
     */
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }
```
`requestDisallowInterceptTouchEvent`
```
/**
     * {@inheritDoc}
     */
    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }

```

`dispatchTransformedTouchEvent`
```
/**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        //看注释，就是这是处理特殊case的代码
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        //这个才是重点
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                //这里child为null，其实代表的是viewgroup自己，而viewgroup的父类为view，所以这里的 super.dispatchTouchEvent(event)，意思就是调用viewgroup自己的dispatchTouchEvent，因为view没有onInterceptTouchEvent，进而调用自己的onTouchEvent，其实就是自己消费这次事件
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);
		//chile不为null的话，就是调用子view 的dispatchTouchEvent事件，就是分发事件给子view
                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }
	
	//这里和上面同理
        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        //handled的值为调用dispatchTouchEvent的值，后面会看到这个值其实是OnTouchEvent的值
        return handled;
    }
```

* 总结
我们会看到这里有三百多行代码，逻辑非常复杂，但是我们剔除一些不必要的干扰项，其实逻辑还是很清晰的

* 1、第一步，首先，对down事件进行处理，清除上次缓存的状态，在down事件里判断viewgroup是否要拦截事件，首先判断标志位FLAG_DISALLOW_INTERCEPT，``有这个标志位表示不允许拦截``，则直接进行第二步事件的分发；``没有标志未表示允许拦截``，则先调用当前viewgroup的onInterceptTouchEvent，根据onInterceptTouchEvent的值，``如果为true``，表示当前viewgroup要消费这个事件，则执行第三步，即执行该viewgroup的onTouchEvent事件;``如果为false``，则表示当前viewgroup不消费此事件，则执行第二步，对该事件进行分发。

* 2、第二步，要对事件进行分发，如果viewgroup有子view，则按照z-order顺序分发事件，首先判断事件的坐标在不在这个view里面，若是才执行子view的dispatchTouchEvent的方法，``若这个方法返回值为true``，就是有子view消费了此事件，则继续遍历（这里虽然用链表来存储接受事件的view，但是这个view每次应该只有一个，因为同一个事件只能一个view来消费，这里还需探讨？），返回true;``若这个方法返回值为false``，则一直遍历完所有的子view，最后返回false。

* 3、

### View
view这里的代码就少很多
`dispatchTouchEvent`

```
/**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

//这里和viewgroup的一样
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            //如果有设置onTouchListener，则执行，且返回结果标记为true，代表事件已消费
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

//没有执行上面的onTouchListener，才会执行onTouchEvent
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```

`onTouchEvent`

```
 /**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

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
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

	//这里要注意只要这个view设置了 CLICKABLE  LONG_CLICKABLE CONTEXT_CLICKABLE这三个属相，那么返回的结果必然为true，而我们一般用到的view绝大部分默认都是设置了的（因为大部分view默认都是可以接受事件的除了View）
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```