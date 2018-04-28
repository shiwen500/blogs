<h2 align = "center">来电流程</h2>

----

概述：来电的过程，是在去电（拨号）的基础上分析的。

来电过程， 是由com.android.phone进程发起的，因为 com.android.phone 进程直接与Moderm层交互， com.android.phone 进程收到来来电消息后，发送消息给 system 进程， system 进程开始和com.android.phone 进程建立链接， 并通知 UI 进程 (com.android.dialer) 更新。大体上和拨号过程类似。

### com.android.phone 进程

#### 1.开机初始化的过程

PhoneApp 是 com.android.phone 进程的 Application 实例， 在它的 onCreate 中

```java
public void onCreate() {
        if (UserHandle.myUserId() == 0) {

            // 创建 Phones RIL 等
            mPhoneGlobals = new PhoneGlobals(this);
            mPhoneGlobals.onCreate();

            mTelephonyGlobals = new TelephonyGlobals(this);

            // 主要是监听 phone 的状态变化
            mTelephonyGlobals.onCreate();
        }
    }
```

mTelephonyGlobals.onCreate()  ==>  TelecomAccountRegistry.setupOnBoot()

```java
void setupOnBoot() {

    listenPhoneState();

}
```

```java
private void listenPhoneState() {
        Phone[] phones = PhoneFactory.getPhones();
        for (Phone phone : phones) {
            int subscriptionId = phone.getSubId();
            if (subscriptionId >= 0 && !mPhoneStateListeners.containsKey(subscriptionId)) {
                // ...

                PhoneStateListener listener = new PhoneStateListener(subscriptionId) {
                    @Override
                    public void onServiceStateChanged(ServiceState serviceState) {
                        // ...
                        if (rebuildAccounts) {

                            tearDownAccounts();
                            // 只关注 setupAccounts
                            setupAccounts();
                            broadcastAccountChanged();
                        }

                    }
                };
                // 注册监听到 TelephonyRegistry， 如果在开机， 或者更换手机卡时， 这个listener将被触发
                mTelephonyManager.listen(listener, PhoneStateListener.LISTEN_SERVICE_STATE);
                mPhoneStateListeners.put(subscriptionId, listener);
            }
        }
    }
```

为每个 Phone 设置 AccountEntry。

```java
private void setupAccounts() {
        // Go through SIM-based phones and register ourselves -- registering an existing account
        // will cause the existing entry to be replaced.
        Phone[] phones = PhoneFactory.getPhones();
        Log.d(this, "Found %d phones.  Attempting to register.", phones.length);
        for (Phone phone : phones) {
            int subscriptionId = phone.getSubId();
            Log.d(this, "Phone with subscription id %d", subscriptionId);
            if (subscriptionId >= 0) {

                // 每一个 subscriptionId >= 0 的 phone 对应一个 AccountEntry
                mAccounts.add(new AccountEntry(phone, false /* emergency */, false /*isDummy*/));
            }
        }
        // ... 省略大量代码
    }
```

AccountEntry 的构造函数

```java
AccountEntry(Phone phone, boolean isEmergency, boolean isDummy) {
            mPhone = phone;
            mAccount = registerPstnPhoneAccount(isEmergency, isDummy);

            // 来电通知监听。 核心过程
            mIncomingCallNotifier = new PstnIncomingCallNotifier((PhoneProxy) mPhone);
            mPhoneCapabilitiesNotifier = new PstnPhoneCapabilitiesNotifier((PhoneProxy) mPhone,
                    this);
        }
```

PstnIncomingCallNotifier 的构造函数  ==>  registerForNotifications()

```java
private void registerForNotifications() {
    Phone newPhone = mPhoneProxy.getActivePhone();
    if (newPhone != mPhoneBase) {
        unregisterForNotifications();

        if (newPhone != null) {
            Log.i(this, "Registering: %s", newPhone);
            mPhoneBase = newPhone;

            // 为 注册监听 来电事件
            mPhoneBase.registerForNewRingingConnection(
                    mHandler, EVENT_NEW_RINGING_CONNECTION, null);
            mPhoneBase.registerForCallWaiting(
                    mHandler, EVENT_CDMA_CALL_WAITING, null);
            mPhoneBase.registerForUnknownConnection(mHandler, EVENT_UNKNOWN_CONNECTION,
                    null);

            mPhoneBase.registerForVoiceCallIncomingIndication(
                    mHandler, EVENT_VOICE_CALL_INCOMING_INDICATION , null);

        }
    }
}
```

#### 2.收到来电消息，通知 system 进程

收到来电通知时， 会触发 mHandler.handleMessage,  （句柄 EVENT_NEW_RINGING_CONNECTION）,  直接调用 handleNewRingingConnection

```java
private void handleNewRingingConnection(AsyncResult asyncResult) {
    Log.d(this, "handleNewRingingConnection");
    Connection connection = (Connection) asyncResult.result;
    if (connection != null) {
        Call call = connection.getCall();
        // Final verification of the ringing state before sending the intent to Telecom.
        if (call != null && call.getState().isRinging()) {
            // 流程将会进入这里
            sendIncomingCallIntent(connection);
        }
    }
}
```

```java
private void sendIncomingCallIntent(Connection connection) {
    // ...
    PhoneAccountHandle handle = findCorrectPhoneAccountHandle();
    if (handle == null) {
        try {
            connection.hangup();
        } catch (CallStateException e) {
            // connection already disconnected. Do nothing
        }
    } else {
        // handle 不为空， 流程进入这里
        TelecomManager.from(mPhoneProxy.getContext()).addNewIncomingCall(handle, extras);
    }
}
```

TelecomManager 通过 TelecomService 的跨进程接口 addNewIncomingCall ， 给 system 进程发送消息。

#### 3. RIL收到来电消息，通知Phone, Phone通知其监听者

有来电时， RIL 将会收到 RIL_UNSOL_RESPONSE_CALL_STATE_CHANGED 消息， 因为 GsmCallTracker 监听了 RIL， GsmCallTracker 的 handleMessage(事件是EVENT_CALL_STATE_CHANGE) 被触发，接着调用 CallTracker.pollCallsWhenSafe 再次获取消息， 最后会进入 GsmCallTracker 的 handlePollCalls。

```java
handlePollCalls(AsyncResult ar) {

    Connection newRinging = null; //or waiting

    // ...  省略大量处理代码， 如果有来电 newRinging 不为空

    if (newRinging != null) {
        if (DBG_POLL) log("notifyNewRingingConnection");

        // 通知 phone 的监听者， 由新来电。
        mPhone.notifyNewRingingConnection(newRinging);

    }

    // ...
}
```

### system 进程

流程来到 TelecomServiceImpl 的 addNewIncomingCall

```java
public void addNewIncomingCall(PhoneAccountHandle phoneAccountHandle, Bundle extras) {
            synchronized (mLock) {
                Log.i(this, "Adding new incoming call with phoneAccountHandle %s",
                        phoneAccountHandle);
                if (phoneAccountHandle != null && phoneAccountHandle.getComponentName() != null) {

                    try {
                        Intent intent = new Intent(TelecomManager.ACTION_INCOMING_CALL);
                        intent.putExtra(TelecomManager.EXTRA_PHONE_ACCOUNT_HANDLE,
                            phoneAccountHandle);
                        intent.putExtra(CallIntentProcessor.KEY_IS_INCOMING_CALL, true);
                        if (extras != null) {
                            intent.putExtra(TelecomManager.EXTRA_INCOMING_CALL_EXTRAS, extras);
                        }

                        // 流程进入这里
                        CallIntentProcessor.processIncomingCallIntent(mCallsManager, intent);
                    } finally {
                        Binder.restoreCallingIdentity(token);
                    }
                } else {
                    Log.w(this,
                            "Null phoneAccountHandle. Ignoring request to add new incoming call");
                }
            }
        }
```

CallIntentProcessor.processIncomingCallIntent  ==>  callsManager.processIncomingCallIntent

```java
void processIncomingCallIntent(PhoneAccountHandle phoneAccountHandle, Bundle extras) {
    Log.d(this, "processIncomingCallIntent");
    Uri handle = extras.getParcelable(TelecomManager.EXTRA_INCOMING_CALL_ADDRESS);
    if (handle == null) {
        // Required for backwards compatibility
        handle = extras.getParcelable(TelephonyManager.EXTRA_INCOMING_NUMBER);
    }

    // 创建一个 call，这里不同于 去电，去电是调用 startOutgoingCall 创建 call 的
    Call call = new Call(
            mContext,
            this,
            mLock,
            mConnectionServiceRepository,
            mContactsAsyncHelper,
            mCallerInfoAsyncQueryFactory,
            handle,
            null /* gatewayInfo */,
            null /* connectionManagerPhoneAccount */,
            phoneAccountHandle,
            true /* isIncoming */,
            false /* isConference */);

    call.setIntentExtras(extras);
    /// M: For VoLTE @{
    if (TelecomVolteUtils.isConferenceInvite(extras)) {
        call.setIsIncomingFromConfServer(true);
    }
    /// @}

    // TODO: Move this to be a part of addCall()
    call.addListener(this);

    // 接下来是创建 和 com.android.phone 进程之间的链接的过程
    // 和 去电的过程是类似的。
    call.startCreateConnection(mPhoneAccountRegistrar);
}
```

由去电的流程知道，当CallsManager.addCall 被调用后， 会触发InCallActicity 启动。 去电和来电第一次调用 addCall 的时机不一样

* 去电在新建Call的时候，就调用了addCall.
* 来电则是在call.startCreateConnection 执行后， system 进程和 com.android.phone 进程建立链接成功后， 由 com.android.phone 进程告诉 system 进程 handleCreateConnectionSuccess, 然后经过层层回调，来到 Call.handleCreateConnectionSuccess

```java
public void handleCreateConnectionSuccess(
            CallIdMapper idMapper,
            ParcelableConnection connection) {
        Log.v(this, "handleCreateConnectionSuccessful %s", connection);
        setTargetPhoneAccount(connection.getPhoneAccount());
        setHandle(connection.getHandle(), connection.getHandlePresentation());
        setCallerDisplayName(
                connection.getCallerDisplayName(), connection.getCallerDisplayNamePresentation());
        setConnectionCapabilities(connection.getConnectionCapabilities());
        setVideoProvider(connection.getVideoProvider());
        setVideoState(connection.getVideoState());
        setRingbackRequested(connection.isRingbackRequested());
        setIsVoipAudioMode(connection.getIsVoipAudioMode());
        setStatusHints(connection.getStatusHints());
        setExtras(connection.getExtras());

        mConferenceableCalls.clear();
        for (String id : connection.getConferenceableConnectionIds()) {
            mConferenceableCalls.add(idMapper.getCall(id));
        }
        // 未知电话， 通知 CallsManager.onSuccessfulUnknownCall
        if (mIsUnknown) {
            for (Listener l : mListeners) {
                l.onSuccessfulUnknownCall(this, getStateFromConnectionState(connection.getState()));
            }
        // 来电， 由 mDirectToVoicemailRunnable.run 间接 通知 CallsManager.onSuccessfulIncomingCall
        } else if (mIsIncoming) {
            // We do not handle incoming calls immediately when they are verified by the connection
            // service. We allow the caller-info-query code to execute first so that we can read the
            // direct-to-voicemail property before deciding if we want to show the incoming call to
            // the user or if we want to reject the call.
            mDirectToVoicemailQueryPending = true;

            // Timeout the direct-to-voicemail lookup execution so that we dont wait too long before
            // showing the user the incoming call screen.
            mHandler.postDelayed(mDirectToVoicemailRunnable, Timeouts.getDirectToVoicemailMillis(
                    mContext.getContentResolver()));
        // 去电， 通知 CallsManager.onSuccessfulOutgoingCall
        } else {
            /// M: ALPS02568075 @{
            // when perform ECC retry from dialing state
            // force to set as CONNECTING this time
            // in order to reset audio mode as normal in CallAudioManager.onCallStateChanged
            // then Ecc is changed to dialing again, can set audio mode once time.
            if (isEmergencyCall() && getState() == CallState.DIALING) {
                Log.v(this, "Change ecc state as connecting");
                for (Listener l : mListeners) {
                    l.onSuccessfulOutgoingCall(this, CallState.CONNECTING);
                }
            }
            /// @}
            for (Listener l : mListeners) {
                l.onSuccessfulOutgoingCall(this,
                        getStateFromConnectionState(connection.getState()));
            }
        }
    }
```

CallsManager.onSuccessfulIncomingCall

```java
public void onSuccessfulIncomingCall(Call incomingCall) {
    Log.d(this, "onSuccessfulIncomingCall");
    setCallState(incomingCall, CallState.RINGING, "successful incoming call");

    if (hasEmergencyCall() || hasMaximumRingingCalls() || hasMaximumDialingCalls()
            || shouldBlockFor3GVT(incomingCall.isVideoCall())) {
        // ....
    } else {
        // 最终会allCall, 从而触发 InCallActicity 启动。
        addCall(incomingCall);
    }
}
```

#### GsmCallTracker.handlePollCalls 对于来电的处理， 和去电不一样

```java
protected synchronized void
    handlePollCalls(AsyncResult ar) {


        for (int i = 0, curDC = 0, dcSize = polledCalls.size()
                ; i < mConnections.length; i++) {


            if (conn == null && dc != null) {
                // 新的电话
                if (mPendingMO != null && mPendingMO.compareTo(dc)) {
                   // 去电
                } else {
                   // 来电，创建一个GsmConnection， TelephonyConnection 中的 originalConnection 就是 GsmConnection
                   mConnections[i] = new GsmConnection(mPhone.getContext(), dc, this, i);
                }
            } else if (conn != null && dc == null) {
                // 旧的电话消失，可能时挂断


            } else if (conn != null && dc != null && !conn.compareTo(dc)) {

                // 旧的电话被替换
            } else if (conn != null && dc != null) {
                // 旧的电话更新, 正在打的电话时，状态更新
            }

        }
        //...
    }
```

在 GsmCallTracker 中有3种类型的 GsmCall

* mRingingCall
* mForegroundCall
* mBackgroundCall

在 new GsmConnection 时， GsmConnection 会添加到对应的 Call 中， 也就是说，来电到来时， GsmConnection 已经准备好了，创建TelephonyConnection可以直接获取mRingingCall 中的GsmConnection。

TelephonyConnectionService.onCreateIncomingConnection 中

```java
public Connection onCreateIncomingConnection(
            PhoneAccountHandle connectionManagerPhoneAccount,
            ConnectionRequest request) {

        // ....
        // ... 获取对应 的GSMPhone
        Phone phone = getPhoneForAccount(accountHandle, isEmergency);
        if (phone == null) {
            return Connection.createFailedConnection(
                    DisconnectCauseUtil.toTelecomDisconnectCause(
                            android.telephony.DisconnectCause.ERROR_UNSPECIFIED,
                            "Phone is null"));
        }

        // 等于是获取GsmCallTracker里面的mRingingCall
        Call call = phone.getRingingCall();
        if (!call.getState().isRinging()) {
            Log.i(this, "onCreateIncomingConnection, no ringing call");
            return Connection.createFailedConnection(
                    DisconnectCauseUtil.toTelecomDisconnectCause(
                            android.telephony.DisconnectCause.INCOMING_MISSED,
                            "Found no ringing call"));
        }

        // 获取originalConnection
        com.android.internal.telephony.Connection originalConnection =
                call.getState() == Call.State.WAITING ?
                    call.getLatestConnection() : call.getEarliestConnection();
        if (isOriginalConnectionKnown(originalConnection)) {
            Log.i(this, "onCreateIncomingConnection, original connection already registered");
            return Connection.createCanceledConnection();
        }

        // 创建TelephonyConnection 与 GsmConnection（originalConnection） 绑定
        // 链接创建成功
        Connection connection =
                createConnectionFor(phone, originalConnection, false /* isOutgoing */,
                        request.getAccountHandle());
        if (connection == null) {
            return Connection.createCanceledConnection();
        } else {
            return connection;
        }
    }
```



### 因为接下来的流程和去电类似，请参考去电过程。

* system 进程和 com.android.phone 绑定成功后， system 进程再和 com.android.dialer 进程绑定， 并启动 InCallActicity。
* RIL 获取到更新后，通过 CallsManager 通知 system 进程
* system 进程通知 com.android.dialer 进程， 然后 InCallPresenter 更新 UI
