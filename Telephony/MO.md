<h2 align = "center">拨号流程</h2>

----

概述：拨号的过程，如果按照进程来分，可以分成3个模块。

1. com.android.dialer进程，即电话APP模块

2. system进程，系统框架层

3. com.android.phone进程，Phone服务模块

拨号的过程，从APP开始，发送消息到system进程，system和Phone服务交互，Phone通过Socket和底层交互，然后将交互结果转交给system， system再告诉电话APP，APP及时将打电话的状态展示。

### com.android.dialer 进程

#### DialtactsActivity 拨号主界面

DialtactsActivity 是APP的主界面，里面包含一个 DialpadFragment 作为拨号键盘，当点击拨号按钮时触发 handleDialButtonPressed

```java
private void handleDialButtonPressed() {
        if (isDigitsEmpty()) { // No number entered.
            //号码空白自动补全
            handleDialButtonClickWithEmptyDigits();
        } else {

            if (number != null&& !TextUtils.isEmpty(mProhibitedPhoneNumberRegexp)
                    && number.matches(mProhibitedPhoneNumberRegexp))
                //处理一些不能拨打电话的情况
            } else {
                //...
                //核心，主流程
                DialerUtils.startActivityWithErrorToast(getActivity(), intent);
                //...
            }
        }
    }
```

DialerUtils.startActivityWithErrorToast 内部调用 startActivityWithErrorToast

```java
public static void startActivityWithErrorToast(Context context, Intent intent, int msgId) {
    try {
        if ((IntentUtil.CALL_ACTION.equals(intent.getAction())
                        && context instanceof Activity)) {
            // All dialer-initiated calls should pass the touch point to the InCallUI
            Point touchPoint = TouchPointManager.getInstance().getPoint();
            if (touchPoint.x != 0 || touchPoint.y != 0) {
                Bundle extras = new Bundle();
                extras.putParcelable(TouchPointManager.TOUCH_POINT, touchPoint);
                intent.putExtra(TelecomManager.EXTRA_OUTGOING_CALL_EXTRAS, extras);
            }
            // Telecom接口
            final TelecomManager tm =
                    (TelecomManager) context.getSystemService(Context.TELECOM_SERVICE);
            // 核心，主过程
            tm.placeCall(intent.getData(), intent.getExtras());
        } else {
            context.startActivity(intent);
        }
    } catch (ActivityNotFoundException e) {
        Toast.makeText(context, msgId, Toast.LENGTH_SHORT).show();
    }
}
```

TelecomManager.placeCall主要获取TelecomService的Binder接口，跨进程进入到TelecomService（`system进程`）内部，至此 com.android.dialer 进程的工作暂时完成

### system 进程

#### TelecomService 访问通话模块的接口

TelecomService 是属于系统框架向外提供的服务，随着系统启动而启动，并由TelecomManager向外提供访问接口。在TelecomService 第一次被绑定时（系统初始化的时候就会进行绑定，并不是在打电话的时候才绑定），TelecomSystem 会被初始化，期间构建一些列必须资源，包括Phone和CI（和底层沟通的对象）的构建。

```java
static void initializeTelecomSystem(Context context) {
        // TelecomSystem被初始化
        if (TelecomSystem.getInstance() == null) {
            TelecomSystem.setInstance(
                    new TelecomSystem(
                           //...
                           ));
        }
        if (BluetoothAdapter.getDefaultAdapter() != null) {
            context.startService(new Intent(context, BluetoothPhoneService.class));
        }
    }
```

#### TelecomSystem 系统进程中，通话模块的核心

TelecomSystem 构造函数

```java
public TelecomSystem(
        Context context,
        MissedCallNotifier missedCallNotifier,
        CallerInfoAsyncQueryFactory callerInfoAsyncQueryFactory,
        HeadsetMediaButtonFactory headsetMediaButtonFactory,
        ProximitySensorManagerFactory proximitySensorManagerFactory,
        InCallWakeLockControllerFactory inCallWakeLockControllerFactory) {
    mContext = context.getApplicationContext();

    // 构建CallsManager
    mCallsManager = new CallsManager(
            mContext,
            mLock,
            mContactsAsyncHelper,
            callerInfoAsyncQueryFactory,
            mMissedCallNotifier,
            mPhoneAccountRegistrar,
            headsetMediaButtonFactory,
            proximitySensorManagerFactory,
            inCallWakeLockControllerFactory);

    mRespondViaSmsManager = new RespondViaSmsManager(mCallsManager, mLock);
    mCallsManager.setRespondViaSmsManager(mRespondViaSmsManager);

    mContext.registerReceiver(mUserSwitchedReceiver, USER_SWITCHED_FILTER);
    mBluetoothPhoneServiceImpl = new BluetoothPhoneServiceImpl(
            mContext, mLock, mCallsManager, mPhoneAccountRegistrar);
    // 构建CallIntentProcessor， 执行Intent的请求
    mCallIntentProcessor = new CallIntentProcessor(mContext, mCallsManager);
    mTelecomBroadcastIntentProcessor = new TelecomBroadcastIntentProcessor(
            mContext, mCallsManager);

    // 构建TelecomServiceImpl,这个是TelecomSystem的真正实现，返回的binder由这里提供，外部对TelecomService的操作实际上都是在TelecomServiceImpl里面进行的。
    mTelecomServiceImpl = new TelecomServiceImpl(
            mContext, mCallsManager, mPhoneAccountRegistrar, mLock);
}
```

`note`:上面的构造都是在打电话前执行的。下面回到拨号流程

上面placeCall会进入到 TelecomServiceImpl 内部的 mBinderImpl 的 placeCall

```java
public void placeCall(Uri handle, Bundle extras, String callingPackage) {
            // ...
            synchronized (mLock) {
                final UserHandle userHandle = Binder.getCallingUserHandle();
                long token = Binder.clearCallingIdentity();
                try {
                    final Intent intent = new Intent(Intent.ACTION_CALL, handle);
                    intent.putExtras(extras);
                    // 核心，主过程
                    new UserCallIntentProcessor(mContext, userHandle).processIntent(intent,
                            callingPackage, hasCallAppOp && hasCallPermission);
                } finally {
                    Binder.restoreCallingIdentity(token);
                }
            }
        }
```

接着是 UserCallIntentProcessor.processIntent -> UserCallIntentProcessor.processOutgoingCallIntent

```java
private void processOutgoingCallIntent(Intent intent, String callingPackageName,
            boolean canCallNonEmergency) {
        Uri handle = intent.getData();
        String scheme = handle.getScheme();
        String uriString = handle.getSchemeSpecificPart();

        // ...

        intent.putExtra(CallIntentProcessor.KEY_IS_PRIVILEGED_DIALER,
                isDefaultOrSystemDialer(callingPackageName));
        // 核心，主过程
        sendBroadcastToReceiver(intent);
    }
```

```java
private boolean sendBroadcastToReceiver(Intent intent) {
    intent.putExtra(CallIntentProcessor.KEY_IS_INCOMING_CALL, false);
    intent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
    // 最后进入到 PrimaryCallReceiver
    intent.setClass(mContext, PrimaryCallReceiver.class);
    Log.d(this, "Sending broadcast as user to CallReceiver");
    mContext.sendBroadcastAsUser(intent, UserHandle.OWNER);
    return true;
}
```

PrimaryCallReceiver 作为拨号的单一入口, 其收到消息后，就转发给 TelecomSystem的 CallIntentProcessor 进行处理

```java
public void onReceive(Context context, Intent intent) {
        synchronized (getTelecomSystem().getLock()) {
            getTelecomSystem().getCallIntentProcessor().processIntent(intent);
        }
    }
```

#### CallsManager 创建Call, 链接，监听Call状态

processIntent 直接转给 processOutgoingCallIntent

```java
static void processOutgoingCallIntent(
            Context context,
            CallsManager callsManager,
            Intent intent) {

        // ....
        // 创建一个call
        Call call = callsManager.startOutgoingCall(handle, phoneAccountHandle, clientExtras);

        if (call != null) {
            // ...
            // 拿到上面创建的call, 进行下一步
            NewOutgoingCallIntentBroadcaster broadcaster = new NewOutgoingCallIntentBroadcaster(
                    context, callsManager, call, intent, isPrivilegedDialer);
            final int result = broadcaster.processIntent();
            // ...
        }
    }
```

> 这是一个断点，因为下面将会详细地深入分析 callsManager.startOutgoingCall 的流程，下面分析 startOutgoingCall 完成后， 再从 NewOutgoingCallIntentBroadcaster.processIntent()往下分析

startOutgoingCall 新建一个（或者重用） call， 将CallsManager绑定监听该 call， 然后通知 callsManager 的监听者， 添加了一个 call

```java
Call startOutgoingCall(Uri handle, PhoneAccountHandle phoneAccountHandle, Bundle extras) {
        Call call = getNewOutgoingCall(handle);

        // 如果是MMI 号码
        if ((isPotentialMMICode(handle) || isPotentialInCallMMICode) && !needsAccountSelection) {
            // 让CallsManager监听call的行为
            call.addListener(this);

        } else if (!mCalls.contains(call)) {
            // 确保Call不会重复添加（getNewOutgoingCall有重用机制）

            // 添加call, 然后callsManager通知监听者，添加了一个call
            addCall(call);
        }

        return call;
    }
```

addCall 添加call, 然后callsManager通知监听者，添加了一个call

```java
private void addCall(Call call) {

    // CallsManager 监听 call
    call.addListener(this);

    // 添加
    mCalls.add(call);

    // 告诉所有callsManager的监听者
    for (CallsManagerListener listener : mListeners) {

        listener.onCallAdded(call);

    }
}
```
接下来查看 CallsManager 到底通知了谁，需要查看 CallsManager 构造过程

CallsManager 的构造函数

```java
CallsManager(
        Context context,
        TelecomSystem.SyncRoot lock,
        ContactsAsyncHelper contactsAsyncHelper,
        CallerInfoAsyncQueryFactory callerInfoAsyncQueryFactory,
        MissedCallNotifier missedCallNotifier,
        PhoneAccountRegistrar phoneAccountRegistrar,
        HeadsetMediaButtonFactory headsetMediaButtonFactory,
        ProximitySensorManagerFactory proximitySensorManagerFactory,
        InCallWakeLockControllerFactory inCallWakeLockControllerFactory) {
    // ....
    // 在CallsManager构造的时候， 会给CallsManager添加一系列监听者， 当CallsManager变化时，会通知他们
    mListeners.add(statusBarNotifier);
    mListeners.add(mCallLogManager);
    mListeners.add(mPhoneStateBroadcaster);
    // 重点关注 InCallController
    mListeners.add(mInCallController);
    mListeners.add(mRinger);
    mListeners.add(new RingbackPlayer(this, playerFactory));
    mListeners.add(new InCallToneMonitor(playerFactory, this));
    mListeners.add(mCallAudioManager);
    mListeners.add(missedCallNotifier);
    mListeners.add(mDtmfLocalTonePlayer);
    mListeners.add(mHeadsetMediaButton);
    mListeners.add(mProximitySensorManager);

    // ...
}
```

目前需要关注的是 InCallController，因为 InCallController 监听CallsManager, 收到来自于CallsManager的消息后， 跨进程回到 最初的 com.android.dialer 进程， 通知 UI 发生变化，也就是说 InCallController 是system 进程 主动沟通 com.android.dialer 进程的桥梁，下面详细解释，InCallController时如何沟通 com.android.dialer进程的。

#### InCallController 控制电话App的逻辑和UI

InCallController 的构造

```java
public InCallController(
        Context context, TelecomSystem.SyncRoot lock, CallsManager callsManager) {
    mContext = context;
    mLock = lock;
    mCallsManager = callsManager;
    Resources resources = mContext.getResources();

    // 这是系统预先配置好的一个组件服务
    // packages/services/Telecomm/res/values/config.xml
    // <!-- Package name for the default in-call UI and dialer [DO NOT TRANSLATE] -->
    // <string name="ui_default_package" translatable="false">com.android.dialer</string>

    // <!-- Class name for the default in-call UI Service [DO NOT TRANSLATE] -->
    // <string name="incall_default_class" translatable="false">com.android.incallui.InCallServiceImpl</string>
    // 这个服务组件时用来控制电话APP的UI的
    mSystemInCallComponentName = new ComponentName(
            resources.getString(R.string.ui_default_package),
            resources.getString(R.string.incall_default_class));
}
```

由上面得知， CallsManager 的 addCall 会触发 InCallController 的 onCallAdded

```java
public void onCallAdded(Call call) {
    if (!isBoundToServices()) {
        // 第一次触发时，服务没有绑定，所以需要先绑定服务
        bindToServices(call);
    } else {
        // ...
    }
}
```

```java
private void bindToServices(Call call) {
        PackageManager packageManager = mContext.getPackageManager();
        Intent serviceIntent = new Intent(InCallService.SERVICE_INTERFACE);

        // InCall逻辑控制组件
        List<ComponentName> inCallControlServices = new ArrayList<>();
        // InCall UI 控制组件
        ComponentName inCallUIService = null;

        // 查找逻辑组件 和UI组件， 用于绑定，绑定后，InCallController 可以发送消息给这些组件
        for (ResolveInfo entry :
                packageManager.queryIntentServices(serviceIntent, PackageManager.GET_META_DATA)) {
            ServiceInfo serviceInfo = entry.serviceInfo;
            if (serviceInfo != null) {
                // ...
                if (isUIService) {
                    // For the main UI service, we always prefer the default dialer.
                    if (isDefaultDialerPackage) {

                        // 查找到UI 组件
                        inCallUIService = componentName;
                        Log.i(this, "Found default-dialer's In-Call UI: %s", componentName);
                    }
                } else {
                    // 查找到控制逻辑组件
                    inCallControlServices.add(componentName);
                }

            }
        }

        // 先尝试去绑定 UI 组件
        if (inCallUIService != null) {
            // skip default dialer if we have an emergency call or if it failed binding.
            if (mCallsManager.hasEmergencyCall()) {
                Log.i(this, "Skipping default-dialer because of emergency call");
                inCallUIService = null;
            } else if (!bindToInCallService(inCallUIService, call, "def-dialer")) {
                Log.event(call, Log.Events.ERROR_LOG,
                        "InCallService UI failed binding: " + inCallUIService);
                inCallUIService = null;
            }
        }

        // 由于我们默认组件，所以必定为null,
        // 最后绑定的是 com.android.incallui.InCallServiceImpl （在config.xml里面配置的）
        if (inCallUIService == null) {
            // 如果 default-dialer 组件不存在，
            // 那么绑定系统默认的UI组件
            // 核心， 主流程
            inCallUIService = mSystemInCallComponentName;
            if (!bindToInCallService(inCallUIService, call, "system")) {
                Log.event(call, Log.Events.ERROR_LOG,
                        "InCallService system UI failed binding: " + inCallUIService);
            }
        }
        mInCallUIComponentName = inCallUIService;

        // 绑定逻辑控制组件 （暂时不管）
        for (ComponentName componentName : inCallControlServices) {
            bindToInCallService(componentName, call, "control");
        }
    }
```

bindToInCallService 确保不会重复绑定同一个Service。

```java
private boolean bindToInCallService(ComponentName componentName, Call call, String tag) {
    // ...
    // 绑定成功，触发 InCallServiceConnection 的 onServiceConnected
    InCallServiceConnection inCallServiceConnection = new InCallServiceConnection();
    if (mContext.bindServiceAsUser(intent, inCallServiceConnection,
                Context.BIND_AUTO_CREATE | Context.BIND_FOREGROUND_SERVICE,
                UserHandle.CURRENT)) {
        // 绑定成功后， 保存链接
        mServiceConnections.put(componentName, inCallServiceConnection);
        /// M: Register voice recording listener @{
        PhoneRecorderHandler.getInstance().setListener(mRecorderListener);
        /// @}
        return true;
    }
    return false;
}
```

InCallServiceConnection.onServiceConnected  =>  onConnected(name, service)

```java
private void onConnected(ComponentName componentName, IBinder service) {

        // 拿到远程访问接口，并保存
        IInCallService inCallService = IInCallService.Stub.asInterface(service);
        mInCallServices.put(componentName, inCallService);

        try {
            // 让绑定的服务，可以通过 InCallAdapter 跨进程 访问 system 进程 （因为当前就是在系统进程）
            // InCallAdapter 是一个 Binder。
            // 需要对Android 通过 Binder实现跨进程调用有一定了解
            // 还有对Android 的AIDL由一定了解
            inCallService.setInCallAdapter(
                    new InCallAdapter(
                            mCallsManager,
                            mCallIdMapper,
                            mLock));
        } catch (RemoteException e) {
            Log.e(this, e, "Failed to set the in-call adapter.");
            Trace.endSection();
            onInCallServiceFailure(componentName, "setInCallAdapter");
            return;
        }


        Collection<Call> calls = mCallsManager.getCalls();
        // mCalls 不为空，（由上面可得）
        if (!calls.isEmpty()) {
            Log.i(this, "Adding %s calls to InCallService after onConnected: %s", calls.size(),
                    componentName);
            for (Call call : calls) {
                try {
                    // addCall 致使 当前的 InCallController 可以 监听该Call 的变化
                    addCall(call);
                    // 通过远程调用， 添加一个Call到 InCallServiceImpl(对，这次绑定的就是它)
                    inCallService.addCall(toParcelableCall(call, true /* includeVideoProvider */));
                } catch (RemoteException ignored) {
                }
            }
            onCallAudioStateChanged(
                    null,
                    mCallsManager.getAudioState());
            onCanAddCallChanged(mCallsManager.canAddCall());
        } else {
            unbindFromServices();
        }
        Trace.endSection();
    }
```

onConnected 拿到远程调用接口后，连续执行了2次远程接口，下面会回到 com.android.dialer 进程进行分析。现在分析过程跑得有点远了（如果忘记得差不多了，建议回去再读一遍）

### com.android.dialer 进程

#### InCallServiceImpl InCallService 的实现，负责和 Telecom 沟通，更新UI

InCallServiceImpl 的核心功能 是在它的父类 InCallService， InCallServiceImpl 被绑定时，返回一个 InCallServiceBinder, 因此 system 进程拿到的接口就是这个, 绑定之前，会先初始化 InCallPresenter, InCallPresenter 是全局唯一（单例）的， 它负责 更新 电话APP的 UI。（MVP 模式）

```java
public IBinder onBind(Intent intent) {
    final Context context = getApplicationContext();
    final ContactInfoCache contactInfoCache = ContactInfoCache.getInstance(context);
    // 初始化 InCallPresenter
    InCallPresenter.getInstance().setUp(
            getApplicationContext(),
            CallList.getInstance(),
            AudioModeProvider.getInstance(),
            new StatusBarNotifier(context, contactInfoCache),
            contactInfoCache,
            new ProximitySensor(
                    context,
                    AudioModeProvider.getInstance(),
                    new AccelerometerListener(context))
            );
    InCallPresenter.getInstance().onServiceBind();
    // 核心， 主过程
    InCallPresenter.getInstance().maybeStartRevealAnimation(intent);
    TelecomAdapter.getInstance().setInCallService(this);

    // 返回一个 InCallServiceBinder
    return super.onBind(intent);
}
```

#### InCallPresenter MVP模式中， 控制逻辑和UI

InCallPresenter 的 maybeStartRevealAnimation 会启动 InCallActivity， 即 拨号进行中的界面（或者是来电界面）

```java
public void maybeStartRevealAnimation(Intent intent) {
    //...

    final Intent incallIntent = getInCallIntent(false, true);
    incallIntent.putExtra(TouchPointManager.TOUCH_POINT, touchPoint);
    mContext.startActivity(incallIntent);
}
```

getInCallIntent 就是创建一个启动 InCallActivity 的 Intent.

```java
public Intent getInCallIntent(boolean showDialpad, boolean newOutgoingCall) {
    final Intent intent = new Intent(Intent.ACTION_MAIN, null);
    intent.setFlags(Intent.FLAG_ACTIVITY_NO_USER_ACTION | Intent.FLAG_ACTIVITY_NEW_TASK);

    // 显式指定启动 InCallActivity
    intent.setClass(mContext, InCallActivity.class);
    if (showDialpad) {
        intent.putExtra(InCallActivity.SHOW_DIALPAD_EXTRA, true);
    }
    // 去点标志 true
    intent.putExtra(InCallActivity.NEW_OUTGOING_CALL_EXTRA, newOutgoingCall);
    return intent;
}
```

至此， 拨号进行中的界面起来了， 接下来查看 InCallServiceBinder 里面的实现，setInCallAdapter 和 addCall。

InCallServiceBinder 收到消息，是直接转交给 mHandler 的

#### Phone 在UI层面上， Phone是唯一的，不分类型，而且数量只有一个

```java
public void handleMessage(Message msg) {
            // ...
            switch (msg.what) {
                case MSG_SET_IN_CALL_ADAPTER:
                    // setInCallAdapter 触发创建 com.android.dialer 的唯一 的phone对象
                    // 也就是说， 在 com.android.dialer 进程的所有操作 都是经过 phone 里面的mInCallAdapter 发送到 system 进程的 （框架层）
                    mPhone = new Phone(new InCallAdapter((IInCallAdapter) msg.obj));

                    // 核心， 让InCallService 监听 Phone 的状态变化
                    mPhone.addListener(mPhoneListener);
                    onPhoneCreated(mPhone);
                    break;
                case MSG_ADD_CALL:
                    mPhone.internalAddCall((ParcelableCall) msg.obj);
                    break;
                case MSG_UPDATE_CALL:
                    mPhone.internalUpdateCall((ParcelableCall) msg.obj);
                    break;
                // ...
                // 省略大量消息类型
            }
        }
```

mPhone.internalAddCall 创建一个本地的Telecom Call (不同于system进程的Call, 下面简称TCall)，通知 InCallService

```java
final void internalAddCall(ParcelableCall parcelableCall) {
    Call call = new Call(this, parcelableCall.getId(), mInCallAdapter,
            parcelableCall.getState());
    mCallByTelecomCallId.put(parcelableCall.getId(), call);
    mCalls.add(call);
    checkCallTree(parcelableCall);
    call.internalUpdate(parcelableCall, mCallByTelecomCallId);
    // 核心， 通知 InCallService 有添加了一个Call
    //private void fireCallAdded(Call call) {
    //    for (Listener listener : mListeners) {
    //        listener.onCallAdded(this, call);
    //    }
    //}
    fireCallAdded(call);
 }
```

mPhoneListener.onCallAdded  =>  InCallServiceImpl.onCallAdded

```java
public void onCallAdded(Call call) {
        // 核心， 主过程
        CallList.getInstance().onCallAdded(call);
        // 绑定监听 TCall 的状态变化
        InCallPresenter.getInstance().onCallAdded(call);
    }
```

#### CallList 维护电话列表

CallList维护着一个打电话列表 (CallList里面的Call简称为 LCall, 区别于上面的 TCall), 每一个 LCall 维护着一个 TCall.

* TCall 是由 Phone 维护的
* LCall 是由 CallList 维护的
* TCall 和 LCall 一一对应

CallList.onCallAdded

```java
public void onCallAdded(android.telecom.Call telecommCall) {
        // 构造LCall，LCall 监听 TCall
        Call call = new Call(telecommCall);
        if (call.getState() == Call.State.INCOMING ||
                call.getState() == Call.State.CALL_WAITING) {
            onIncoming(call, call.getCannedSmsResponses());
        } else {
            // 因为是去电，所以是update
            onUpdate(call);
        }
    }
```

LCall 构造函数

```java
public Call(android.telecom.Call telecommCall) {
    // LCall 维护 TCall
    mTelecommCall = telecommCall;
    mId = ID_PREFIX + Integer.toString(sIdCounter++);

    updateFromTelecommCall();
    // 查看 mTelecomCallCallback 可得， TCall 变化时触发 LCall.update()
    //  LCall.update() 触发  CallList.onUpdate(LCall)
    // CallList.onUpdate(LCall) 触发 CallList.notifyGenericListeners()
    // private void notifyGenericListeners() {
    //   for (Listener listener : mListeners) {
    //     listener.onCallListChange(this);
    //   }
    // }
    mTelecommCall.registerCallback(mTelecomCallCallback);
}
```

也就是说，只要 一个 TCall 变化， 都会告诉 监听 CallList 的所有监听者， 下面查找谁监听了 CallList

InCallPresenter 实现了 CallList.Listener 接口， 所以 InCallPresenter.onCallListChange 会被触发

```java
public void onCallListChange(CallList callList) {
        // ...
        // 根据新状态，决定是否打开 InCallActivity， 或则关闭 InCallActivity, 确保不会重复打开
        newState = startOrFinishUi(newState);

        for (InCallStateListener listener : mListeners) {

            // 凡是监听 InCallPresenter 的监听者，都得到通知。
            listener.onStateChange(oldState, mInCallState, callList);
        }

    }
```

#### InCallActivity 展示拨号，或者来电状态

InCallActivity 是通过 展示 或隐藏 Fragment 来 区分不同状态的， 拨号进行中的Fragment 是 CallCardFragment, 而 CallCardFragment 对应的 Presenter 是 CallCardPresenter, onUiReady 如下

```java
public void onUiReady(CallCardUi ui) {
    super.onUiReady(ui);

    // Contact search may have completed before ui is ready.
    if (mPrimaryContactInfo != null) {
        updatePrimaryDisplayInfo();
    }

    // Register for call state changes last
    InCallPresenter.getInstance().addListener(this);
    InCallPresenter.getInstance().addIncomingCallListener(this);
    InCallPresenter.getInstance().addDetailsListener(this);
    InCallPresenter.getInstance().addInCallEventListener(this);
}
```

所以 CallCardPresenter.onStateChange 会被触发 接着就是更新UI了（MVP）

#### 总结一下： system 进程的 InCallController 收到来自于 CallsManager 的消息后， 发送到 com.android.dialer 进程， 一步步地更新 拨号界面的状态， 整个过程已经全部打通了。

### system 进程

从上面设的断点开始 继续分析 NewOutgoingCallIntentBroadcaster.processIntent() 的过程

```java
int processIntent() {
        // ...

        if (callImmediately) {
            // ...
            // 回到 CallsManager
            mCallsManager.placeOutgoingCall(mCall, Uri.fromParts(scheme, number, null), null,
                    speakerphoneOn, videoState);

        }
        broadcastIntent(intent, number, !callImmediately);
        return DisconnectCause.NOT_DISCONNECTED;
    }
```

CallsManager.placeOutgoingCall

```java
void placeOutgoingCall(Call call, Uri handle, GatewayInfo gatewayInfo, boolean speakerphoneOn,
        int videoState) {
    // ...
    // 省略大量状态判断

    if (call.getTargetPhoneAccount() != null || isEmergencyCall) {
        // If the account has been set, proceed to place the outgoing call.
        // Otherwise the connection will be initiated when the account is set by the user.
        call.startCreateConnection(mPhoneAccountRegistrar);
    }
}
```

#### Call 由 CallsManager 维护的拨号，或者来电实例

Call.startCreateConnection

```java
void startCreateConnection(PhoneAccountRegistrar phoneAccountRegistrar) {
        mCreateConnectionProcessor = new CreateConnectionProcessor(this, mRepository, this,
                phoneAccountRegistrar, mContext);
        mCreateConnectionProcessor.process();
    }
```

CreateConnectionProcessor.process  ==>  CreateConnectionProcessor.attemptNextPhoneAccount

```java
private void attemptNextPhoneAccount() {
    // ...

    if (mResponse != null && attempt != null) {
        Log.i(this, "Trying attempt %s", attempt);
        PhoneAccountHandle phoneAccount = attempt.connectionManagerPhoneAccount;

        // 取到一个 ConnectionServiceWrapper （有缓存）
        ConnectionServiceWrapper service =
                mRepository.getService(
                        phoneAccount.getComponentName(),
                        phoneAccount.getUserHandle());
        if (service == null) {
            Log.i(this, "Found no connection service for attempt %s", attempt);
            attemptNextPhoneAccount();
        } else {
            mCall.setConnectionManagerPhoneAccount(attempt.connectionManagerPhoneAccount);
            /// M: Valid phone account for ECC may have been set.@{
            if (mCall.getTargetPhoneAccount() == null) {
                mCall.setTargetPhoneAccount(attempt.targetPhoneAccount);
            }
            /// @}
            mCall.setConnectionService(service);
            setTimeoutIfNeeded(service, attempt);

            // 开始进行链接，核心过程
            service.createConnection(mCall, new Response(service));
        }
    } else {
        // ...
        // 失败操作
    }
```

#### ConnectionServiceWrapper 负责和 Telephony 沟通

创建链接就是一个绑定服务的过程

```java
void createConnection(final Call call, final CreateConnectionResponse response) {

    // 链接创建成功后 onSuccess 会被调用
    BindCallback callback = new BindCallback() {
        @Override
        public void onSuccess() {

            try {
                /// M: For VoLTE @{
                boolean isConferenceDial = call.isConferenceDial();

                // 会议通话
                if (isConferenceDial) {
                    logOutgoing("createConference(%s) via %s.", call, getComponentName());
                    mServiceInterface.createConference(
                            call.getConnectionManagerPhoneAccount(),
                            callId,
                            new ConnectionRequest(
                                    call.getTargetPhoneAccount(),
                                    call.getHandle(),
                                    extras,
                                    call.getVideoState()),
                            call.getConferenceParticipants(),
                            call.isIncoming());

                // 非会议（拨打电话会走这里）
                } else {

                    // 通过远程接口创建链接
                    mServiceInterface.createConnection(
                            call.getConnectionManagerPhoneAccount(),
                            callId,
                            new ConnectionRequest(
                                    call.getTargetPhoneAccount(),
                                    call.getHandle(),
                                    extras,
                                    call.getVideoState()),
                            call.isIncoming(),
                            call.isUnknown());
                }
                /// @}
            } catch (RemoteException e) {

            }
        }

        @Override
        public void onFailure() {
        }
    };

    mBinder.bind(callback, call);
}
```

```java
void bind(BindCallback callback, Call call) {

            if (mServiceConnection == null) {
                Intent serviceIntent = new Intent(mServiceAction).setComponent(mComponentName);
                ServiceConnection connection = new ServiceBinderConnection(call);

                // 进行绑定
                if (mUserHandle != null) {
                    isBound = mContext.bindServiceAsUser(serviceIntent, connection, bindingFlags,
                            mUserHandle);
                } else {
                    isBound = mContext.bindService(serviceIntent, connection, bindingFlags);
                }
                if (!isBound) {
                    handleFailedConnection();
                    return;
                }
            } else {
            }
        }
```

绑定成功后，ServiceBinderConnection 的 onServiceConnected 会触发

```java
public void onServiceConnected(ComponentName componentName, IBinder binder) {
            synchronized (mLock) {

                // 设置 mServiceInterface
                // 会触发 ConnectionServiceWrapper.setServiceInterface  ==>  ConnectionServiceWrapper.addConnectionServiceAdapter 通过mServiceInterface，给绑定的服务，提供一个访问自己的接口
                setBinder(binder);

                // 触发bind(BindCallback callback, Call call) 中 callback 的 onSuccess
                handleSuccessfulConnection();
            }
        }
```

整个绑定过程，只做2件事，一是给远程服务提供访问自己的接口，二是利用远程接口创建一个通话链接。 这2件事都是跨进程进行的。远程服务访问自己的接口是 `ConnectionServiceWrapper.Adapter` , 是一个Binder。

`ConnectionServiceWrapper.Adapter` 提供了一套更新Call 状态的 操作， 因为目前分析的是拨打电话流程，所以先分析setDialing

```java
public void setDialing(String callId) {
            long token = Binder.clearCallingIdentity();
            try {
                synchronized (mLock) {
                    logIncoming("setDialing %s", callId);
                    if (mCallIdMapper.isValidCallId(callId)) {
                        Call call = mCallIdMapper.getCall(callId);
                        if (call != null) {
                            // 核心代码，查看
                            mCallsManager.markCallAsDialing(call);
                        } else {
                        }
                    }
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
```

CallsManager.markCallAsDialing  ==> CallsManager.setCallState

```java
private void setCallState(Call call, int newState, String tag) {
        // ...
        if (newState != oldState) {
            // ...
            if (mCalls.contains(call)) {
                for (CallsManagerListener listener : mListeners) {

                    // 因为 InCallController 是 CallsManager 的监听者， 所以， 这里会通知 InCallController 更新
                    listener.onCallStateChanged(call, oldState, newState);

                }
                updateCallsManagerState();
            }      
            handleActionProcessComplete(call);
        }
    }
```

#### system 进程小结： ConnectionServiceWrapper 绑定远程服务后， 远程服务跨进程 更新 CallsManager 的状态， CallsManager 通知 InCallController 更新， InCallController 通知 com.android.dialer 进程进行UI更新。

### com.android.phone 进程

com.android.phone 在开机的时候，就启动了，而且无法关闭。 它负责和底层通讯，然后将通讯结果提交给 system(框架层), 最后更新UI

#### TelephonyConnectionService com.android.phone 进程向外提供的服务接口

直接得知 ConnectionServiceWrapper 绑定的服务是 TelephonyConnectionService 。这个服务在 com.android.phone 进程的角色是负责和 system 进程通讯。 TelephonyConnectionService 的父类 ConnectionService 实现了主要的功能，同样，ConnectionService 返回的 Binder 直接将操作转交给 mHandler。

```java
private final Handler mHandler = new Handler(Looper.getMainLooper()) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_ADD_CONNECTION_SERVICE_ADAPTER:
                    // 添加 来自于 system 进程的接口
                    mAdapter.addAdapter((IConnectionServiceAdapter) msg.obj);
                    onAdapterAttached();
                    break;
                case MSG_REMOVE_CONNECTION_SERVICE_ADAPTER:
                    // 移除
                    mAdapter.removeAdapter((IConnectionServiceAdapter) msg.obj);
                    break;
                case MSG_CREATE_CONNECTION: {
                    SomeArgs args = (SomeArgs) msg.obj;
                    try {
                        final PhoneAccountHandle connectionManagerPhoneAccount =
                                (PhoneAccountHandle) args.arg1;
                        final String id = (String) args.arg2;
                        final ConnectionRequest request = (ConnectionRequest) args.arg3;
                        final boolean isIncoming = args.argi1 == 1;
                        final boolean isUnknown = args.argi2 == 1;
                        if (!mAreAccountsInitialized) {
                            Log.d(this, "Enqueueing pre-init request %s", id);
                            mPreInitializationConnectionRequests.add(new Runnable() {
                                @Override
                                public void run() {
                                    createConnection(
                                            connectionManagerPhoneAccount,
                                            id,
                                            request,
                                            isIncoming,
                                            isUnknown);
                                }
                            });
                        } else {
                            // 创建链接
                            createConnection(
                                    connectionManagerPhoneAccount,
                                    id,
                                    request,
                                    isIncoming,
                                    isUnknown);
                        }
                    } finally {
                        args.recycle();
                    }
                    break;
                }
                // ... 省略大量操作
            }
        }
    };
```

创建通话链接，创建完成后，通知system 进程

```java
private void createConnection(
        final PhoneAccountHandle callManagerAccount,
        final String callId,
        final ConnectionRequest request,
        boolean isIncoming,
        boolean isUnknown) {
    // ...

    Connection connection = isUnknown ? onCreateUnknownConnection(callManagerAccount, request)
            : isIncoming ? onCreateIncomingConnection(callManagerAccount, request)
            : onCreateOutgoingConnection(callManagerAccount, request);


    if (connection.getState() != Connection.STATE_DISCONNECTED) {
        // 监听 Connection 的状态变化，通知 system 进程（CallsManager）更新
        // 通过 system 进程提供的 adapter 实现
        addConnection(callId, connection);
    }

    // 通知 system 进程（即CallsManager） 电话链接已经创建完成
    mAdapter.handleCreateConnectionComplete(
            callId,
            request,
            new ParcelableConnection(
                    /// M: CC036: [ALPS01794357] Set PhoneAccountHandle for ECC @{
                    handle,
                    //// @}
                    connection.getState(),
                    connection.getConnectionCapabilities(),
                    connection.getAddress(),
                    connection.getAddressPresentation(),
                    connection.getCallerDisplayName(),
                    connection.getCallerDisplayNamePresentation(),
                    connection.getVideoProvider() == null ?
                            null : connection.getVideoProvider().getInterface(),
                    connection.getVideoState(),
                    connection.isRingbackRequested(),
                    connection.getAudioModeIsVoip(),
                    connection.getConnectTimeMillis(),
                    connection.getStatusHints(),
                    connection.getDisconnectCause(),
                    createIdList(connection.getConferenceables()),
                    connection.getExtras()));
}
```

mAdapter.handleCreateConnectionComplete 调用回到 ConnectionServiceWrapper 的 handleCreateConnectionComplete， 然后回到 Call 的 handleCreateConnectionSuccess

```java
public void handleCreateConnectionSuccess(
            CallIdMapper idMapper,
            ParcelableConnection connection) {

        if (mIsUnknown) {

        } else if (mIsIncoming) {

        } else {
            // 因为时去电的流程，所以会走到这个部分
            for (Listener l : mListeners) {
                // 由于 CallsManager 监听了 Call（在addCall的时候）, 所以 CallsManager 会得到通知。信息最终提交给 com.android.dialer
                l.onSuccessfulOutgoingCall(this,
                        getStateFromConnectionState(connection.getState()));
            }
        }
    }
```

去电链接的真实创建者是 TelephonyConnectionService， onCreateOutgoingConnection

```java
public Connection onCreateOutgoingConnection(
            PhoneAccountHandle connectionManagerPhoneAccount,
            final ConnectionRequest request) {

        final TelephonyConnection connection;
        // 两个创建过程类似，都是创建一个GSM链接
        if (switchPhoneType != PhoneConstants.PHONE_TYPE_NONE) {
            connection = createConnectionForSwitchPhone(switchPhoneType, null, true,
                    request.getAccountHandle());
        } else {
            connection = createConnectionFor(phone, null, true /* isOutgoing */,
                    request.getAccountHandle());
        }
        /// @}
        if (connection == null) {
            // 创建失败
            return Connection.createFailedConnection(
                    DisconnectCauseUtil.toTelecomDisconnectCause(
                            android.telephony.DisconnectCause.OUTGOING_FAILURE,
                            "Invalid phone type"));
        }

        if (isEmergencyNumber) {


        } else {

            // 拨号走这个流程
            placeOutgoingConnection(connection, phone, request);
        }

        return connection;
    }
```

placeOutgoingConnection 只做2件事，一是使用对应的phone创建对应的链接（Original），一是让 TelephonyConnection 和 Original 建立起关联， 一个 TelephonyConnection 维护一个 Original, Original 变化会引起 TelephonyConnection 变化。

```java
private void placeOutgoingConnection(
            TelephonyConnection connection, Phone phone, ConnectionRequest request) {

        if (isEmergencyNumber) {
            // ...
            // 为紧急号码做一些处理
        }
        /// @}
        com.android.internal.telephony.Connection originalConnection;
        try {
            originalConnection =
                    phone.dial(number, null, request.getVideoState(), request.getExtras());
        } catch (CallStateException e) {
            // 失败处理
        }

        if (originalConnection == null) {
            // 失败处理
        } else {
            // TelephonyConnection 监听 phone, 更新时，从originalConnection里面取。
            connection.setOriginalConnection(originalConnection);
        }
    }
```

TelephonyConnection.setOriginalConnection 的实现, 目的主要是为将 mHandler 注册注册到 phone 的监听者列表里面， phone 变化， 会触发 mHandler 里面的handleMessage 被调用。

```java
void setOriginalConnection(com.android.internal.telephony.Connection originalConnection) {

    mOriginalConnection = originalConnection;
    getPhone().registerForPreciseCallStateChanged(
            mHandler, MSG_PRECISE_CALL_STATE_CHANGED, null);
    getPhone().registerForHandoverStateChanged(
            mHandler, MSG_HANDOVER_STATE_CHANGED, null);
    getPhone().registerForRingbackTone(mHandler, MSG_RINGBACK_TONE, null);
    getPhone().registerForDisconnect(mHandler, MSG_DISCONNECT, null);
    /// M: CC031: Radio off notification @{
    getPhone().registerForRadioOffOrNotAvailable(
            mHandler, EVENT_RADIO_OFF_OR_NOT_AVAILABLE, null);
    if (getPhone() instanceof ImsPhone) {
        ((ImsPhone) getPhone()).getDefaultPhone().registerForSpeechCodecInfo(mHandler,
                EVENT_SPEECH_CODEC_INFO, null);
    } else {
        getPhone().registerForSpeechCodecInfo(mHandler, EVENT_SPEECH_CODEC_INFO, null);
    }
    /// @}
    /// M: For 3G VT only @{
    getPhone().registerForVtStatusInfo(mHandler, EVENT_VT_STATUS_INFO, null);
    /// @}
    /// M: For CDMA call accepted @{
    getPhone().registerForCdmaCallAccepted(mHandler, EVENT_CDMA_CALL_ACCEPTED, null);

    registerSuppMessageManager(getPhone());
    /// @}

    getPhone().registerForSuppServiceNotification(mHandler,
}
```

mHandler 收到需要更新的消息后， 触发 updateState, 接着通过 originalConnection 获取到最新状态后（状态为DIALING, 拨号中）， 触发 setDialing, 接着 setState(STATE_DIALING)，

```java
private void setState(int state) {
    if (mState == STATE_DISCONNECTED && mState != state) {
    }
    if (mState != state) {

        onStateChanged(state);

        // 通知 TelephonyConnection 的所有监听者，状态更新
        // 因为上面分析到 ConnectionServiceWrapper 的 addConnection 会让 ConnectionServiceWrapper 加入到 TelephonyConnection 的监听者列表中， 所以ConnectionServiceWrapper 会得到通知, 最终通过 mAdapter 通知 system 进程
        for (Listener l : mListeners) {
            l.onStateChanged(this, state);
        }
    }
}
```

#### GSMPhone GSM类型的手机卡对应的Phone

现在流程回到 phone.dial 只分析GSM类型的phone, GSMPhone.dial 如下

```java
public Connection
    dial (String dialString, UUSInfo uusInfo, int videoState, Bundle intentExtras)
            throws CallStateException {

        return dialInternal(dialString, null, videoState, intentExtras);
    }
```

```java
protected Connection
    dialInternal (String dialString, UUSInfo uusInfo, int videoState, Bundle intentExtras)
            throws CallStateException {
        // ...
        GsmMmiCode mmi =
                GsmMmiCode.newFromDialString(networkPortion, this, mUiccApplication.get());
        if (LOCAL_DEBUG) Rlog.d(LOG_TAG,
                               "dialing w/ mmi '" + mmi + "'...");

        if (mmi == null) {
            /// M: For 3G VT only @{
            //return mCT.dial(newDialString, uusInfo, intentExtras);
            if (videoState == VideoProfile.STATE_AUDIO_ONLY) {
                // 流程会进入这里， mCT 是 GsmCallTracker
                return mCT.dial(newDialString, uusInfo, intentExtras);

            // MTK 为了支持VT加入的代码
            } else {
                if (!is3GVTEnabled()) {
                    throw new CallStateException("cannot vtDial for non-3GVT-capable device");
                }
                return mCT.vtDial(newDialString, uusInfo, intentExtras);
            }
            /// @}
        } else if (mmi.isTemporaryModeCLIR()) {
            // ...
        } else {
            // ...
        }
    }
```

#### GsmCallTracker 调用CI 执行指令，返回结果，通知GSMPhone

GsmCallTracker 会借助 mCi 发送指令，这里面会将包含 mCi 执行结果的 Message 提前发送出去，等到 mCi 执行完毕后， Message 会被发送到对应的 Handler 里面去。

```java
synchronized Connection
    dial (String dialString, int clirMode, UUSInfo uusInfo, Bundle intentExtras)
            throws CallStateException {

        // 如果当前正在打电话时，需要先挂起当前电话，才能拨号。
        if (mForegroundCall.getState() == GsmCall.State.ACTIVE) {
            // ...
        }


        if ( mPendingMO.getAddress() == null || mPendingMO.getAddress().length() == 0
                || mPendingMO.getAddress().indexOf(PhoneNumberUtils.WILD) >= 0
        ) {
            // 处理拨号无效的情况
        } else {
            // 拨号时设置不能静音
            setMute(false);

            /// M: CC015: CRSS special handling @{
            if (!mWaitingForHoldRequest.isWaiting()) {
                /// M: CC010: Add RIL interface @{
                if (PhoneNumberUtils.isEmergencyNumber(mPhone.getSubId(), dialString)
                        && !PhoneNumberUtils.isSpecialEmergencyNumber(dialString)) {
                    // 紧急拨号
                    // ...
                } else {
                    // 正常拨号
                    // 需要注意， EVENT_DIAL_CALL_RESULT 是对应操作的一个句柄， 等到结果返回时， 会让 Handler 进行处理
                    mCi.dial(mPendingMO.getAddress(), clirMode, uusInfo,
                            obtainCompleteMessage(EVENT_DIAL_CALL_RESULT));
                }
            } else {
                mWaitingForHoldRequest.set(mPendingMO.getAddress(), clirMode, uusInfo);
            }
            /// @}
        }

        // 更新 phone 状态
        updatePhoneState();
        mPhone.notifyPreciseCallStateChanged();

        return mPendingMO;
    }
```

GsmCallTracker 本身就是一个 Handler, 它的 handleMessage 将会处理 mCi 返回的结果。

```java
handleMessage (Message msg) {


        switch (msg.what) {
            // ...

            case EVENT_DIAL_CALL_RESULT:
                ar = (AsyncResult) msg.obj;
                if (ar.exception != null) {
                    // 出异常
                    log("dial call failed!!");
                    mHelper.PendingHangupRequestUpdate();
                }
                // 再次获取最新信息
                operationComplete();
            break;
            //...
        }
    }
```

mCi 获取最新状态， 返回到 handleMessage 的 EVENT_POLL_CALLS_RESULT

```java
private void
operationComplete() {

    if (mPendingOperations == 0 && mNeedsPoll) {
        mLastRelevantPoll = obtainMessage(EVENT_POLL_CALLS_RESULT);
        mCi.getCurrentCalls(mLastRelevantPoll);
    } else if (mPendingOperations < 0) {

    }
}
```

```java
handleMessage (Message msg) {


        switch (msg.what) {
            // ...

            case EVENT_POLL_CALLS_RESULT:
                ar = (AsyncResult)msg.obj;

                if (msg == mLastRelevantPoll) {
                    if (DBG_POLL) log(
                            "handle EVENT_POLL_CALLS_RESULT: set needsPoll=F");
                    mNeedsPoll = false;
                    mLastRelevantPoll = null;
                    handlePollCalls((AsyncResult)msg.obj);
                }
            break;
            //...
        }
    }
```

handlePollCalls 处理4种状态后，通知 phone 更新状态。

```java
protected synchronized void
    handlePollCalls(AsyncResult ar) {


        for (int i = 0, curDC = 0, dcSize = polledCalls.size()
                ; i < mConnections.length; i++) {


            if (conn == null && dc != null) {
                // 新的电话
            } else if (conn != null && dc == null) {
                // 旧的电话消失，可能时挂断


            } else if (conn != null && dc != null && !conn.compareTo(dc)) {

                // 旧的电话被替换
            } else if (conn != null && dc != null) {
                // 旧的电话更新, 正在打的电话时，状态更新
            }

        }


        // 过一会在获取最新状态
        if (needsPollDelay) {
            pollCallsAfterDelay();
        }

        // 更新 phone 的状态
        updatePhoneState();

        // hasNonHangupStateChanged 为 true
        if ((hasNonHangupStateChanged || newRinging != null || hasAnyCallDisconnected)

            // 通知 GSMPhone 状态更新
            mPhone.notifyPreciseCallStateChanged();
        }

        //...
    }
```

```java
updatePhoneState() {
    PhoneConstants.State oldState = mState;

    if (mState != oldState) {
        // 更新 phone 的状态
        mPhone.notifyPhoneStateChanged();
    }
}
```

mPhone.notifyPhoneStateChanged => mNotifier.notifyPhoneState , mNotifier 是一个 DefaultPhoneNotifier , notifyPhoneState 会通知 telephony.registry 更新，telephony.registry 会通知所有调用了TelephonyManager.listen 的监听者， 实际上，这是提供给第三方APP监听Phone状态的一个接口。

```java
public void notifyPhoneState(Phone sender) {
    // ...
    try {
        if (mRegistry != null) {
            // mRegistry 是一个远程接口 (TelephonyRegistry)
            // protected DefaultPhoneNotifier() {
            //mRegistry      =ITelephonyRegistry.Stub.asInterface(ServiceManager.getService(
            //        "telephony.registry"));
    }
            mRegistry.notifyCallStateForPhoneInfo(phoneId, phoneType, subId,
                  convertCallState(sender.getState()), incomingNumber);
        }
    } catch (RemoteException ex) {
    }
}
```

mPhone.notifyPreciseCallStateChanged()  ==>  PhoneBase.notifyPreciseCallStateChangedP() 这里会通知 TelephonyConnection 更新。 触发 TelephonyConnection 的 updateState()

接下来流程来到了 mCi.dial, mCi 是一个 CommandsInterface 接口， 由 GsmCallTracker 的构造可知， GsmCallTracker 的 mCi 来自于 GSMPhone 的 mCi， 即 RIL。

### RIL与通话模块底层通讯过程

RIL通讯主要由RILSender和RILReceiver构成，用于通讯传输的载体的有 RILRequest,    
Registrant(RegistrantList)。

1. RILSender
  是一个Handler, 通过 #send(RILRequest) 将请求发送到 mSenderThread线程中，handleMessage 将请求写入mSocket中。

2. RILReceiver
  在run内无限轮询，一旦读取到通话底层返回的数据，交给 #processResponse(Parcel)处理。
  其中应答分为有请求的应答，无请求的应答（即状态改变后的反馈）

  * RIL_REQUEST_xxxx  有请求
  * RIL_UNSOL_xxxx 无请求

3. RILRequest
  成员变量：

  * mSerial 请求序号，唯一，保证请求和反馈的一致。
  * mRequest 请求类型 即 RIL_xxxx。
  * mResult 用来发送请求的结果的句柄，由RIL请求的调用者提供。

4. Registrant
  在 RIL内部维护着一系列的RegistrantList(Registrant的集合)，每个集合代表一种状态改变的类型，对于有请求的应答，RIL是通过对应的mResult来发送结果的，但是对于无请求的应答，RIL是将反馈通知告诉对应的RegistrantList的（#notifyRegistrants
  ），RegistrantList会通知每一个Registrant。
  在RIL的直接父类定义了这些RegistrantList，并提供了注册到RegistrantList的方法（eg .#registerForCallStateChanged，#unregisterForCallStateChanged）  
  所以，如果想要监听通话状态的变化，那么就需要注册监听到对应的RegistrantList（监听着需要提供一个handler和一个int和object，handler用来发送数据，int区分监听的事件，object是额外的信息）。
