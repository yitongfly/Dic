## SingleInstance Activity的回退过程

​	singleInstance Activity在Android系统中只能有一个实例，且它自身要单独存在与一个Task中，无论显示启动还是隐式启动，只要系统中有这个Activity的实例存在，那么就会将这个Activity所在的Task置于前端，并将这个Activity显示出来（同时传递给它一个new intent）。

​	它的启动过程比较简单，比较诡异的是回退过程。假设有N个Activity：HomeActivity（即系统的launcher），RecentActivity，应用P的ActivityA，SingleInstanceActivity，ActivityB，应用P1的CallSingleActivity。

1）ActivityA，SingleInstanceActivity，ActivityB的启动和回退：

​	在系统中没有应用P的进程的时候，通过launcher打开应用P，顺序是：

​	HomeActivity-->ActivityA-->SingleInstanceActivity-->ActivityB

​	对于这种打开顺序，回退顺序却是：

​	ActivityB-->>ActivityA-->>SingleInstanceActivity-->>HomeActivity

​	ActivityB没有回退到SingleInsActivity，为什么？如果启动ActivityB以后不按返回键，而是按Home键回到HomeActivity，再通过launcher启动应用P，我们看到的Activity是ActivityB，此时回退的话，顺序变成了：

​	ActivityB-->>ActivityA-->>HomeActivity

​	为什么第一种情况ActivityA退到SingleInstanceActivity，而第二种退到了HomeActivity	呢？

​	先看Activity创建过程中，ActivityStack、TaskRecord和Activity的结构：

​	系统在开机后会自动创建一个ActivityStack，里面存放的是系统的是系统的HomeActivity;再启动ActivityA的时候，系统会再创建一个新的ActivityStack和一个TaskRecord，用来放ActivityA，给这个TaskRecord一个代号TaskCommon，ActivityA启动SingleInsActivity的时候，由于SingleInsActivity的特殊性，系统会再为它创建一个TaskRecord，并存放SingleInsActivity，给这个TaskRecord一个代号TaskSingle。由于ActivityA是由HomeActivity启动的，系统会为它所在Task中的mTaskToReturnTo变量赋值为HOME_ACTIVITY_TYPE，这表示当这个Task退出时，返回HomeActivity。SingleInsActivity是被其他应用Activity启动的，所在Task的mTaskToReturnTo变量被赋值为APPLICATION_ACTIVITY_TYPE，表示task退出时先返回到启动它的Activity。

​	当SingleInsActivity再启动ActivityB的时候，根据TaskRecord的taskAffinity可以找到与ActivityB的taskAffinity相同的Task，ActivityB就会被加入到这个Task中（ActivityB若通过"android:taskAffinity"指定要进入的Task则寻找对应的Task或新键Task），也就是这里的TaskCommon（因为默认的taskAffinity就是应用的packageName），在启动过程中会将TaskCommon置顶，要调用insertTaskAtTop函数，这个函数会对被启动Activity所在Task退出时应该返回哪里进行重新判断，因为ActivityB是由应用启动的，这个函数会把TaskCommon的mTaskToReturnTo重新赋值为APPLICATION_ACTIVITY_TYPE。

```java
insertTaskAtTop(TaskRecord task, ActivityRecord newActivity){
     // If the moving task is over home stack, transfer its return type to next task
  (1) if (task.isOverHomeStack()) {
            final TaskRecord nextTask = getNextTask(task);
            if (nextTask != null) {
                nextTask.setTaskToReturnTo(task.getTaskToReturnTo());
            }
        }
  (2) if (isOnHomeDisplay()) {
            ActivityStack lastStack = mStackSupervisor.getLastStack();
            final boolean fromHome = lastStack.isHomeStack();
            if (!isHomeStack() && (fromHome || topTask() != task)) {
                task.setTaskToReturnTo(fromHome
                        ? lastStack.topTask() == null
                                ? HOME_ACTIVITY_TYPE
                                : lastStack.topTask().taskType
                        : APPLICATION_ACTIVITY_TYPE);
            }
        } else {
            task.setTaskToReturnTo(APPLICATION_ACTIVITY_TYPE);
        }
    .......
}
```

​	（1）如果要置顶的task的mTaskToReturnTo为HOME_ACTIVITY_TYPE，那么将该task的mTaskToReturnTo传递给下一个Task，这里的nextTask是比task的index大1的task，表示task被置顶后，从nextTask回退时不是返回到task里的Activity而是回退到HomeActivity。

​	因此，当把TaskCommon置顶的时候，它的mTaskToReturnTo  HOME_ACTIVITY_TYPE被传递给了TaskSingle，TaskSingle变成了退出时返回到HomeActivity。

​	（2）判断要置顶的task是被谁移动的，如果是HomeActivity，将task的mTaskToReturnTo置为HOME_ACTIVITY_TYPE，表示它退出时应该回到HomeActivity；如果是应用Activity，将task的mTaskToReturnTo置为APPLICATION_ACTIVITY_TYPE，表示它退出时应该回到应用Activity。

​	Activity的finish过程包括了pause和destory两部分，真正使得这个mTaskToReturnTo能够决定返回到哪里的函数是在finish过程中调用的ActivityStack.adjustFocusedActivityLocked，这个是在pause Activity之前：

```java
    private void adjustFocusedActivityLocked(ActivityRecord r, String reason) {
        if (mStackSupervisor.isFrontStack(this) && mService.mFocusedActivity == r) {
            ActivityRecord next = topRunningActivityLocked(null);
            final String myReason = reason + " adjustFocus";
            if (next != r) {
                final TaskRecord task = r.task;
        	(**)if (r.frontOfTask && task == topTask() && task.isOverHomeStack()){
                    if (!mFullscreen
                            && adjustFocusToNextVisibleStackLocked(null, myReason)) {
                        return;
                    }
                    if (mStackSupervisor.moveHomeStackTaskToTop(
                            task.getTaskToReturnTo(), myReason)) {
                        // Activity focus was already adjusted. Nothing else to do...
                        return;
                    }
                }
            }
            final ActivityRecord top = mStackSupervisor.topRunningActivityLocked();
            if (top != null) {
                mService.setFocusedActivityLocked(top, myReason);
            }
        }
    }
```

​	**处的代码是对finish过程中Task状态的判断，如果被finish的Activity r在Task中的index为0，它的task已经置顶，且task销毁后要返回HomeActivity（即Task的mTaskToReturnTo为HOME_ACTIVITY_TYPE），那么通过moveHomeStackTaskToTop函数将HomeStack的HomeActivity赋值给AMS的mFocusedActivity，AMS又会调用Activity StackSupervisor的setFocusedStack函数将HomeActivity所在的HomeStack置顶，这样，当查找topRunning的Activity时（也就是finish了当前Activity之后要显示的Activity）找到的就是HomeActivity。

​	pause结束后AMS的activityPaused会被调用，进入到completePauseLocked函数中，在这个函数里要显示此时在topStack中的top Activity：

```java
private void completePauseLocked(boolean resumeNext) {
    ......
	if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            }
   ......
}
```

​	从传入resumeTopActivitiesLocked的参数可以看出，要resume的是AMS中FocusedStack（也就是系统中的顶层ActivityStack）的Activity，下一步就是找到那个Activity，而这个Activity是在ActivityStack的resumeTopActivityInnerLocked函数中找到的：

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
    ......
         final ActivityRecord next = topRunningActivityLocked(null);
    ......
}
```

​	在AMS里，系统中的Activity存放在TaskRecord的mActivities里，用户越早看到的Activity在mActivities中index越小，用户越晚看到的Activity的index越大。next的查找过程就是按照index从大到小遍历系统中的每一个ActivityStack的mTaskHistory的mActivities中的每一个Activity，直到找到一个不为空，finishing不为true的Activity。

​	在第一种情况下，当启动ActivityA的时候，系统为它新键了一个TaskRecord，我们以TaskCommon表示，并且由于上一个显示的Activity所在的ActivityStack是HomeStack，所以系统为TaskCommon的mTaskToReturnTo赋值为HOME_ACTIVITY_TYPE，表示希望这个TaskRecord退出时回到HomeActivity。当启动SingleInsActivity的时候，系统也为其新键了一个TaskRecord，我们以TaskSingle表示，mTaskToReturnTo取值为APPLICATION_ACTIVITY_TYPE，因为启动它时的那个ActivityStack是应用的ActivityStack。当启动ActivityB的时候，如果TaskCommon原来是要返回HomeActiivty的，那就将这个返回类型传递给在它之上的TaskRecord，希望它代替自己返回到HomeActivity，这样不会出现不断显示下层TaskRecord里的Activity而无法返回到HomeActivity的情况，并且由于此次启动ActivityB的时候，将TaskCommon置顶前的ActivityStack不是HomeStack而是应用的ActivityStack了，TaskCommon的mTaskToReturnTo就变成了APPLICATION_ACTIVITY_TYPE，表示它退出时应该回到应用的Activity。

​	首先看ActivityB的回退，因为在ActivityB所在的TaskRecord中，当进行到completePauseLocked以后，ActivityB的finishing已经被置为true了，那么在遍历ActivityB所在的TaskRecord时，找到的next就是ActivityA，所以系统会resume的是ActivityA，也就是从ActivityB回退到ActivityA。

​	回退ActivityA的过程与此相同，因为ActivityA是所在TaskRecor的最后一个Activity且finishing为true，那么这个TaskRecord就被跳过，查找下一个TaskRecord，正是SingleInstanceActivity所在的TaskRecord，SingleInsActivity也符合topRunningActivity的条件，所以系统resume的就是SingleInsActivity。

​	而SingleInsActivity的回退则不一样了，因为在启动ActivityB的时候，TaskCommon已经将它的mTaskToReturnTo传递给TaskSingle，此时TaskSingle的mTaskToReturnTo变成了HOME_ACTIVITY_TYPE，按了back键以后，在finish SingleInsActivity的流程里，因为SingleInsActivity是所在Task的root，并且表示要回退到HomeActivity（因为Task的mTaskToReturnTo是HOME_ACTIVITY_TYPE），AMS会把HomeStack的置顶，同时将HomeActivity置顶，当SingleInsActivity被pause以后，显示的就是HomeActivity了。

​	那么为什么第二种情况中从ActivityA回到的却是HomeActivity呢？

​	因为当HomeActivity再次启动ActivityA的时候，是将它所在的Task置顶了，所以显示的ActivityB，由于它是由HomeActivity启动的，所以TaskCommon的mTaskToReturnTo并且在Activity回退的过程中没有变化，所以再回退ActivityA的情况就与上面回退SingleInsActivity是一样的了，直接回到了HomeActivity。

<video src="Single_Instance.m4v"></video>

​	2)再考虑一个场景，这次是两个进程P和P1，P1里有个CallSingleActivity，它可以通过Intent打开上面的SingleInsActivity，SingleInsActivity运行在P进程里

​	先打开CallSingleActivity，再打开SingleInsActivity，打开顺序是

​	CallSingleActivity-->SingleInsActivity

​	回退顺序是

​	SingleInsActivity-->>CallSingleActivity

​	这个看着很正常，如果打开SingleInsActivity以后，打开一下recent列表，再从recent列表恢复SingleInsActivity：

​	CallSingleActivity-->SingleInsActivity-->RecentActivity-->SingleInsActivity

​	回退顺序是

​	SingleInsActivity-->>HomeActivity

​	为什么会突然跳到HomeActiivty？CallSingleActivity和RecentActivity呢？

​	这个其实原因和上面是一样的，第一种情况的回退与1）中从ActivityB回退到SingleInsActivity的原理相同，上一个TaskRecord里的Activity全部finish了且上一个TaskRecord的mTaskToReturnTo是APPLICATION_ACTIVITY_TYPE，显示的下一个Activity就是同ActivityStack中的下一个TaskRecord里的Activity，也就从CallSingleActivity回退到了SingleInsActivity。

​	但是在第二中情况中，由于显示了一次RecentActivity，而RecentActivity是运行在HomeStack中的，当再次把SingleInsActivity置顶的时候，由于仍然要运行insertTaskAtTop函数，insertTaskAtTop里的fromHome为true了，TaskCommon的mTaskToReturnTo就要重新赋值了，RecentActivity所在Task的taskType是RECENTS_ACTIVITY_TYPE，但是AMS对这种类型Task的处理方式与HOME_ACTIVITY_TYPE非常相似，恢复SingleInsActivity的时候，TaskSingle的mTaskToReturnTo不会被赋值为RECENTS_ACTIVITY_TYPE而是被赋值为HOME_ACTIVITY_TYPE，所以当SingleInsActivity回退的时候也就回退到了HomeActivity。
