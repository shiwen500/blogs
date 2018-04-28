Service使用文档

#### 1. 服务的创建与启动

可以通过以下两种方式创建服务

1） startService(service);
如果服务不存在，那么首先创建服务，调用服务的onCreate()方法；如果服务存在，那么onCreate不会触发。
startService可以多次调用，但是一旦调用stopService，如果此时此服务没被其他组件绑定的话，服务将会销毁。

```java
    // Activity
    public void do_startService(View view) {
        Log.d(TAG, "do_startService");
        Intent service = new Intent(this, MyService.class);
        startService(service);

    }

    public void do_stopService(View view) {
        Log.d(TAG, "do_stopService");
        Intent service = new Intent(this, MyService.class);
        stopService(service);
    }
    
    // Service
    @Override
    public void onCreate() {
        Log.d(TAG, "onCreate " + this);
        super.onCreate();
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "onDestroy " + this);
        super.onDestroy();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "onStartCommand startId ->" + startId + " " + this);
        return START_STICKY;
    }
```

触发了2次do_startService后，再触发一次do_stopService：
```java
05-15 10:47:45.398 16504-16504/? D/Seven: do_startService
05-15 10:47:45.408 16504-16504/? D/Seven: onCreate com.seven.www.testserviceserver.MyService@4619f57
05-15 10:47:45.409 16504-16504/? D/Seven: onStartCommand startId ->1 com.seven.www.testserviceserver.MyService@4619f57

05-15 10:48:01.068 16504-16504/? D/Seven: do_startService
05-15 10:48:01.071 16504-16504/? D/Seven: onStartCommand startId ->2 com.seven.www.testserviceserver.MyService@4619f57

05-15 10:48:09.141 16504-16504/? D/Seven: do_stopService
05-15 10:48:09.144 16504-16504/? D/Seven: onDestroy com.seven.www.testserviceserver.MyService@4619f57
```

2） bindService(service, scn, Context.BIND_AUTO_CREATE);

第一个参数没什么好说的，第二个参数是一个ServiceConnection，如果使用同一个ServiceConnection对象进行多次绑定，那么只有一次绑定是有效的，
后续的绑定都不会触发它的onServiceConnected；第三个参数是一个标识位，如果使用Context.BIND_AUTO_CREATE，那么服务不存在时会自动创建，
如果使用0，那么这个绑定必须等到服务创建时才会触发绑定。注意：如果对已经解绑的ServiceConnection再次进行解绑，那么程序崩溃。

```java
    // Service
    ServiceConnection scn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "onServiceConnected -> " + name + "  " + service);
            messenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "onServiceDisconnected -> " + name);
        }
    };
    public void do_bindService(View view) {
        Log.d(TAG, "do_bindService");
        Intent service = new Intent(this, MyService.class);
        Log.d(TAG, "bindService result -> " + bindService(service, scn, 0));
    }

    public void do_unbindService(View view) {
        Log.d(TAG, "do_unbindService");
        unbindService(scn);
    }
    // Activity
    public void do_startService(View view) {
        Log.d(TAG, "do_startService");
        Intent service = new Intent(this, MyService.class);
        startService(service);

    }
```

调试标识位是0的情况，先触发do_bindService，再触发do_startService：

```java
05-15 11:05:31.501 16847-16847/? D/Seven: do_bindService
05-15 11:05:31.505 16847-16847/? D/Seven: bindService result -> true

05-15 11:06:19.909 16847-16847/? D/Seven: do_startService
05-15 11:06:19.919 16847-16847/? D/Seven: onCreate com.seven.www.testserviceserver.MyService@6d2167b
05-15 11:06:19.920 16847-16847/? D/Seven: onBind com.seven.www.testserviceserver.MyService@6d2167b
05-15 11:06:19.924 16847-16847/? D/Seven: onStartCommand startId ->1 com.seven.www.testserviceserver.MyService@6d2167b
05-15 11:06:19.924 16847-16847/? D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.Handler$MessengerImpl@1634cf1

```

可以看出，在绑定服务时，并不会自动创建服务，但是绑定结果是成功的；后续通过startService去创建服务，在服务创建后立刻触发了绑定onBind。

触发3次do_bindService，再触发do_startService：
```java
05-15 11:12:41.681 17040-17040/? D/Seven: do_bindService
05-15 11:12:41.685 17040-17040/? D/Seven: bindService result -> true
05-15 11:12:42.801 17040-17040/? D/Seven: do_bindService
05-15 11:12:42.803 17040-17040/? D/Seven: bindService result -> true
05-15 11:12:44.598 17040-17040/? D/Seven: do_bindService
05-15 11:12:44.600 17040-17040/? D/Seven: bindService result -> true
05-15 11:12:47.099 17040-17040/? D/Seven: do_startService
05-15 11:12:47.116 17040-17040/? D/Seven: onCreate com.seven.www.testserviceserver.MyService@1634cf1
05-15 11:12:47.118 17040-17040/? D/Seven: onBind com.seven.www.testserviceserver.MyService@1634cf1
05-15 11:12:47.121 17040-17040/? D/Seven: onStartCommand startId ->1 com.seven.www.testserviceserver.MyService@1634cf1
05-15 11:12:47.126 17040-17040/? D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.Handler$MessengerImpl@4619f57

```
可以看出，对于同一个ServiceConnection对象，进行多次绑定，只有第一次是有效的。

调试标识位是Context.BIND_AUTO_CREATE的情况
```java
public void do_bindService(View view) {
        Log.d(TAG, "do_bindService");
        Intent service = new Intent(this, MyService.class);
        Log.d(TAG, "bindService result -> " + bindService(service, scn, Context.BIND_AUTO_CREATE));
    }
```
先触发do_bindService，再触发do_startService：
```java
05-15 11:18:28.146 17288-17288/? D/Seven: do_bindService
05-15 11:18:28.151 17288-17288/? D/Seven: bindService result -> true
05-15 11:18:28.156 17288-17288/? D/Seven: onCreate com.seven.www.testserviceserver.MyService@678a44
05-15 11:18:28.157 17288-17288/? D/Seven: onBind com.seven.www.testserviceserver.MyService@678a44
05-15 11:18:28.160 17288-17288/? D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.Handler$MessengerImpl@e88b462

05-15 11:18:33.199 17288-17288/? D/Seven: do_startService
05-15 11:18:33.206 17288-17288/? D/Seven: onStartCommand startId ->1 com.seven.www.testserviceserver.MyService@678a44
```

可以看出，服务会在绑定时自动创建，并且，在startService时不会再次创建。

如果每次都使用一个全新的ServiceConnection来绑定服务：
```java
    // Activity
    ArrayList<ServiceConnection> scns = new ArrayList<>();

    public void do_bindService(View view) {
        Log.d(TAG, "do_bindService");
        Intent service = new Intent(this, MyService.class);
        ServiceConnection scn = new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                Log.d(TAG, "onServiceConnected -> " + name + "  " + service);
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {
                Log.d(TAG, "onServiceDisconnected -> " + name);
            }
        };

        if (bindService(service, scn, Context.BIND_AUTO_CREATE)) {
            scns.add(scn);
        };
    }

    public void do_unbindService(View view) {
        Log.d(TAG, "do_unbindService");
        if (scns.size() > 0) {
            unbindService(scns.remove(scns.size() - 1));
        }
    }
```
先触发2次do_bindService，再触发2次do_unbindService：
```java
04-19 16:16:05.950 3897-3897/com.seven.www.testserviceserver D/Seven: do_bindService
04-19 16:16:05.971 3897-3897/com.seven.www.testserviceserver D/Seven: onCreate com.seven.www.testserviceserver.MyService@8818907
04-19 16:16:05.972 3897-3897/com.seven.www.testserviceserver D/Seven: onBind com.seven.www.testserviceserver.MyService@8818907
04-19 16:16:05.975 3897-3897/com.seven.www.testserviceserver D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.Handler$MessengerImpl@951aa34

04-19 16:16:11.352 3897-3897/com.seven.www.testserviceserver D/Seven: do_bindService
04-19 16:16:11.367 3897-3897/com.seven.www.testserviceserver D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.Handler$MessengerImpl@951aa34

04-19 16:16:52.414 3897-3897/com.seven.www.testserviceserver D/Seven: do_unbindService
04-19 16:16:58.829 3897-3897/com.seven.www.testserviceserver D/Seven: do_unbindService
04-19 16:16:58.843 3897-3897/com.seven.www.testserviceserver D/Seven: onUnbind com.seven.www.testserviceserver.MyService@8818907
04-19 16:16:58.843 3897-3897/com.seven.www.testserviceserver D/Seven: onDestroy com.seven.www.testserviceserver.MyService@8818907
```

可以看出：
1. 对于同一个客户端多次绑定只会触发一次onBind，这里所谓的客户端是指一个进程。
2. 对于每一个ServiceConnection，在绑定成功后都会触发onServiceConnected。
3. 有多少次bindService，就必须对应多少次unbindService，当所有链接都解绑后，onUnbind才会触发。

有一个更特殊的情况，就是当一个服务的onUnbind被调用后，如果服务没有销毁，那么我们再次绑定服务，onBind不会被触发。

先do_startService，do_bindService和do_unbindService；再do_bindService和do_unbindService

```java
// onUnbind
    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "onUnbind " + this);
        return super.onUnbind(intent);
    }

// Log
04-24 14:10:25.715 12447-12447/? D/MyService: do_startService
04-24 14:10:25.755 805-1945/? I/ActivityManager: Start proc 12559:com.seven.www.testserviceserver:MyService/u0a107 for service com.seven.www.testserviceserver/.MyService
04-24 14:10:25.931 12559-12559/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:10:25.933 12559-12559/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@71ddf2e

04-24 14:10:29.941 12447-12447/? D/MyService: do_bindService
04-24 14:10:29.946 12559-12559/? D/MyService: onBind com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:10:29.962 12447-12447/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@c78f414
04-24 14:10:31.039 12447-12447/? D/MyService: do_unbindService
04-24 14:10:31.043 12559-12559/? D/MyService: onUnbind com.seven.www.testserviceserver.MyService@71ddf2e

04-24 14:10:33.489 12447-12447/? D/MyService: do_bindService
04-24 14:10:33.504 12447-12447/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@c78f414
04-24 14:10:35.841 12447-12447/? D/MyService: do_unbindService
```

可以看出，如果服务触发了onUnbind，那么再也不会触发onBind了（在下次绑定时），但是如果我们想要监听这种场景也是可以的，将onUnbind的返回值修改成true，
那么下一次绑定时，onRebind就会触发，再次解绑时onUnbind依然会被调用。每次onUnbind的返回值，只能决定下一次绑定时onRebind是否会被调用。比如第一次onUbind
时返回了true，但是下一次onUbind时返回了false，那么第三次（及以后）绑定都不会触发onRebind了。注意：触发onBind/onRebind，才会触发onUnbind。

```java
    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "onUnbind " + this);
        return true;
    }
```

先do_startService，do_bindService和do_unbindService；再do_bindService和do_unbindService

```java
04-24 14:26:43.146 13654-13654/? D/MyService: do_startService
04-24 14:26:43.166 805-6467/? I/ActivityManager: Start proc 13708:com.seven.www.testserviceserver:MyService/u0a107 for service com.seven.www.testserviceserver/.MyService
04-24 14:26:43.323 13708-13708/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:26:43.326 13708-13708/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:26:44.206 13654-13654/? D/MyService: do_bindService
04-24 14:26:44.211 13708-13708/? D/MyService: onBind com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:26:44.225 13654-13654/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@c78f414
04-24 14:26:44.872 13654-13654/? D/MyService: do_unbindService
04-24 14:26:44.876 13708-13708/? D/MyService: onUnbind com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:26:46.255 13654-13654/? D/MyService: do_bindService
04-24 14:26:46.259 13708-13708/? D/MyService: onRebind com.seven.www.testserviceserver.MyService@71ddf2e
04-24 14:26:46.272 13654-13654/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@c78f414
04-24 14:26:46.905 13654-13654/? D/MyService: do_unbindService
04-24 14:26:46.909 13708-13708/? D/MyService: onUnbind com.seven.www.testserviceserver.MyService@71ddf2e
```

其实上面的测试只是在同一个APP的同一个进程中，并不能说明跨进程的情况。下面一节是关于跨进程绑定的情况。

#### 2. 服务的跨进程访问

我们将MyService声明运行在其他进程：
```java
<service android:name=".MyService"
   android:process=":MyService"/>
```

先触发2次do_bindService，再触发2次do_unbindService：
```java
// 客户端进程
04-19 17:23:54.395 6182-6182/com.seven.www.testserviceserver D/Seven: do_bindService
04-19 17:23:54.673 6182-6182/com.seven.www.testserviceserver D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@951aa34
04-19 17:23:55.277 6182-6182/com.seven.www.testserviceserver D/Seven: do_bindService
04-19 17:23:55.292 6182-6182/com.seven.www.testserviceserver D/Seven: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@951aa34
04-19 17:24:05.826 6182-6182/com.seven.www.testserviceserver D/Seven: do_unbindService
04-19 17:24:07.079 6182-6182/com.seven.www.testserviceserver D/Seven: do_unbindService
// 服务进程
04-19 17:23:54.668 6235-6235/com.seven.www.testserviceserver:MyService D/Seven: onCreate com.seven.www.testserviceserver.MyService@2ac2e7c
04-19 17:23:54.669 6235-6235/com.seven.www.testserviceserver:MyService D/Seven: onBind com.seven.www.testserviceserver.MyService@2ac2e7c
04-19 17:24:07.085 6235-6235/com.seven.www.testserviceserver:MyService D/Seven: onUnbind com.seven.www.testserviceserver.MyService@2ac2e7c
04-19 17:24:07.085 6235-6235/com.seven.www.testserviceserver:MyService D/Seven: onDestroy com.seven.www.testserviceserver.MyService@2ac2e7c
```

可以看出，结果和同进程绑定是一致的。结论已经在上面给出，这里就不重复了。
不同APP的跨进程服务绑定也是如此。

设置服务允许外部调用：
```java
        <service android:name=".MyService"
            android:exported="true"
            android:process=":MyService">
            <intent-filter>
                <action android:name="com.seven.www.testserviceserver.myservice"/>
            </intent-filter>
        </service>
```

外部APP "com.seven.www.testservice" 代码：
```java
    private static final String ACTION_MYSERVICE = "com.seven.www.testserviceserver.myservice";
    private ComponentName mServiceComponent = new ComponentName("com.seven.www.testserviceserver",
        "com.seven.www.testserviceserver.MyService");
        
    public void do_startService(View view) {
        Log.d(TAG, "do_startService");
        Intent service = new Intent(ACTION_MYSERVICE);
        service.setComponent(mServiceComponent);
        startService(service);

    }

    public void do_stopService(View view) {
        Log.d(TAG, "do_stopService");
        Intent service = new Intent(ACTION_MYSERVICE);
        service.setComponent(mServiceComponent);
        stopService(service);
    }
        
    ArrayList<ServiceConnection> scns = new ArrayList<>();

    public void do_bindService(View view) {
        Log.d(TAG, "do_bindService");
        Intent service = new Intent(ACTION_MYSERVICE);
        service.setComponent(mServiceComponent);
        ServiceConnection scn = new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                Log.d(TAG, "onServiceConnected -> " + name + "  " + service);
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {
                Log.d(TAG, "onServiceDisconnected -> " + name);
            }
        };

        if (bindService(service, scn, Context.BIND_AUTO_CREATE)) {
            scns.add(scn);
        }
        ;
    }

    public void do_unbindService(View view) {
        Log.d(TAG, "do_unbindService");
        if (scns.size() > 0) {
            unbindService(scns.remove(scns.size() - 1));
        }
    }
```

外部APP可以通过指定服务组件，进行相应的startService和bindService操作，这种情况和同一个APP多个进程是类似的。

如果服务不想被外部APP启动，那么设置
```java
android:exported="false"
```

不过这是一棒子打死所有人的做法，有时候我们需要允许自家的其他APP绑定自己的服务，那么可以通过设置权限解决：

在com.seven.www.testserviceserver添加一个自定义权限，设置只有相同签名的其他APP才可以访问。
```java
    <permission
        android:name="com.seven.www.testserviceserver.mypermission"
        android:description="@string/app_name"
        android:permissionGroup="com.seven.www.testserviceserver.group"
        android:protectionLevel="signature" />
```

组件相应地加上权限
```java
        <service android:name=".MyService"
            android:exported="true"
            android:permission="com.seven.www.testserviceserver.mypermission"
            android:process=":MyService">
            <intent-filter>
                <action android:name="com.seven.www.testserviceserver.myservice"/>
            </intent-filter>
        </service>
```

在com.seven.www.testservice申请权限:
```java
<uses-permission android:name="com.seven.www.testserviceserver.mypermission" />
```

#### 3. 服务销毁与重建

下面说下正常情况下服务的销毁：
1. 如果服务是通过startService启动的，期间没有bindService，那么无论执行了多少次startService，一旦触发stopService，那么服务就会销毁。注意：如果服务在
onStartCommand里面启用了线程，那么就算服务销毁了，线程依然会保持执行。
2. 在 1 的情况下，在服务内部调用stopSelf()也相当于stopService的效果。
3. 在 1 的情况下，在服务内部调用stopSelf(startId);也会销毁服务，但是这个startId必须是最新的startId.
   还有一个带返回值的版本stopSelfResult(startId); 返回true表示可以成功销毁，返回false不可以。如果startId为最后一个，那么就会返回true。

其实1、2和3的区别是：1、2情况无论如何都会触发销毁；3情况只有在所有的start请求都被执行完成后才会销毁；假设用户同时调用了多次startService，那么只有
最后一次start请求才可以销毁服务。

##### onStartCommand的返回值：

`START_STICKY_COMPATIBILITY` 兼容到SDK=5的情况，目前可以不用考虑了。
`START_STICKY` 如果服务在onStartCommand执行完成后被杀掉了，那么系统稍后会重建这个服务；如果在被杀掉时有其他start请求没有执行，那么这些没被执行的start请求都会被继续传递给onStartCommand执行；如果被杀掉时没有start请求，那么传递给onStartCommand的intent将是一个null。

将服务的进程属性去掉，保证服务运行在主进程，打开com.seven.www.testserviceserver， 触发了一次do_startService，然后通过最近任务把APP杀掉后：
```java
04-20 16:17:05.345 32180-32180/? D/MyService: do_startService
04-20 16:17:05.367 32180-32180/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@951aa34
04-20 16:17:05.371 32180-32180/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@951aa34

// 在这期间通过最近任务杀掉了APP
04-20 16:17:25.959 810-2637/? W/ActivityManager: Scheduling restart of crashed service com.seven.www.testserviceserver/.MyService in 1000ms
04-20 16:17:26.960 810-903/? W/ActivityManager: Unable to launch app com.seven.www.testserviceserver/10110 for service Intent { cmp=com.seven.www.testserviceserver/.MyService }: process is bad
```

可以看出，ActivityManager会在1000ms内重启服务，但是由于进程已经被杀掉，所以服务重启失败；也就说，服务想要自动重建，必须保证服务重建时，所在的进程是活着的。
否则重建失败。

将服务设置进程属性，其他保持和上面不变：
```java
04-20 16:24:18.903 32417-32417/? D/MyService: do_startService
04-20 16:24:18.934 810-9326/? I/ActivityManager: Start proc 32472:com.seven.www.testserviceserver:MyService/u0a110 for service com.seven.www.testserviceserver/.MyService
04-20 16:24:19.241 32472-32472/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:24:19.243 32472-32472/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2ac2e7c

04-20 16:24:32.873 810-9326/? I/ActivityManager: Killing 32472:com.seven.www.testserviceserver:MyService/u0a110 (adj 500): remove task
04-20 16:24:32.938 810-9326/? W/ActivityManager: Scheduling restart of crashed service com.seven.www.testserviceserver/.MyService in 1000ms
04-20 16:24:33.940 810-903/? W/ActivityManager: Unable to launch app com.seven.www.testserviceserver/10110 for service Intent { cmp=com.seven.www.testserviceserver/.MyService }: process is bad
```

可以看出，整个流程就多了创建进程和杀掉进程，其他都是一致的。

因为正常情况下，我们无法在1000ms内启动进程，那么就无法测试服务自动重建的情况，在这里我们可以利用系统的崩溃重新打开的提示进行测试，在onStartCommand里添加崩溃逻辑，保证服务运行在独立的进程中。
```java
    @Override
    public int onStartCommand(Intent intent, int flags, final int startId) {
        Log.d(TAG, "onStartCommand intent ->" + intent + " startId ->" + startId + " " + this);
        if (startId == 1) {
            throw new RuntimeException();
        }
        return START_STICKY;
    }
```

触发do_bindService， 在触发do_startService，将会出现崩溃：

```java
04-20 16:37:58.999 1363-1363/? D/MyService: do_bindService
04-20 16:37:59.021 810-5302/? I/ActivityManager: Start proc 1428:com.seven.www.testserviceserver:MyService/u0a110 for service com.seven.www.testserviceserver/.MyService
04-20 16:37:59.344 1428-1428/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:37:59.345 1428-1428/? D/MyService: onBind com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:37:59.348 1363-1363/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@951aa34


04-20 16:38:46.372 1363-1363/? D/MyService: do_startService
04-20 16:38:46.377 1428-1428/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:38:46.388 1428-1428/? E/AndroidRuntime: FATAL EXCEPTION: main
                                                 Process: com.seven.www.testserviceserver:MyService, PID: 1428
```

这时候系统会弹出一个重新打开应用的对话框，点击“重新打开应用”，那么服务所在的进程会自动重建，等待一定时间后服务也会自动重建并继续传递上面没有完成的intent进来：

<img src="https://raw.githubusercontent.com/shiwen500/blogs/master/images/Sevices1.png" width="30%"/>

```java
04-20 16:44:01.741 810-2105/? I/ActivityManager: Killing 1710:com.seven.www.testserviceserver:MyService/u0a110 (adj 0): crash
04-20 16:44:01.742 810-2105/? W/ActivityManager: Scheduling restart of crashed service com.seven.www.testserviceserver/.MyService in 9986ms
04-20 16:44:01.810 1641-1641/? D/MyService: onServiceDisconnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}

04-20 16:44:11.751 810-903/? I/ActivityManager: Start proc 1756:com.seven.www.testserviceserver:MyService/u0a110 for service com.seven.www.testserviceserver/.MyService
04-20 16:44:12.077 1756-1756/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:44:12.079 1756-1756/? D/MyService: onBind com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:44:12.081 1641-1641/? D/MyService: onServiceConnected -> ComponentInfo{com.seven.www.testserviceserver/com.seven.www.testserviceserver.MyService}  android.os.BinderProxy@223aea3
04-20 16:44:12.082 1756-1756/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 16:44:12.089 1756-1756/? E/AndroidRuntime: FATAL EXCEPTION: main
                                                 Process: com.seven.www.testserviceserver:MyService, PID: 1756
                                                 java.lang.RuntimeException: Unable to start service 
```

可以看出，当点击“重新打开应用时”，onServiceDisconnected被触发，并且ActivityManager将在9986ms后重启服务。重启服务时，会伴随着进程的创建（这个和从最近任务杀掉应用是不一样的，最近任务杀掉无法自动重建进程）。然后重新绑定（这里可以暂时忽视），接着传递之前没执行完成intent和相同的startId，最后依然是奔溃。

`START_NOT_STICKY` 如果服务在onStartCommand执行完成后被杀掉了，系统也不会重建服务。

`START_REDELIVER_INTENT` 服务每次自动重建，都会传递已经执行过的所有Intent以及没执行的Intent。将服务onStartCommand代码修改如下：
```java
    @Override
    public int onStartCommand(Intent intent, int flags, final int startId) {
        Log.d(TAG, "onStartCommand intent ->" + intent + " startId ->" + startId + " " + this);

        return START_REDELIVER_INTENT;
    }
```

触发3次do_startService， 最近任务杀掉应用，接着重新打开应用，再触发一次do_startService：
```java
04-20 17:16:45.572 2796-2796/? D/MyService: do_startService
04-20 17:16:45.578 2834-2834/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2e9bf5a
04-20 17:16:45.581 2834-2834/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2e9bf5a
04-20 17:16:46.208 2796-2796/? D/MyService: do_startService
04-20 17:16:46.214 2834-2834/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->2 com.seven.www.testserviceserver.MyService@2e9bf5a
04-20 17:16:47.891 2796-2796/? D/MyService: do_startService
04-20 17:16:47.895 2834-2834/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->3 com.seven.www.testserviceserver.MyService@2e9bf5a

04-20 17:16:54.750 810-5302/? I/ActivityManager: Killing 2834:com.seven.www.testserviceserver:MyService/u0a110 (adj 500): remove task
04-20 17:16:54.828 810-5302/? W/ActivityManager: Scheduling restart of crashed service com.seven.www.testserviceserver/.MyService in 18506ms

// 重新打开应用，触发do_startService
04-20 17:17:04.264 3610-3610/? D/MyService: do_startService
04-20 17:17:04.285 810-850/? I/ActivityManager: Start proc 3672:com.seven.www.testserviceserver:MyService/u0a110 for service com.seven.www.testserviceserver/.MyService
04-20 17:17:04.594 3672-3672/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 17:17:04.596 3672-3672/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 17:17:04.599 3672-3672/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->2 com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 17:17:04.600 3672-3672/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->3 com.seven.www.testserviceserver.MyService@2ac2e7c
04-20 17:17:04.603 3672-3672/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->5 com.seven.www.testserviceserver.MyService@2ac2e7c
```

服务运行在主进程中也是一样的：
```java
04-20 17:19:14.726 3848-3848/? D/MyService: do_startService
04-20 17:19:14.747 3848-3848/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@951aa34
04-20 17:19:14.749 3848-3848/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@951aa34
04-20 17:19:16.080 3848-3848/? D/MyService: do_startService
04-20 17:19:16.096 3848-3848/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->2 com.seven.www.testserviceserver.MyService@951aa34
04-20 17:19:16.694 3848-3848/? D/MyService: do_startService
04-20 17:19:16.709 3848-3848/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->3 com.seven.www.testserviceserver.MyService@951aa34
04-20 17:19:19.842 810-2621/? W/ActivityManager: Scheduling restart of crashed service com.seven.www.testserviceserver/.MyService in 10218ms

// 重新打开应用
04-20 17:19:25.198 3947-3947/? D/MyService: onCreate com.seven.www.testserviceserver.MyService@2f59bb1
04-20 17:19:25.200 3947-3947/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->1 com.seven.www.testserviceserver.MyService@2f59bb1
04-20 17:19:25.202 3947-3947/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->2 com.seven.www.testserviceserver.MyService@2f59bb1
04-20 17:19:25.204 3947-3947/? D/MyService: onStartCommand intent ->Intent { cmp=com.seven.www.testserviceserver/.MyService } startId ->3 com.seven.www.testserviceserver.MyService@2f59bb1
```

总结：服务重建自动需要依赖进程存活，只有在进程意外退出（奔溃、内存不足回收）后，服务自动重建才可以伴随着创建对应的进程，用户手动结束进程的情况服务自动重建不能重建进程。

##### 绑定服务重建

下面说的都是属于 服务运行在独立进程的情况。

从上面可以知道，所绑定的远程服务一旦死亡，那么onServiceDisconnected会触发，并且等到远程服务`自动`重建时，会自动重新绑上该服务。注意 ，服务被杀死后，有可能很快就会重启，而在服务重启时，如果进程没有处于活跃状态，那么服务重启就会失败, 那么也不会重新绑定了。

注意：只有当服务死亡时，ActivityManager在一段时间后自动调度Service restart，在这段时间内服务的绑定状态是保留的，所以在这段时间内服务重启成功的话，之前的绑定都会重新跑一次。服务重启失败，或者超出了这段时间没有把服务所在进程启动起来，那么服务的状态就被清除了。

也就是说，我们应该在onServiceDisconnected主动调用unbindService(conn), 确保服务自动重启不会自动重新绑定，因为这个机制是不可靠的。

不想自动重新绑定上可以这样做：
```java
            @Override
            public void onServiceDisconnected(ComponentName name) {
                Log.d(TAG, "onServiceDisconnected -> " + name);
                unbindService(this);
            }
```

实现服务死亡主动重连：

1. 在onServiceDisconnected 时主动 调用 unbindService(this)解绑，然后重新绑定。

2. 使用Binder.linkToDeath实现同样的效果。 因为如果客户端绑定服务端后，给服务端传递了一个Binder，那么服务端就可以这样监听。

注意，无法通过这种办法启动一个处于关闭APP，但是如果服务所在的APP是处于打开状态，那么即使服务声明在独立子进程里面，其他APP也可以启动这个服务。















