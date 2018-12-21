# #startActivityForResult

​	

​	HomeActivity在启动了SecondActivity之后要启动一个新的Activity ForResultActivity了，但是这次是使用startActivityForResult来启动，在提供Intent之外还提供一个整数requestCode：

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    ......
        Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
    ......

}
```

​	startActivity能够启动新的Activity其实也是调用的startActivityForResult，只是传入的requestCode是-1，在这种情况下则不需要被启动的Activity在finish的时候调用setResult。而通过startActivityForResult调用的时候，requestCode就不能是-1了，不然就变成startActivity了。mInstrumentation.execStartActivity会调用如下函数通知AMS去启动ForResultActivity。

```
int result = ActivityManagerNative.getDefault().startActivity(whoThread,
who.getBasePackageName(), intent,intent.resolveTypeIfNeeded(who.getContentResolver()),
token, target != null ? target.mEmbeddedID : null,requestCode, 0, null, options);
```

​	whoThread是进程的ApplicationThread，who.getBasePackageName()是Activity的packageName，intent.resolveTypeIfNeeded是Null，token就是Activity里的mToken，通过它去寻找Home Activity的ActivityRecord对象，target就是启动新的Activity的Activity，也就是我们的Home Activity，target.mEmbeddedID通常情况下都是Null，requestCode就是我们设置的非-1的数字，options则是Null。

​	ActivityManagerNative.getDefault().startActivity通过Binder调用进入到AMS中，流程与启动SecondActivity是一样的，不同的只是requestCode。

​	启动过程与SecondActivity一直相同，直到ActivityStackSupervisor.startActivityLocked：

```java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] 		   outActivity,ActivityContainer container, TaskRecord inTask) {
     ......
      if (resultTo != null) {
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
     ......
          final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;
     ......
          ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
 requestCode, componentSpecified, voiceSession != null, this, container, options);
	......
         err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
   ......
       return err;
    }
```

​	参数里的resultTo是HomeActiivty的mToken，resultRecord是要接收result的ActivityRecord，在startActivityLocked里有个判断条件，if (requestCode >= 0 && !sourceRecord.finishing)，resultRecord才不会为Null，这也就是说如果我们的RequestCode是负数，那就不会有什么result回传了。

​	之后会进行很多对Activity的判断，一些情况会导致回传给source activity的结果是RESULT_CANCELED，如果没有发生RESULT_CANCELED的情况，会继续执行到startActivityUncheckedLocked：

```java
final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int 	startFlags,boolean doResume, Bundle options, TaskRecord inTask) {
    ......
     if (r.resultTo != null && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0
                && r.resultTo.task.stack != null) {
            r.resultTo.task.stack.sendActivityResultLocked(-1,
                    r.resultTo, r.resultWho, r.requestCode,
                    Activity.RESULT_CANCELED, null);
            r.resultTo = null;
        }
    ......
    if (r.packageName != null) {
    ......
    }else {
            if (r.resultTo != null && r.resultTo.task.stack != null) {
                r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho,
                        r.requestCode, Activity.RESULT_CANCELED, null);
            }
            ActivityOptions.abort(options);
            return ActivityManager.START_CLASS_NOT_FOUND;
        }
    ......
    targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
	......
    return ActivityManager.START_SUCCESS;
}
```

​	如果ForResultActivity已经找到了resultRecord，但是在启动Intent里设置了FLAG_ACTIVITY_NEW_TASK，那么立刻会给resultRecord发一个RESULT_CANCELED的result，ForResultActivity的启动仍正常进行。这说明获取Activity的result不能跨task，不在同一个TaskRecord中的Activity不能传递setResult的结果。

​	如果在PackageManagerService中没有找到ForResultActivity的packageName(即ActivityInfo.applicationInfo.packageName),那么ForResultActivity的启动不能成功，立刻返回给SecondActivity一个 RESULT_CANCELED的result。

​	在source Activity的onActivityResult中可能得到的resultCode有三种RESULT_OK( -1 )、RESULT_CANCELED( 0 )、RESULT_FIRST_USER，RESULT_FIRST_USER则不是一个具体的resultCode，而是表示用户自定义的大于等于1的resultCode，在onActivityResult中，用户可以通过判断resultCode进行不同的逻辑处理。只有RESULT_CANCELED表示获取result的过程失败了，被返回结果为RESULT_CANCELED的result有如下几种情况：

​	1.未找到能匹配启动intent的Activity；

​	2.未找到intent中指定的Activity；

​	3.启动一个其他用户安装的应用中包含的Activity，且该Activity并没有声明android:showForAllUsers="true"；

​	4.source Activity是语音交互的，但是target Activity却没有Intent.CATEGORY_VOICE的声明；

​	5.AppOpsManager对启动intent里的component或者action进行了限制；

​	6.AMS的IntentFirewall不允许启动这个activity intent；

​	7.AMS里注册了IActivityController对象且Controller不允许启动该intent；

​	8.source Activity（启动其他Activity的Activity）与target Activity（被启动的Activity）不在同一个task中；

​	9.找不到target Activity的包名；

​	10.target Activity意外crash；

​	11.target Activity在finish之前没有调用setResult；

其中的1，2，3，4，8，9，10，11是我们可能经常遇到的使source Activity的onActivityResult里得到RESULT_CANCELED的情况。

​	假设我们的ForResultActivity的启动很正常，可以进行用户交互、逻辑处理等，在处理完以后，调用

```java
int resultCode = Activity.RESULT_OK;
Intent data = new Intent();
data.setxxx();
setResult( resultCode,  data);
finish();
```

​	setResult(int resultCode, Intent data)函数在给source Activity返回resultCode的同时也返回了一个Intent，提供一些需要source Activity处理的数据，这两个参数分别保存在Activity的mResultCode和mResultData中。finish是执行了Activity的finish（false）,进而调用AMS的finishActivity：

```java
public final boolean finishActivity(IBinder token, int resultCode, Intent resultData, boolean 	finishTask/*false*/){
        ······
    synchronized(this) {
        ActivityRecord r = ActivityRecord.isInStackLocked(token);
        TaskRecord tr = r.task;
        ActivityRecord rootR = tr.getRootActivity();
        ······
        if (finishTask && r == rootR) {
            // If requested, remove the task that is associated to this activity only if it
            // was the root activity in the task. The result code and data is ignored
            // because we don't support returning them across task boundaries.
            res = removeTaskByIdLocked(tr.taskId, false);
            if (!res) {
                Slog.i(TAG, "Removing task failed to finish activity");
            }
        } else {
            res = tr.stack.requestFinishActivityLocked(token, resultCode,
                resultData, "app-request", true);
            if (!res) {   
                Slog.i(TAG, "Failed to finish by app-request");
            }
        }
    ······
}
```

​	token是Activity的mToken变量（在“第一个Activity的流程中”有介绍），resultCode和resultData分别是finish之前Activity调用setResult传入的值。

​	finishTask参数会导致Activity所在的TaskRecord出现两种不同的结局：一种被remove，一种被保留。

​    1.finishTask为true：

​    当我们要finish的Activity是TaskRecord中的root，也就是TaskRecord中mActivities列表中未finish且index最小的Activity，那么会调用removeTaskByIdLocked。也就是说我们的Activity如果不是root也是不会remove这个TaskRecord的。remove 的过程包括：将TaskRecord中的Activity全部finish，从Recents中删除这个TaskRecord，将与TaskRecord相关的一些进程杀掉，但这些进程不包括HOME进程、有前台Service的进程、其他用户的进程等。resultCode和resultData也会被忽略，因为result结果不支持跨TaskRecord传递，而此时source Activiy此时要么被finish了，要么就在其他TaskRecord中，不论哪种都不会将result传给它。

​    2.1.finishTask为false：

​    这时由Activity所在的ActivityStack来处理。将Activity对应ActivityRecord的finishing置为true，TaskRecord的frontOfTask变成index最小且没finish的Activity，调整AMS的mFocusedActivity，若Activity是由startActivityForResult启动的，将result信息存入启动它的Activity对应的ActivityRecord中results列表中。如果要finish的Activity就是当前ActivityStack中的mResumedActivity且当前ActivityStack中的mPausingActivity为空，Activity先进入pause流程，此时ActivityStack的mPausingActivity就是我们要finish的Activity了。pause流程与前面一样，区别是在completePauseLocked函数中，这个函数里先判断mPausingActivity的finishing变量，因为前面已经将finishing置为true，这里就会直接调用finishCurrentActivityLocked将mPausingActivity进行finish操作。

```java
 final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
                                    String reason, boolean oomAdj) {
	......
    r.makeFinishingLocked();
    ......
    adjustFocusedActivityLocked(r, "finishActivity");
    finishActivityResultsLocked(r, resultCode, resultData);
    if (mResumedActivity == r) {
            boolean endTask = index <= 0;
        	......
            if (mPausingActivity == null) {
                startPausingLocked(false, false, false, false);
            }
        	......
        } else if (r.state != ActivityState.PAUSING) {
        	......
            return finishCurrentActivityLocked(r, FINISH_AFTER_PAUSE, oomAdj) == null;
        } 
     ......
 }
```

​	makeFinishingLocked将ActiviyRecord的finishing置为true，表示这个Activity处于finish的流程中。如果当前被finish的Activity所在的ActivityStack是mFocusedStack，且该Activity是AMS的mFocusedActivity，adjustFocusedActivityLocked就会调整AMS中mFocusedStack和mFocusedActivity的值：

​	1.若finishing的Actiivty是所在task的root，且该task是所在ActivityStack的top，同时对应TaskRecord对象中的mTaskToReturnTo值为HOME_ACTIVITY_TYPE或者RECENTS_ACTIVITY_TYPE（TaskRecord初始化的时候默认mTaskToReturnTo的值就是HOME_ACTIVITY_TYPE，但是也可以通过TaskRecord.setTaskToReturnTo将其设置为APPLICATION_ACTIVITY_TYPE或者RECENTS_ACTIVITY_TYPE），那么调整当前Activity被finish以后需要显示的Activity：

1）若task对应TaskRecord的mTaskToReturnTo为RECENTS_ACTIVITY_TYPE，显示recent列表

2）若task对应TaskRecord的mTaskToReturnTo为HOME_ACTIVITY_TYPE，将mHomeStack置顶，将AMS的mFocusedActivity置为HomeActivity。

​	2.如果finishing的Activity不满足1所要求的条件，找到ActivityStackSupervisor中最顶层不处于finishing状态的ActivityRecord，将它的ActivityStack置顶，并将它赋值为AMS的mFocusedActivity，这意味着下一个显示的Activity就是这个了。

​	finishActivityResultsLocked找到r.resultTo，并调用r.resultTo的addResultLocked:

```java
  void addResultLocked(ActivityRecord from, String resultWho,
            int requestCode, int resultCode,
            Intent resultData) {
        ActivityResult r = new ActivityResult(from, resultWho,
                requestCode, resultCode, resultData);
        if (results == null) {
            results = new ArrayList<ResultInfo>();
        }
        results.add(r);
    }
```

​	addResultLocked把要传递的result封装成ActivityResult对象存入到了r.resultTo的results列表中，这样在r.resultTo里就有了一个需要传给source Activity的ActivityResult对象。finishActivityResultsLocked还会将r.resultTo置Null，表示result的传递已经与它无关了。

​	因为ForResultActivity是系统中当前显示的Activity，也就是mResumedActivity == r，在finishActivityResultsLocked之后进入的就是startPausingLocked函数，这个函数的逻辑与前面启动第二个Activity时是一样的，在Activity的pause流程结束后，也要回调AMS的activityPaused，直到调用至ActivityStack的completePauseLocked函数。

​	由于在makeFinishingLocked中，已经将ForResultActivity对应ActivityRecord中的finishing置为true，那么completePauseLocked会直接进入finishCurrentActivityLocked运行：

```java
private void completePauseLocked(boolean resumeNext) {
        ActivityRecord prev = mPausingActivity;
    	......
        if (prev != null) {
            prev.state = ActivityState.PAUSED;
            if (prev.finishing) {
                ......
                prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false);
            }
            	......
                mPausingActivity = null;
        }
        ......
         if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                ......
            }
        }
    .......
}
```

​	finishCurrentActivityLocked在整个finish的过程中要执行两次：

```java
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj) {
	......
    if (mode == FINISH_AFTER_VISIBLE && r.nowVisible) {
        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
            mStackSupervisor.mStoppingActivities.add(r);
            if (mStackSupervisor.mStoppingActivities.size() > 3
                    || r.frontOfTask && mTaskHistory.size() <= 1) {
                mStackSupervisor.scheduleIdleLocked();
            } else {
                mStackSupervisor.checkReadyForSleepLocked();
            }
        }
        r.state = ActivityState.STOPPING;
        if (oomAdj) {
            mService.updateOomAdjLocked();
        }
        return r;
    }
/////////////////////////////////第二次的分割线////////////////////////////////////////////
    // make sure the record is cleaned out of other places.
    mStackSupervisor.mStoppingActivities.remove(r);
    mStackSupervisor.mGoingToSleepActivities.remove(r);
    mStackSupervisor.mWaitingVisibleActivities.remove(r);
    if (mResumedActivity == r) {
        mResumedActivity = null;
    }
    final ActivityState prevState = r.state;
    r.state = ActivityState.FINISHING;

    if (mode == FINISH_IMMEDIATELY
            || (mode == FINISH_AFTER_PAUSE && prevState == ActivityState.PAUSED)
            || prevState == ActivityState.STOPPED
            || prevState == ActivityState.INITIALIZING) {
        // If this activity is already stopped, we can just finish
        // it right now.
        r.makeFinishingLocked();
        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm");
        if (activityRemoved) {
            mStackSupervisor.resumeTopActivitiesLocked();
        }
        if (DEBUG_CONTAINERS) Slog.d(TAG_CONTAINERS,
                "destroyActivityLocked: finishCurrentActivityLocked r=" + r +
                " destroy returned removed=" + activityRemoved);
        return activityRemoved ? null : r;
    }

    // Need to go through the full pause cycle to get this
    // activity into the stopped state and then finish it.
    if (DEBUG_ALL) Slog.v(TAG, "Enqueueing pending finish: " + r);
    mStackSupervisor.mFinishingActivities.add(r);
    r.resumeKeyDispatchingLocked();
    mStackSupervisor.getFocusedStack().resumeTopActivityLocked(null);
    return r;
}
```

​	第一次：如果当前要finish的Activity仍然可见，且调用finishCurrentActivityLocked时传入的参数mode的值是FINISH_AFTER_VISIBLE，它的ActivityRecord会暂时放入ActivityStackSupervisor的mStoppingActivities列表中，等待下次scheduleIdle时处理。因为WMS还没有将ForResultActivity所在的Window隐藏，所以此时Activity是可见的,它的ActivityRecord被放入mStoppingActivities中，ActivityRecord的state被置为ActivityState.STOPPING。之后completePauseLocked函数会调用resumeTopActivityInnerLocked，此时传入的prev是ActivityStack中的mPausingActivity，也就是我们要finish的ForResultActivity的ActivityRecrod，resumeTopActivityInnerLocked的流程和启动第二个Activity时resumeTopActivityInnerLocked第二次调用的流程是一样的，所获得的next就是当前所有ActivityStack中的top，也就是SecondActivity。之后next就进入了resume流程，只是resume流程中增加了onRestart和onStart的过程，之所以前面的resume过程没有onRestart和onStart，是因为Activity中有一个变量mStopped，Activity的过程中会调用performRestart，在这个函数中，当mStopped为true的时候才会调用onRestart及performStart，performStart会进一步调用onStart。

 	第二次：当下一个Activity的resume结束，重新安排的Idler就会执行了，scheduleIdle会调用activityIdleInternalLocked，使ForResultActivity进入stop流程，因为finishing为true，此时ForResultActivity的window会先隐藏。在“启动第二个Activity”中说过，Activity的stop流程是由ActivityStackSupervisor.activityIdleInternalLocked函数完成的，这里所不同的是，由于ActivityRecord的finishing为true，activityIdleInternalLocked 会直接调用finishCurrentActivityLocked，所传入finishCurrentActivityLocked的参数也与上一次不同，是FINISH_IMMEDIATELY。函数运行中先从ActivityStackSupervisor中的各个列表mStoppingActivities、mGoingToSleepActivities、mWaitingVisibleActivities中将ActivityRecord移除，将ActivityRecord的state置为ActivityState.FINISHING.因为传入的FINISH_IMMEDIATELY，destroyActivityLocked会被调用，在这里将ActivityRecord从mFinishingActivities删除，从对应ProcessRecord的activities中删除，如果这个Activity 已经是ProcessRecord里的最后一个Actiivty，那么更新AMS的Lru列表和进程的oom，然后通知应用进行window销毁，Cursor清理等。ActivityRecord的nowVisible置为false,state置为ActivityState.DESTROYING。当应用端处理完以后，通过ActivityDestoryed函数通知AMS，AMS交给ActivityStack处理。ActivityStack将ActivityRecord的PendingIntent都清空，与该Activity相关的Service连接也都清理。清理Activity之后就是清理与该Activity有关的历史记录，在这里将ActivityRecord的state置为ActivityState.DESTROYED，WMS、AMS、TaskRecord都要清除这个ActivityRecord的引用，如果这个Activity是所在TaskRecord的最后一个Activity，这个TaskRecord也要从ActivityStack的TaskHistory中删除。如果删除TaskRecord后ActivityStack里也没有TaskRecord了，那么这个ActivityStack也要删除，下次再重新生成ActivityStack的时候，stackId就会递增1。

​	finishCurrentActivityLocked的第一次运行结束后，就开始显示之前的Activity也就是SecondActivity了（如果在这个过程中没有再启动新的Activity，但即使启动了其他Activity也没关系，result的发送要等到需要显示SecondActivity的时候才会进行），resumeTopActivitiesLocked将pause ForResultActivity过程中调整到ActivityStack最顶层的Activity显示出来，它会调用ActivityStack.resumeTopActivityInnerLocked，根据之前写的Activity显示过程的回调顺序，在onResume之前，onActivityResult就已经被执行了，而执行的时机就在resumeTopActivityInnerLocked：

```java	
 private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {   
	......
    		// Deliver all pending results.
                ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                "Delivering results to " + next + ": " + a);
                        next.app.thread.scheduleSendResult(next.appToken, a);
                    }
                    ......
                    next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,mService.isNextTransitionForward(), resumeAnimOptions);
                }
	......
 }
```

​	resumeTopActivityInnerLocked将SecondActivity对应ActivityRecord中的results列表整个传递给ApplicationThread的scheduleSendResult函数，这说明SecondActivity可以在restart的时候可以同时接收多个result数据。scheduleSendResult将results和Activity的token一起封装起来，scheduleSendResult会调用ActivityThread.deliverResults:

```java
 private void deliverResults(ActivityClientRecord r, List<ResultInfo> results) {
        final int N = results.size();
        for (int i=0; i<N; i++) {
            ResultInfo ri = results.get(i);
            try {
                if (ri.mData != null) {
                    ri.mData.setExtrasClassLoader(r.activity.getClassLoader());
                    ri.mData.prepareToEnterProcess();
                }
                r.activity.dispatchActivityResult(ri.mResultWho,
                        ri.mRequestCode, ri.mResultCode, ri.mData);
                ......
   }
```

​	在这里遍历要传给SecondActivity的所有ResultInfo，并将它们传给Activity处理:

```java
    void dispatchActivityResult(String who, int requestCode,int resultCode, Intent data) {
        mFragments.noteStateNotSaved();
        if (who == null) {
            onActivityResult(requestCode, resultCode, data);
        } else if (who.startsWith(REQUEST_PERMISSIONS_WHO_PREFIX)) {
            who = who.substring(REQUEST_PERMISSIONS_WHO_PREFIX.length());
            if (TextUtils.isEmpty(who)) {
                dispatchRequestPermissionsResult(requestCode, data);
            } else {
                Fragment frag = mFragments.findFragmentByWho(who);
                if (frag != null) {
                    dispatchRequestPermissionsResultToFragment(requestCode, data, frag);
                }
            }
        } else if (who.startsWith("@android:view:")) {
            ArrayList<ViewRootImpl> views = WindowManagerGlobal.getInstance().getRootViews(
                    getActivityToken());
            for (ViewRootImpl viewRoot : views) {
                if (viewRoot.getView() != null
                        && viewRoot.getView().dispatchActivityResult(
                                who, requestCode, resultCode, data)) {
                    return;
                }
            }
        } else {
            Fragment frag = mFragments.findFragmentByWho(who);
            if (frag != null) {
                frag.onActivityResult(requestCode, resultCode, data);
            }
        }
    }
```

​	在传递给具体的接收者的时候，dispatchActivityResult根据who对接收Result的对象类型进行区分：

​	1.who为null：接收者是Activity对象

​	2.who以“@android:requestPermissions:”开头，接收者是Fragment对象，只用于Fragment向PackageManagerService进行权限请求

​	3.who以"@android:view:"开头，接收者是View对象

​	4.以上都不是则根据who查找是否有对应的Fragment，若有则接收者是查找到的Fragment对象

​	由此可知，我们startActivityForResult不是一定要在Activity中，Fragment和View也可以，同样的，接收函数onActivityResult也可以写在Fragment和View中，不过View的startActivityForResult是一个内部方法，普通应用是不能调用的。

​	到这里onActivityResult就可以被调用了，流程也介绍完了，再见。