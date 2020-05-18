---
layout:     post
title:      ActivityRecord模块
subtitle:   
date:       2020-05-17
author:     DoubleWay
header-img: img/home-bg-o.jpg
catalog: 	 true
tags:
    - Android
---

# ActivityRecord

ActivityRecord是System_setver进程中的对象，通过图来看：

![2020-05-17-1.1](/img/2020-05-17/2020-05-17-1.1.png)

ActivityRecord是AMS调度Activity的基本单位，它记录着在Androidmanifest.xml定义的Activity的静态信息，同时，也要记录Activity被调度时的状态变化

一个ActivityRecord对应一个Activity，包含一个Activity的所有信息，但是一个activity可能对应多个ActivityRecord，因为一个activity可能被多次启动，这个要看其的启动模式。

TaskRecord包含task的信息，包含一个或多个ActivityRecord，具有栈先进后出的信息。

ActivityStack主要是管理TaskRecord，包含多个TaskRecord

### ActivityRecord

ActivityRecord中含有大量的成员变量，包含了一个activity的信息

```java
public WindowProcessController app;      // if non-null, hosting application，跑在那个进程
private TaskRecord task;        // the task this is in 跑在哪个task
private ActivityState mState;    // current state we are in activity的状态
public ApplicationInfo appInfo; // information about activity's app 跑在那个app
final ComponentName mActivityComponent;  // the intent component, or target of an alias 组件名
public final String packageName; // the package implementing intent's component 包名
int launchMode;         // the launch mode activity attribute 启动模式
int userId //该Activity运行在哪个用户id
```

省略了许多的成员变量，通过变量task指向TaskRecord

### TaskRecord

内部维护一个**ArrayList<ActivityRecord>**来保存Activity

```java
class TaskRecord extends ConfigurationContainer {
    private static final String TAG = TAG_WITH_CLASS_NAME ? "TaskRecord" : TAG_ATM;
    private static final String TAG_ADD_REMOVE = TAG + POSTFIX_ADD_REMOVE;
     。。。。。。。。。。
    final int taskId;       // Unique identifier for this task. 任务ID
    /** List of all activities in the task arranged in history order */
    final ArrayList<ActivityRecord> mActivities;  //使用一个ArrayList来保存所有的ActivityRecord
     /** Current stack. Setter must always be used to update the value. */
    private ActivityStack mStack; //TaskRecord所在的ActivityStack
    
     //添加Activity到指定的索引位置
        void addActivityAtIndex(int index, ActivityRecord r) {
            //...

            r.setTask(this);//为ActivityRecord设置TaskRecord，就是这里建立的联系

            //...
            
            index = Math.min(size, index);
            mActivities.add(index, r);//添加到mActivities
            
            //...
        }
```

- 可以看到`TaskRecord`中使用了一个ArrayList来保存所有的`ActivityRecord`。
- 其中的mStack变量指定了属于那个ActivityStack

### ActivityStack

在内部创建了一个ArrayList<TaskRecord> mTaskHistory，来保管所有的TaskRecord

```java
class ActivityStack extends ConfigurationContainer {
    private static final String TAG = TAG_WITH_CLASS_NAME ? "ActivityStack" : TAG_ATM;
    private static final String TAG_ADD_REMOVE = TAG + POSTFIX_ADD_REMOVE;
    //.....
    
    /**
     * The back history of all previous (and possibly still
     * running) activities.  It contains #TaskRecord objects.
     */
    private final ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();//ArrayList来保管所有的TaskRecord
    
    //.....
    TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent,
                                    IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                    boolean toTop, int type) {
                                    
            //创建一个task
            TaskRecord task = new TaskRecord(mService, taskId, info, intent, voiceSession, voiceInteractor, type);
            
            //将task添加到ActivityStack中去
            addTask(task, toTop, "createTaskRecord");

            //其他代码略

            return task;
        }
      //......
        void addTask(final TaskRecord task, int position, boolean schedulePictureInPictureModeChange,
            String reason) {
        // TODO: Is this remove really needed? Need to look into the call path for the other addTask
        mTaskHistory.remove(task); //有的话先移除
            
        //其他代码省略
            
        mTaskHistory.add(position, task);//添加task到mTaskHistory
        task.setStack(this);//设置task的ActivityStack

         //其他代码省略
    }
```

可以看到，通过创建了一个ArrayList-->mTaskHistory来管理TaskRecord。

```java
/** Run all ActivityStacks through this */
protected final ActivityStackSupervisor mStackSupervisor;
```

其中还有一个变量ActivityStackSupervisor，ActivityStackSupervisor是用来管理ActivityStack的，ActivityStack是由ActivityStackSupervisor来创建。

### ActivityStackSupervisor

ActivityStackSupervisor是ActivityStack的管理者

```java
  if (inMultiWindowMode && !task.isResizeable()) {
   //.....省略代码
        stack = stack.getDisplay().createStack(
                WINDOWING_MODE_FULLSCREEN, stack.getActivityType(), toTop);
    }
    return stack;
}
```

在ActivityStackSupervisor中调用了**ActivityStack**的**getDisplay（）**方法，来获得**Activitydisplay**，调用它的**createStack**方法

```java
<T extends ActivityStack> T createStack(int windowingMode, int activityType, boolean onTop) {

    //....省略代码

    final int stackId = getNextStackId();
    return createStackUnchecked(windowingMode, activityType, stackId, onTop);
}
```

```java
@VisibleForTesting
<T extends ActivityStack> T createStackUnchecked(int windowingMode, int activityType,
        int stackId, boolean onTop) {
    if (windowingMode == WINDOWING_MODE_PINNED && activityType != ACTIVITY_TYPE_STANDARD) {
        throw new IllegalArgumentException("Stack with windowing mode cannot with non standard "
                + "activity type.");
    }
    //创建一个ActivityStack
    return (T) new ActivityStack(this, stackId,
            mRootActivityContainer.mStackSupervisor, windowingMode, activityType, onTop);
}
```

可以看到最后返回了ActivityStack。

AMS在初始化的时候会创建一个ActivityStackSupervisor对象
