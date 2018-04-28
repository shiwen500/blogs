Activity的LaunchMode和task

# LaunchMode

## standard

在AndroidManifest中声明
```java
<activity android:name=".lm_standard.ActivityA"
            android:launchMode="standard"/>
            
// java
public class ActivityA extends Activity {

    private static final String TAG = "lm_standard_ActivityA";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_standard_a);

        Log.d(TAG, "onCreate " + this + " tasksId -> " + getTaskId());
    }

    public void startA(View view) {

        Log.d(TAG, "startA from " + this);

        Intent activity = new Intent(this, ActivityA.class);
        startActivity(activity);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        Log.d(TAG, "onDestroy " + this);
    }
}
```

进入ActivityA后，连续触发了2次startA，然后陆续返回3次：

```java
04-26 17:49:03.065 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onCreate com.seven.www.activityusage.lm_standard.ActivityA@c2ba76d tasksId -> 280
04-26 17:49:04.225 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: startA from com.seven.www.activityusage.lm_standard.ActivityA@c2ba76d
04-26 17:49:04.337 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onCreate com.seven.www.activityusage.lm_standard.ActivityA@9990b77 tasksId -> 280
04-26 17:49:05.534 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: startA from com.seven.www.activityusage.lm_standard.ActivityA@9990b77
04-26 17:49:05.650 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onCreate com.seven.www.activityusage.lm_standard.ActivityA@286e87b tasksId -> 280

04-26 17:49:11.312 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onDestroy com.seven.www.activityusage.lm_standard.ActivityA@286e87b
04-26 17:49:13.033 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onDestroy com.seven.www.activityusage.lm_standard.ActivityA@9990b77
04-26 17:49:15.450 25218-25218/com.seven.www.activityusage D/lm_standard_ActivityA: onDestroy com.seven.www.activityusage.lm_standard.ActivityA@c2ba76d
```

可以看出每次都会启动一个全新的Activity，并且放在同一个task里面，返回也是按照创建顺序的倒叙逐个销毁。

## singleTop

添加了两个Activity, 分别为ActivityB ActivityC，其中ActivityB launchMode为singleTop, ActivityC 的LaunchMode为standard.

```java
// ActivityB
public class ActivityB extends Activity {

    private static final String TAG = "lm_singleTop";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_singletop_b);

        Log.d(TAG, "onCreate " + this + "  taskId -> " + getTaskId());
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);

        Log.d(TAG, "onNewIntent " + this + " taskId -> " + getTaskId());
    }

    public void startB(View view) {
        Log.d(TAG, "startB");
        Intent activity = new Intent(this, ActivityB.class);
        startActivity(activity);
    }

    public void startC(View view) {
        Log.d(TAG, "startC");
        Intent activity = new Intent(this, ActivityC.class);
        startActivity(activity);
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy " + this);
    }
}
// ActivityC
public class ActivityC extends Activity {

    private static final String TAG = "lm_singleTop";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_singletop_c);

        Log.d(TAG, "onCreate " + this + "  taskId -> " + getTaskId());
    }

    public void startB(View view) {
        Log.d(TAG, "startB");
        Intent activity = new Intent(this, ActivityB.class);
        startActivity(activity);
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy " + this);
    }
}
```

进入ActivityB后触发了一次startB，再点击了一下返回：
```java
04-27 09:10:04.838 25444-25444/com.seven.www.activityusage D/lm_singleTop: onCreate com.seven.www.activityusage.lm_singletop.ActivityB@9a6d920  taskId -> 285
04-27 09:10:08.832 25444-25444/com.seven.www.activityusage D/lm_singleTop: startB
04-27 09:10:08.847 25444-25444/com.seven.www.activityusage D/lm_singleTop: onNewIntent com.seven.www.activityusage.lm_singletop.ActivityB@9a6d920 taskId -> 285

04-27 09:10:15.019 25444-25444/com.seven.www.activityusage D/lm_singleTop: onDestroy com.seven.www.activityusage.lm_singletop.ActivityB@9a6d920 taskId -> 284


```

进入ActivityB依次触发startC -> startB，再点击3次返回：
```java
04-27 09:11:42.972 25444-25444/com.seven.www.activityusage D/lm_singleTop: onCreate com.seven.www.activityusage.lm_singletop.ActivityB@78d9444  taskId -> 285
04-27 09:11:44.321 25444-25444/com.seven.www.activityusage D/lm_singleTop: startC
04-27 09:11:44.408 25444-25444/com.seven.www.activityusage D/lm_singleTop: onCreate com.seven.www.activityusage.lm_singletop.ActivityC@4d57c36  taskId -> 285
04-27 09:11:46.010 25444-25444/com.seven.www.activityusage D/lm_singleTop: startB
04-27 09:11:46.100 25444-25444/com.seven.www.activityusage D/lm_singleTop: onCreate com.seven.www.activityusage.lm_singletop.ActivityB@33043b3  taskId -> 285
04-27 09:12:16.328 25444-25444/com.seven.www.activityusage D/lm_singleTop: onDestroy com.seven.www.activityusage.lm_singletop.ActivityB@33043b3
04-27 09:12:22.529 25444-25444/com.seven.www.activityusage D/lm_singleTop: onDestroy com.seven.www.activityusage.lm_singletop.ActivityC@4d57c36
04-27 09:12:24.426 25444-25444/com.seven.www.activityusage D/lm_singleTop: onDestroy com.seven.www.activityusage.lm_singletop.ActivityB@78d9444
```

可以看出，如果ActivityB处于栈顶，那么startB不会新建一个Activity，而是触发栈顶Activity的onNewIntent方法。如果ActivityB不处于栈顶，那么startB的结果
和standard的模式是一致的。

## singleTask

添加2个Activity，ActivityD和ActivityE，D为singleTask，E为standard:
```java
// ActivityD
public class ActivityD extends Activity {

    private static final String TAG = "lm_singleTask";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_singletask_d);

        Log.d(TAG, "onCreate " + this + " taskId -> " + getTaskId());
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Log.d(TAG, "onNewIntent " + this + " taskId -> " + getTaskId());
    }

    public void startE(View view) {
        Intent activity = new Intent(this, ActivityE.class);
        startActivity(activity);
    }

    public void startD(View view) {
        Intent activity = new Intent(this, ActivityD.class);
        startActivity(activity);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy " + this);
    }
}
// ActivityE
public class ActivityE extends ActivityD {
}
```

进入D，触发2次startE，再触发startD：
```java
04-27 09:43:07.850 26815-26815/com.seven.www.activityusage D/lm_singleTask: onCreate com.seven.www.activityusage.lm_singletask.ActivityD@7dd0052 taskId -> 287
04-27 09:43:27.325 26815-26815/com.seven.www.activityusage D/lm_singleTask: onCreate com.seven.www.activityusage.lm_singletask.ActivityE@b6edeac taskId -> 287
04-27 09:43:28.259 26815-26815/com.seven.www.activityusage D/lm_singleTask: onCreate com.seven.www.activityusage.lm_singletask.ActivityE@5194736 taskId -> 287
04-27 09:43:34.044 26815-26815/com.seven.www.activityusage D/lm_singleTask: onDestroy com.seven.www.activityusage.lm_singletask.ActivityE@b6edeac
04-27 09:43:34.069 26815-26815/com.seven.www.activityusage D/lm_singleTask: onNewIntent com.seven.www.activityusage.lm_singletask.ActivityD@7dd0052 taskId -> 287
04-27 09:43:34.417 26815-26815/com.seven.www.activityusage D/lm_singleTask: onDestroy com.seven.www.activityusage.lm_singletask.ActivityE@5194736
```

可以看出，当启动D时，如果D已经在栈内，那么不会重新创建D，而是触发D的onNewIntent，并且全部销毁处于D头上的其他Activity。如果D处于栈顶，则退化成singleTop模式。

## singleInstance

添加2个Activity，分别为ActivityF和ActivityG，F为singleInstance，G为standard

```java
// ActivityF
public class ActivityF extends Activity {

    private static final String TAG = "lm_singleInstance";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_singleinstance_f);
        Log.d(TAG, "onCreate " + this + " taskId -> " + getTaskId());
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Log.d(TAG, "onNewIntent " + this + " taskId -> " + getTaskId());
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy " + this);
    }

    public void startF(View view) {
        Log.d(TAG, "startF");
        Intent activity = new Intent(this, ActivityF.class);
        startActivity(activity);
    }

    public void startG(View view) {
        Log.d(TAG, "startG");
        Intent activity = new Intent(this, ActivityG.class);
        startActivity(activity);
    }
}
// ActivityG
public class ActivityG extends ActivityF {
}
```

进入G，触发startF -> startG -> startF -> startG，然后陆续点击返回

```java
04-27 10:52:34.069 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onCreate com.seven.www.activityusage.lm_singleinstance.ActivityG@d2e261f taskId -> 313
04-27 10:52:35.310 29785-29785/com.seven.www.activityusage D/lm_singleInstance: startF
04-27 10:52:35.413 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onCreate com.seven.www.activityusage.lm_singleinstance.ActivityF@372e491 taskId -> 314
04-27 10:52:46.427 29785-29785/com.seven.www.activityusage D/lm_singleInstance: startG
04-27 10:52:46.502 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onCreate com.seven.www.activityusage.lm_singleinstance.ActivityG@233f873 taskId -> 313
04-27 10:54:17.462 29785-29785/com.seven.www.activityusage D/lm_singleInstance: startF
04-27 10:54:17.494 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onNewIntent com.seven.www.activityusage.lm_singleinstance.ActivityF@372e491 taskId -> 314
04-27 10:56:53.108 29785-29785/com.seven.www.activityusage D/lm_singleInstance: startG
04-27 10:56:53.188 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onCreate com.seven.www.activityusage.lm_singleinstance.ActivityG@9e4a99a taskId -> 313

04-27 10:58:05.029 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onDestroy com.seven.www.activityusage.lm_singleinstance.ActivityG@9e4a99a
04-27 10:58:06.689 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onDestroy com.seven.www.activityusage.lm_singleinstance.ActivityG@233f873
04-27 10:58:10.425 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onDestroy com.seven.www.activityusage.lm_singleinstance.ActivityG@d2e261f
04-27 10:58:25.168 29785-29785/com.seven.www.activityusage D/lm_singleInstance: onDestroy com.seven.www.activityusage.lm_singleinstance.ActivityF@372e491
```

可以看出，启动F时，如果F不存在，那么创建新的task，将F加入新的task中；如果F存在，那么就触发F的onNewIntent。从F启动G时，系统切换到G默认的任务，再重新创建G。
因此可以知道，singleInstance的Activity独占一个task，即一个task如果含有一个singleInstance的Activity，那么就不能再存在其他Activity了。

# FLAG_ACTIVITY_XXX 标识 和 taskAffinity

默认情况下，每个Activity的taskAffinity都是包名，Android的最近任务是按照taskAffinity来进行分组的，不同的task如果taskAffinity相同，那么将放在同一个卡片组里面。也就是说，最近任务不是显示每一个任务，而是按照不同的taskAffinity显示其靠前的任务的最顶层Activity

假设存在3个task：

task1: A -> B -> C    (taskAffinity1)

task2: D -> E         (taskAffinity2)

task3: G              (taskAffinity2)

因为只存在2个taskAffinity，所以在最近任务里面只有2个卡片， 卡片1显示的 C，卡片2则需要根据实际情况而定：

1. 如果 task2在task stack中比task3靠前，那么显示的就是E
2. 反之 显示的是G

查看那个task靠前，可以通过`adb shell dumpsys activity activities`查看.

注意：如果我们在最近任务里面选择了某个task，那么系统仅仅是将该task调度到最前面，但是和这个task具有相同的taskAffinity的task被放到了最近任务和系统桌面的下面，也就是说，我们永远无法通过返回按键或者最近任务按键 回到那些没显示在最近任务里的task了。因此一般来说，设置taskAffinity都不是什么好事情。尽量避免。

## FLAG_ACTIVITY_NEW_TASK

如果目标Activity已经在一个task（不能是默认task）中运行了，并且，这次startActivity查找到的任务就是该task，那么这次操作仅仅是将该任务切换到前台，不会创建新的Activity。
如果目标Activity不存在：

 1） 如果目标Activity设置了taskAffinity，那么系统将创建一个全新的task，然后创建该Activity放在task的里面。
 2） 如果没有设置taskAffinity，那么系统不会创建task，而是创建该Activity放在默认task里面。(如果设置了singleInstance，那么还是会创建一个新task)
 
 这个FLAG需要和taskAffinity配合使用。
 
 在Android7.0以前，如果使用Service/Application/Broadcast/类型的Context
 
## FLAG_ACTIVITY_CLEAR_TOP

如果目标Activity在task中存在，那么销毁目标Activity上面的Activty，并触发onNewIntent.
如果不存在，那么创建一个新的，加入task栈顶。

这个FLAG和 launchMode:singleTask类似

## FLAG_ACTIVITY_SINGLE_TOP

这个FLAG和 launchMode:singleTop类似

## FLAG_ACTIVITY_NO_HISTORY

离开Activity后该Activity即销毁，该Activity的onActivityResult也不会触发。










