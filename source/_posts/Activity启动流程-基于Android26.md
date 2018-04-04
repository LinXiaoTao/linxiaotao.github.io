---
title: Activity启动流程(基于Android26)
date: 2018-03-27 09:54:17
categories: Android Framework
tags:
---

> 基于 Android 26，分析 Android Activity 启动流程

## 参考

[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)

## 源码

**源码篇幅可能过长，所以会省略一下不必要的代码和注释**

### Activity

``` java
@Override                                    
public void startActivity() {   
    this.startActivity(intent, null);        
}

@Override                                                           
public void startActivity() {
    if (options != null) {                                          
        startActivityForResult(intent, -1, options);                
    } else {                                                        
        startActivityForResult(intent, -1);                         
    }                                                               
}                                                                   
```

最终都会调用到 `startActivityForResult`：

``` java
public void startActivityForResult() {                                                             
    if (mParent == null) {
        // 转场动画的处理
        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity();                                                                                                                                                      
        if (ar != null) {                                                                       
            mMainThread.sendActivityResult();                                                                                                             
        }                                                                                       
        if (requestCode >= 0) {                                                                                        
            mStartedActivity = true;                                                            
        }                                                                                                                                                                             
    } else {                                                                                    
        // 如果存在 Parent Activity 则交由它处理                                                                                    
    }                                                                                           
}                                                                                               
```

可以知道 Activity 启动委托给了 Instrumentation 进行实现

### Instrumentation

``` java
public ActivityResult execStartActivity() {
    // 由 ActivityThread.getApplicationThread 提供的 ApplicationThread
    IApplicationThread whoThread = (IApplicationThread) contextThread;                                                                
    if (mActivityMonitors != null) {                                       
        synchronized (mSync) {                                             
            final int N = mActivityMonitors.size();                        
            for (int i=0; i<N; i++) {                                      
                final ActivityMonitor am = mActivityMonitors.get(i);       
                ActivityResult result = null;                              
                if (am.ignoreMatchingSpecificIntents()) {
                    // true 表示这个监视器被用于使用 onStartActivity 拦截所有 Activity 启动 
                    result = am.onStartActivity(intent);                   
                }                                                          
                if (result != null) {                                      
                    am.mHits++;                                            
                    return result;                                         
                } else if (am.match(who, null, intent)) {
                    // match 使用 IntentFilter 和 类名 进行匹配
                    am.mHits++;                                            
                    if (am.isBlocking()) {
                        // 当前监视器阻止 Activity 启动
                        return requestCode >= 0 ? am.getResult() : null;   
                    }                                                      
                    break;                                                 
                }                                                          
            }                                                              
        }                                                                  
    }                                                                      
    try {
        // 委托为 ActivityManagerService(AMS) 去处理
        int result = ActivityManager.getService().startActivity();                          
        // 检查 AMS 的处理结果
        checkStartActivityResult(result, intent);                          
    } catch (RemoteException e) {                                          
        throw new RuntimeException("Failure from system", e);              
    }                                                                      
    return null;                                                           
}                                                                          
```

可以通过最后的调用委托给了`ActivityManager.getService`，实际上就是 ActivityManagerService(AMS) 的 Binder 代理类实现

> 相对于之前的版本，比如 Android 23，现在已经没有了 ActivityManagerProxy 和 ActivityManagerNative，将直接与 system_server 进程上的 AMS 进行 IPC

``` java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();  
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =          
        new Singleton<IActivityManager>() {                                           
            @Override                                                                 
            protected IActivityManager create() {                                     
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);     
                return am;                                                            
            }                                                                         
        };                                                                            
```

### ActivityManagerService

``` java
@Override                                                                                       
public final int startActivity() {                           
    return startActivityAsUser();                                                     
}

@Override                                                                                                                          
public final int startActivityAsUser() {                                                                                                                                   
    return mActivityStarter.startActivityMayWait();                                                                                                
}                                                                                                                                  
```

这里将启动请求又委托给了 ActivityStarter

### ActivityStarter

``` java
final int startActivityMayWait() {                                                
    // Intent 的响应
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);                                                                                                                                           
    // Intent 的 Activity 信息                                                               
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);                                                                                                                                                                                                                                                                                                                                        
        final ActivityRecord[] outRecord = new ActivityRecord[1];
    	// 调用 startActivityLocked
        int res = startActivityLocked(outRecord);                                                                                                                          
        return res;                                                                                                       
    }                                                                                                                     
}                                                                                                                         
```

``` java
private int startActivity() {
    // 源 Activity 记录，即在哪个 Activity 进行 startActivity
    ActivityRecord sourceRecord = null;
    // 如果使用 startActivityForResult，result 返回的 Activity 称为结果 Activity
    ActivityRecord resultRecord = null;                                                                       
    if (resultTo != null) {
        // 获取 Activity Stack 中已经存在的源 Activity 记录
        sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);                                                                                 
        if (sourceRecord != null) {                                                                           
            if (requestCode >= 0 && !sourceRecord.finishing) {
                // requestCode >= 0，源 Activity 同时为 结果 Activity
                resultRecord = sourceRecord;                                                                  
            }                                                                                                 
        }                                                                                                     
    }                                                                                                         
                                                                                                              
    final int launchFlags = intent.getFlags();                                                                
                                                                                                              
    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {                   
        // 使用 FLAG_ACTIVITY_FORWARD_RESULT，可以将返回结果的源 Activity 转移为当前正在新启动的 Activity
        // 比如：A -> B,B - C 使用了 FLAG_ACTIVITY_FORWARD_RESULT,那么 C 的 setResult 会返回给 A
        if (requestCode >= 0) {
            // 不允许有 requestCode
            ActivityOptions.abort(options);                                                                   
            return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;                                        
        }
        // 将源 Activity 的结果 Activity 设置为新 Activity 的结果 Activity
        resultRecord = sourceRecord.resultTo;                                                                 
        if (resultRecord != null && !resultRecord.isInStackLocked()) {                                        
            resultRecord = null;                                                                              
        }
        // requestCode 处理
        resultWho = sourceRecord.resultWho;                                                                   
        requestCode = sourceRecord.requestCode;                                                               
        sourceRecord.resultTo = null;                                                                         
        if (resultRecord != null) {
            // 删除源 Activity 记录
            resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);                           
        }                                                                                                                                                                                         
    }                                                                                                         
                                                                                                              
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {                              
        // component 找不到                                                                         
        err = ActivityManager.START_INTENT_NOT_RESOLVED;                                                      
    }                                                                                                         
                                                                                                              
    if (err == ActivityManager.START_SUCCESS && aInfo == null) {                                              
        // ActivityInfo 找不到                                                                         
        err = ActivityManager.START_CLASS_NOT_FOUND;                                                          
    }                                                                                                         
                                                                                                              
    if (err == ActivityManager.START_SUCCESS && sourceRecord != null                                          
            && sourceRecord.getTask().voiceSession != null) {                                                 
        // 语音启动 Activity，检查是否符合                                                                      
    }                                                                                                         
                                                                                                              
    if (err == ActivityManager.START_SUCCESS && voiceSession != null) {                                       
        // 启动语音会话                                                       
    }                                                                                                         
                                                                                                              
    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.getStack();                  
                                                                                                              
    if (err != START_SUCCESS) {
        // 启动 Activity 失败
        if (resultRecord != null) {
            // 发送取消通知
            resultStack.sendActivityResultLocked(                                                             
                    -1, resultRecord, resultWho, requestCode, RESULT_CANCELED, null);                         
        }                                                                                                     
        ActivityOptions.abort(options);                                                                       
        return err;                                                                                           
    }                                                                                                         
                                                                                                              
    // 进行一些权限检查，判断是否终止                                                              
    if (abort) {
        // 如果需要终止 Activity
        if (resultRecord != null) {                                                                           
            resultStack.sendActivityResultLocked();                                                                   
        }                                                                                                     
        // 返回启动成功，实际终止                                                                
        ActivityOptions.abort(options);                                                                       
        return START_SUCCESS;                                                                                 
    }                                                                                                         
                                                                                                              
    // 如果权限检查是否在启动 Activity 之前，那么先启动权限检查的 Intent                                                             
                                                                                                              
    // 处理 ephemeral app                                                                    
    
    // 构造一个 ActivityRecord
    ActivityRecord r = new ActivityRecord();                                                   
                                                                                                     
    final ActivityStack stack = mSupervisor.mFocusedStack;                                                    
    if (voiceSession == null && (stack.mResumedActivity == null                                               
            || stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
        // 前台 stack 还没 resume 状态的 Activity，检查是否允许 app 切换
        if (!mService.checkAppSwitchAllowedLocked() {                                          
            PendingActivityLaunch pal =  new PendingActivityLaunch();                                              
            mPendingActivityLaunches.add(pal);                                                                
            ActivityOptions.abort(options);
            // 切换 app 失败
            return ActivityManager.START_SWITCHES_CANCELED;                                                   
        }                                                                                                     
    }                                                                                                         
                                                                                                              
    if (mService.mDidAppSwitch) {                                                                             
        // 从上次禁止 app 切换以来，这是第二次，允许 app 切换，并将切换时间设置为 0                                                            
        mService.mAppSwitchesAllowedTime = 0;                                                                 
    } else {                                                                                                  
        mService.mDidAppSwitch = true;                                                                        
    }                                                                                                         
    
    // 执行因为不允许 app 切换，而加到等待启动的 Activity
    doPendingActivityLaunchesLocked(false);                                                                   
                                                                                                              
    return startActivity();                                                                    
}                                                                                                             
```

``` java
private int startActivity() {                                                        
    int result = START_CANCELED;                                                               
    try {                                                                                      
        mService.mWindowManager.deferSurfaceLayout();
        // 下一步流程
        result = startActivityUnchecked();                           
    } finally {                                                                                
        // 如果启动 Activity 没有成功， 从 task 中移除 Activity                                                      
        if (!ActivityManager.isStartResultSuccessful(result)                                   
                && mStartActivity.getTask() != null) {                                         
            mStartActivity.getTask().removeActivity(mStartActivity);                           
        }                                                                                      
        mService.mWindowManager.continueSurfaceLayout();                                       
    }                                                                                          
                                                                                               
    postStartActivityProcessing();                                                                     
                                                                                               
    return result;                                                                             
}                                                                                              
```

``` java
private int startActivityUnchecked() {
	
    // 设置一些初始化状态
    setInitialState();
    // 计算 launch flags
    computeLaunchingTaskFlags();
    // 计算源 Task，源 Task 是否存在等
    computeSourceStack();
    // 获取是否存在可以复用的 Activity，根据 flags 和 launchMode
    ActivityRecord reusedActivity = getReusableIntentActivity();
    if (reusedActivity != null) {
    	// 存在可复用的 Activity，复用它
        // 可能需要清除 Task 中其他 Activity，并将启动的 Activity 前置
    }
    if (mStartActivity.packageName == null) {
    	return START_CLASS_NOT_FOUND;
    }
    // 如果启动的 Activity 与当前 Task 顶部的 Activity 相同，判断是否需要继续启动新的 Activity
    final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.realActivity.equals(mStartActivity.realActivity)
                && top.userId == mStartActivity.userId
                && top.app != null && top.app.thread != null
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || mLaunchSingleTop || mLaunchSingleTask);
    if(dontStart){
        // 传递一个新的 Intent 到 onNewIntent 
        top.deliverNewIntentLocked();
        return START_DELIVERED_TO_TOP;
    }
    
    // 获取 mTargetStack
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
    	newTask = true;
        // 需要创建新的 Task
        result = setTaskFromReuseOrCreateNewTask();
    } else if (mSourceRecord != null) {
        // 从源 Activity 中获取 Task
    	result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        // 从 InTask 中获取 Task
    	result = setTaskFromInTask();
    } else {
        // 可能创建新的 Task 或使用当前 Task，一般不会发生
    	setTaskToCurrentTopOrCreateNewTask();
    }
    
    // 使用 mTargetStack 启动 Activity 
    mTargetStack.startActivityLocked();
    
    if (mDoResume) {
        if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
        	// 目标 Task 不可聚焦 ，或者源 Task 栈顶 Activity 总是在其他 Activity 之上，并且不为源 Activity
            // 那么我们不恢复目标 Task，只需要确保它可见即可
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
    } else {
        if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
        	// 如果目标 Task 之前不是可聚焦，但是现在为可聚焦，那么移动到前台
            mTargetStack.moveToFront("startActivityUnchecked");
        }
        // 恢复目标 Task
        mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
    } else {
        // 如果不需要恢复，那么加到"最近活动"中
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    
    return START_SUCCESS;
}

private void setInitialState(){
    // 获取 DisplayId
    mSourceDisplayId = getSourceDisplayId();
    // 获取用于启动 Activity 的范围，Rect
    mLaunchBounds = getOerrideBounds();
    // launchMode
    mLaunchSingleTop = r.launchmode == LAUNCH_SINGLE_TOP;
    mLaunchSingleInstance = r.launchMode == LAUNCH_SINGLE_INSTANCE;
    mLaunchSingleTask = r.launchMode == LAUNCH_SINGLE_TASK;
    // Intent flags 的处理，如果和 Manifest 存在冲突，以 Manifest 为主
    // 如果 requestCode >= 0，同时启动的 Activity 位于新的 Task，发送取消的结果给源 Activity
    sendNewTaskResultRequestIfNeeded();
}

private void computeLaunchingTaskFlags(){
    if (mSourceRecord == null && mInTask != null && mInTask.getStack() != null){
        // 如果不存在源 Activity
        final Intent baseIntent = mInTask.getBaseIntent();
        final ActivityRecord root = mInTask.getRootActivity();
        if (baseIntent == null){
            throw new IllegalArgumentException();
        }
        if (mLaunchSingleInstance || mLaunchSingleTask) {
            // 如果设置了 SingleInstacne 或 SingleTask
        	if (!baseIntent.getComponent().equals(mStartActivity.intent.getComponent())){
                // Task 不符合
                throw new IllegalArgumentException();
            }
            
            if(root != null){
                // 已经存在 Task 根 Activity
                throw new IllegalArgumentException();
            }
        }
        
        if (root == null) {
        	// 如果不存在根 Activity，重新设置 launch flags
            mAddingToTask = true;
        }else if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
             mAddingToTask = false;
        }else {
            mAddingToTask = true;
        }
    }else {
                     if ((mStartActivity.isResolverActivity() || mStartActivity.noDisplay) && mSourceRecord != null
                    && mSourceRecord.isFreeform())  {
                         // 如果使用 ResolverActivity 启动或者 noDisplay
                mAddingToTask = true;
            }
    }
    
    if(mInTask == null){
        if (mSourceRecord == null) {
        	if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                // 不存在 Task，并且不存在源 Activity
            	mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        }else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
        	// 如果源 Activity 的 launchMode 是 SingleInstance，要设置 NEW_TASK flag
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        }else if (mLaunchSingleInstance || mLaunchSingleTask) {
        	// 如果启动 Activity 的 launchMode 是 SingleInstance 或 SingleTask，需要设置 NEW_TASK flag
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        }
    }
    
}
```

ActivityStarter 主要的作用是，计算 launch flags，创建或者复用合适的 Task，即 ActivityStack，从而启动 Activity

### ActivityStack

``` java
final void startActivityLocked(){
    if(!newTask) {
      // 如果从已存在的 Task 中启动 Activity
      boolean startIt = true;
      for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
      	task = mTaskHistory.get(taskNdx);
          if (task.getTopActivity() == null){
              // 如果 task 不存在 Activity
              continue;
          }
          if (task == rTask) {
          	// 找到对应的 task
            if (!startIt) {
            	// 如果当前对于用户还不可见，那么只是添加它，而不启动它，它将在用户导航回来时启动
                r.createWindowContainer();
                ActivityOptions.abort(options);
                return;
            }
          } else if (task.numFullscreen > 0) {
              startIt = false;
          }
      }
    }
    
    // 如果当前 task 为活动 task，那么不需要传递 onuserLeaving 回调
    if (task == activityTask && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
        mStackSupervisor.mUserLeaving = false;
    }
    
    if (!isHomeOrRecentsStack() || numActivities() > 0) {
    	// 如果当前不是 Home 或 Recent Task,或者活动 Activity 数量大于 0
        
        // 处理 动画
        
        if (newTask) {
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
            // 如果设置了重置的标记
            resetTaskIfNeededLocked(r, r);
            doShow = topRunningNonDelayedActivityLocked(null) == r;
            } else if (options != null && options.getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                // 需要进行转场动画
                doShow = false;
        }
    }
        
        
        if (r.mLaunchTaskBehind) {
        	// 如果为 true，那么不开启 window，但要确保 Activity 是可见的
            r.setVisibility(true);
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            TaskRecord prevTask = r.getTask();
            ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
            if (prev != null) {
                // 以下两种情况不展示之前的 Activity 预览
            	if (prev.getTask() != prevTask) {
                    // 之前的 Activity 在不同的 Task
                	prev = null;
                } else if (prev.nowVisible) {
                    // 现在可见
                	 prev = null;
                }
            }
            // 显示启动 Activity 的 Window
        	r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
        } else {
            // 当前为栈顶 Activity
            ActivityOptions.abort(options);
        }
    
}
```

`ActivityStack.startActivityLocked` 主要是创建 WindowContainer，同时显示 Window

### ActivityStackSupervisor

``` java
boolean resumeFocusedStackTopActivityLocked(){
    if (targetStack != null && isFocusedStack(targetStack)) {
        // 存在 targetStack
    	return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || r.state != RESUMED) {
        // 恢复聚焦 task
    	mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    } else if (r.state == RESUMED) {
        // 执行应用转场动画
    	mFocusedStack.executeAppTransition(targetOptions);
    }
}
```

### ActivityStack

``` java
boolean resumeTopActivityUncheckedLocked() {
    if (mStackSupervisor.inResumeTopActivity) {
        // 防止递归
    	return false;
    }
    try {
        // 设置恢复标记
        mStackSupervisor.inResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    // 在恢复过程中，确保必要的暂停逻辑
    mStackSupervisor.checkReadyForSleepLocked();
}
```

``` java
private boolean resumeTopActivityInnerLocked() {
    
    // 寻找需要恢复的栈顶 Activity，它必须是未结束并且聚焦
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
    final boolean hasRunningActivity = next != null;
    final ActivityRecord parent = mActivityContainer.mParentActivity;
    final boolean isParentNotResumed = parent != null && parent.state != ActivityState.RESUMED;
    if (hasRunningActivity
                && (isParentNotResumed || !mActivityContainer.isAttachedLocked())) {
           // 如果父 Activity 不是恢复状态，则不恢复当前 Activity
            return false;
        }
    
    if (!hasRunningActivity) {
        // 当前 Task 没有需要恢复的 Activity
    	return resumeTopActivityInNextFocusableStack(prev, options, "noMoreActivities");
    }
    
    if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
        // 如果 Activity 已经是恢复状态
        // 确保已经执行了所有等待的转场
        executeAppTransition(options);
    	return false;
    }
    
    if (mService.isSleepingOrShuttingDownLocked()
                && mLastPausedActivity == next
                && mStackSupervisor.allPausedActivitiesComplete()) {
    	// 如果系统处于休眠状态，当前 Activity 处于暂停状态
        // 确保转场执行
        executeAppTransition(options);
        return false;
    }
    
    if (!mService.mUserController.hasStartedUserState(next.userId)) {
        // 如果拥有该 Activity 的用户没有启动
    	return false;
    }
    
    if (!mStackSupervisor.allPausedActivitiesComplete()) {
    	// 如果存在暂停 Activity 的操作未完成
        return false;
    }
    
    boolean lastResumedCanPip = false;
    final ActivityStack lastFocusedStack = mStackSupervisor.getLastStack();
    if (lastFocusedStack != null && lastFocusedStack != this) {
    	final ActivityRecord lastResumed = lastFocusedStack.mResumedActivity;
        // 最后一个恢复的 Activity 是否可以 画中画
        lastResumedCanPip = lastResumed != null && lastResumed.checkEnterPictureInPictureState(
                    "resumeTopActivity", true /* noThrow */, userLeaving /* beforeStopping */);
    }
    
    // 是否需要可以在上一个 Activity 暂停时进行恢复
    final boolean resumeWhilePausing = (next.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0
                && !lastResumedCanPip;
    // 是否暂停了回退的 task
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
    
    if (mResumedActivity != null) {
        // 暂停上一个恢复状态的 Activity
    	pausing |= startPausingLocked(userLeaving, false, next, false);
    }
    
    if (pausing && !resumeWhilePausing) {
    	// 之前的 Activity 已经暂停，但不能进行恢复当前 Activity
        if (next.app != null && next.app.thread != null) {
            // hosting application，一般不执行
            // 让当前 Activity 放在 Lru 的顶部，避免早早杀死
        	mService.updateLruProcessLocked(next.app, true, null);
        }
        return true;
    } else if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                mStackSupervisor.allResumedActivitiesComplete()) {
    	// 当前需要恢复的 Activity 已经是恢复状态
        // 确保执行转场
        executeAppTransition(options);
        return true;
    }
    
    if (mService.isSleepingLocked() && mLastNoHistoryActivity != null &&
                !mLastNoHistoryActivity.finishing) {
        // 结束因为系统休眠而还没结束的 Activity
    	requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                    null, "resume-no-history", false);
        mLastNoHistoryActivity = null;
    }
    
    if (prev != null && prev != next) {
    	if (!mStackSupervisor.mActivitiesWaitingForVisibleActivity.contains(prev)
                    && next != null && !next.nowVisible) {
        	mStackSupervisor.mActivitiesWaitingForVisibleActivity.add(prev);
        } else {
            // 如果当前需要恢复的 Activity 已经可见，所有隐藏上一个 Activity
            if (prev.finishing) {
            	prev.setVisibility(false);
            } else {
                
            }
        }
    }
    
    // Activity 转场处理
    
    ActivityStack lastStack = mStackSupervisor.getLastStack();
    if (next.app != null && next.app.thread != null) {
        // 上一个 Activity 是否为透明
    	final boolean lastActivityTranslucent = lastStack != null
                    && (!lastStack.mFullscreen
                    || (lastStack.mLastPausedActivity != null
                    && !lastStack.mLastPausedActivity.fullscreen));
        
        if (!next.visible || next.stopped || lastActivityTranslucent) {
            // 前一个 Activity 为透明，并且当前 Activity 还没显示
            // 设置为显示状态
            next.setVisibility(true);
        }
        
        // 让窗口管理器重新基于新的 Activity 顺序评估屏幕的方向
    boolean notUpdated = true;
    if (mStackSupervisor.isFocusedStack(this)) {
    	final Configuration config = mWindowManager.updateOrientationFromAppTokens(
                        mStackSupervisor.getDisplayOverrideConfiguration(mDisplayId),
                        next.mayFreezeScreenLocked(next.app) ? next.appToken : null, mDisplayId);
        
        if (config != null) {
    	next.frozenBeforeDestroy = true;
        }
        notUpdated = !mService.updateDisplayOverrideConfigurationLocked(config, next,
                        false /* deferResume */, mDisplayId);
    }
    
     if (notUpdated) {
         // 配置发生更新无法保持已经存在的 Activity 实例
         // 重新获取需要恢复的 Activity
         ActivityRecord nextNext = topRunningActivityLocked();
         // 确保 Activity 仍然保持在栈顶，同时安排另外一次执行
         if (nextNext != next) {
         	mStackSupervisor.scheduleResumeTopActivities();
         }
         if (!next.visible || next.stopped) {
         	next.setVisibility(true);
         }
         next.completeResumeLocked();
         return true;
     }
    
    try {
        // 传递所有等待的结果
        
        if (next.newIntents != null){
            next.app.thread.scheduleNewIntent(
                            next.newIntents, next.appToken, false /* andPause */);
        }
        
        next.notifyAppResumed(next.stopped);
        
        // 准备恢复 Activity
        next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                        mService.isNextTransitionForward(), resumeAnimOptions);
    } catch (Exception e){
        // 发生异常，重新启动 Activity
        mStackSupervisor.startSpecificActivityLocked(next, true, false);
        return true;
    }
        
        try {
            next.completeResumeLocked();
        } catch(Exception e){
            // 发生异常，结束 Activity
            requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
            return true;
        }
        
    } else {
        // 需要启动 Activity
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
    
    return true;

}
```

如果可以不需要重新启动 Activity，则直接恢复 Activity 即可，否则进行重新启动流程：

### ActivityStackSupervisor

``` java
void startSpecificActivityLocked() {
    // 获取应用进程信息
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);
    
    if (app != null && app.thread != null) {
    	// 如果进程已经启动
        try {
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        }catch (RemoteException e){
            
        }
    }
    
    // 启动应用进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
}
```

``` java
final boolean realStartActivityLocked() {
    // 冻结屏幕
    r.startFreezingScreenLocked(app, 0);
    if (checkConfig) {
    	// 根据新的 Activity 顺序重新评估屏幕的方向
    }
    
    int idx = app.activities.indexOf(r);
    if (idx < 0) {
        // 将 Activity 添加到应用进程中
        app.activities.add(r);
    }
    
    try {
        app.thread.scheduleLaunchActivity();
    }catch(RemoteException e){
        // 发生异常，结束 Activity
        stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                        "2nd-crash", false);
        return false;
    }
    
    return true;
}
```

启动流程最后还是回到了 ApplicationThread

### ApplicationThread

``` java
public final void scheduleLaunchActivity() {
    // 通过 Handler 发送 LAUNCH_ACTIVITY
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

我们单独把 LAUNCH_ACTIVITY 的处理拿出来：

``` java
final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
```

### ActivityThread

``` java
private void handleLaunchActivity() {
    // 执行启动 Activity
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        // resume activity
        handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
    } else {
        
    }
}
```

``` java
private performLaunchActivity() {
    // Activity 信息初始化
    
    // 创建 context
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // 构建 Activity
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
    }catch(Exception e){
        
    }
    
    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if(activity != null){
        	activity.attach();
            
            // 通过 Instrumentation 执行 Activity onCreate
            if (r.isPersistable()) {
                 mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            }else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            
            if (!r.activity.mFinished) {
                // Activity onStart
            	activity.performStart();
            }
            
            // 通过 Instrumentation 执行 Activity onRestoreInstanceState
            if (!r.activity.mFinished) {
            	if (r.isPersistable()) {
                	if (r.state != null || r.persistentState != null) {
                    	mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            
             // 通过 Instrumentation 执行 Activity onPostCreeate
            if (!r.activity.mFinished) {
            	if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                }else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
            }       
    }
        
    return activity;
    
}
```

``` java
final void handleResumeActivity() {
    r = performResumeActivity();
    if(r != null) {
        final Activity a = r.activity;
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            if (a.mVisibleFromClient) {
            	if (!a.mWindowAdded) {
                	a.mWindowAdded = true;
                    // window
                    wm.addView(decor, l);
                }
            }
        }
    }
}
```

``` java
public final ActivityClientRecord performResumeActivity() {
 	if (r != null && !r.activity.mFinished) {
    	try {
            // 处理等待的 Intent
            if (r.pendingIntents != null) {
            	deliverNewIntents(r, r.pendingIntents);
                r.pendingIntents = null;
            }
            // 处理等待的 result
            if (r.pendingResults != null) {
            	deliverResults(r, r.pendingResults);
                r.pendingResults = null;
            }
            // 执行 resume
            r.activity.performResume();
        }
    }
}
```

### Instrumentation

``` java
public Activity newActivity(){
    return (Activity)cl.loadClass(className).newInstance();
}
```

``` java
private void callActivityOnCreate(){
    prePerformCreate(activity);
    activity.performCreate(icicle);
    postPerformCreate(activity);
}
```

### Activity

``` java
final void performCreate() {
    restoreHasCurrentPermissionRequest(icicle);
    // 调用 onCreate
    onCreate(icicle);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```

``` java
final void performStart() {
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    mInstrumentation.callActivityOnStart(this);
    mActivityTransitionState.enterReady(this);
}
```

``` java
final void performResume() {
    // 执行 restart
    performRestart();
    mInstrumentation.callActivityOnResume(this);
}
```

``` java
final void performRestart() {
    if (mStopped) {
        mStopped = false;
        mInstrumentation.callActivityOnRestart(this);
        // 执行 start
        performStart();
    }
}
```

### Instrumentation

``` java
public void callActivityOnStart() {
    activity.onStart();
}
```

``` java
public void callActivityOnResume() {
     activity.mResumed = true;
    activity.onResume();
}
```

``` java
public void callActivityOnRestart() {
    activity.onRestart();
}
```

## 总结

简单总结下 Activity 的启动流程：

1. 从用户应用进程开始启动，如果从桌面启动，则为 launcher 进程，用户进程通过 Binder 机制与 system_server 进程进行通信
2. ActivityManagerService 用于管理所有的 Activity 活动
   * 当接受到启动 Activity 的调用时，使用 `resolveActivity` ，查询系统中符合要求的 Activity
   * 创建使用合适的  ActivityStack 和 launch flags 来启动 Activity
   * 如果存在可以直接恢复 Activity，则恢复，否则重新启动 Activity
   * 如果不存在应用进程，先创建应用进程
3. 最终启动流程又会通过 Binder 调用回应用进程，使用 ActivityThread 去执行
   * 使用 Instrumentation 去通过反射构建 Activity 实例
   * 使用 Handler 机制调用 Activity 的生命周期

下面的图例来自[博客](http://gityuan.com/2016/03/12/start-activity/)：

![启动流程](http://gityuan.com/images/activity/start_activity_process.jpg)

### 时序图

![Android启动流程](http://oy017242u.bkt.clouddn.com/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png)

