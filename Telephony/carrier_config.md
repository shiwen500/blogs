<h2 align = "center">carrier_config相关研究</h2>

------

概述： 在PhoneGlobals创建时，会同时创建carrier_config 服务，实现carrier_config 的类是CarrierConfigLoader。CarrierConfigLoader的主要目的是，绑定特定运营商的App，从而获取特定运营的配置信息。

#### 1. 相关类和aidl

> 1. frameworks/base/telephony/java/com/android/internal/telephony/ICarrierConfigLoader.aidl

> 2. packages/services/Telephony/src/com/android/phone/CarrierConfigLoader.java

> 3. frameworks/base/telephony/java/android/telephony/CarrierConfigManager.java

#### 2. 初始化

```java
    private CarrierConfigLoader(Context context) {
        mContext = context;

        // 监听开机
        IntentFilter bootFilter = new IntentFilter();
        bootFilter.addAction(Intent.ACTION_BOOT_COMPLETED);
        context.registerReceiver(mBootReceiver, bootFilter);

        // Register for package updates. Update app or uninstall app update will have all 3 intents,
        // in the order or removed, added, replaced, all with extra_replace set to true.
        // 监听安装或者卸载
        IntentFilter pkgFilter = new IntentFilter();
        pkgFilter.addAction(Intent.ACTION_PACKAGE_ADDED);
        pkgFilter.addAction(Intent.ACTION_PACKAGE_REMOVED);
        pkgFilter.addAction(Intent.ACTION_PACKAGE_REPLACED);
        pkgFilter.addDataScheme("package");
        context.registerReceiverAsUser(mPackageReceiver, UserHandle.ALL, pkgFilter, null, null);

        int numPhones = TelephonyManager.from(context).getPhoneCount();
        // 运营商默认配置
        mConfigFromDefaultApp = new PersistableBundle[numPhones];
        // 运营商App设置
        mConfigFromCarrierApp = new PersistableBundle[numPhones];
        mServiceConnection = new CarrierServiceConnection[numPhones];
        // Make this service available through ServiceManager.
        // 让外部可以通过ServiceManager访问carrier_config
        ServiceManager.addService(Context.CARRIER_CONFIG_SERVICE, this);
        log("CarrierConfigLoader has started");
        mHandler.sendEmptyMessage(EVENT_CHECK_SYSTEM_UPDATE);
    }
```

在构造函数初始化完成后，发送了EVENT_CHECK_SYSTEM_UPDATE事件，Handler对于该事件的处理如下：

```java
case EVENT_CHECK_SYSTEM_UPDATE:
    SharedPreferences sharedPrefs =
            PreferenceManager.getDefaultSharedPreferences(mContext);
    final String lastFingerprint = sharedPrefs.getString(KEY_FINGERPRINT, null);
    // 如果系统的构建信息发生了变化，即系统更新了，那么就需要清除缓存。
    if (!Build.FINGERPRINT.equals(lastFingerprint)) {
        log("Build fingerprint changed. old: "
                + lastFingerprint + " new: " + Build.FINGERPRINT);
        // 清除缓存
        clearCachedConfigForPackage(null);
        // 更新系统构建信息
        sharedPrefs.edit().putString(KEY_FINGERPRINT, Build.FINGERPRINT).apply();
    }
    break;
```

清除缓存的代码如下，运营商的信息都是通过xml文件保存下来的，packageName为null时会清除所有的xml文件。

```java
private boolean clearCachedConfigForPackage(final String packageName) {
    File dir = mContext.getFilesDir();
    File[] packageFiles = dir.listFiles(new FilenameFilter() {
        public boolean accept(File dir, String filename) {
            if (packageName != null) {
                return filename.startsWith("carrierconfig-" + packageName + "-");
            } else {
                // 因为packageName==null,所以会清除掉所有以carrierconfig-开头的文件。
                return filename.startsWith("carrierconfig-");
            }
        }
    });
    if (packageFiles == null || packageFiles.length < 1) return false;
    for (File f : packageFiles) {
        log("deleting " + f.getName());
        f.delete();
    }
    return true;
}
```

#### 3. 开机更新运营商信息

接收到开机广播后mHandler接收到EVENT_SYSTEM_UNLOCKED事件，对该事件的处理如下:

```java
case EVENT_SYSTEM_UNLOCKED:
                    for (int i = 0; i < TelephonyManager.from(mContext).getPhoneCount(); ++i) {
                        // 如果是单卡，phoneId为0, 双卡：phoneId 为0,1
                        // 更新每个卡的运营商信息
                        updateConfigForPhoneId(i);
                    }
                    break;
```        

更新内存中的运营商信息，如果特定运营商app不存在，那么清空对应的缓存：

```java
private void updateConfigForPhoneId(int phoneId) {
    // Clear in-memory cache for carrier app config, so when carrier app gets uninstalled, no
    // stale config is left.
    if (mConfigFromCarrierApp[phoneId] != null &&
            getCarrierPackageForPhoneId(phoneId) == null) {
        mConfigFromCarrierApp[phoneId] = null;
    }
    // 重新获取运营商信息
    mHandler.sendMessage(mHandler.obtainMessage(EVENT_FETCH_DEFAULT, phoneId, -1));
}
```

首先从本地文件中读取配置信息，如果本地不能存在，那么尝试绑定android.service.carrier.CarrierService，绑定成功后，从该服务的接口中读取信息。

```java
case EVENT_FETCH_DEFAULT:
                    iccid = getIccIdForPhoneId(phoneId);
                    // 从文件中读取默认的配置
                    config = restoreConfigFromXml(DEFAULT_CARRIER_CONFIG_PACKAGE, iccid);
                    if (config != null) {
                        log("Loaded config from XML. package=" + DEFAULT_CARRIER_CONFIG_PACKAGE
                                + " phoneId=" + phoneId);
                        // xml文件存在并加载成功，发送加载默认配置成功事件
                        mConfigFromDefaultApp[phoneId] = config;
                        Message newMsg = obtainMessage(EVENT_LOADED_FROM_DEFAULT, phoneId, -1);
                        newMsg.getData().putBoolean("loaded_from_xml", true);
                        mHandler.sendMessage(newMsg);
                    } else {
                        // xml文件不存在，绑定android.service.carrier.CarrierService，该服务的实现不具体分析。
                        if (bindToConfigPackage(DEFAULT_CARRIER_CONFIG_PACKAGE,
                                phoneId, EVENT_CONNECTED_TO_DEFAULT)) {
                            sendMessageDelayed(obtainMessage(EVENT_BIND_DEFAULT_TIMEOUT, phoneId, -1),
                                    BIND_TIMEOUT_MILLIS);
                        } else {
                            // Send bcast if bind fails
                            // 告诉监听者，运营商信息已经发生了变化，这个时候，特定运营商信息为空。
                            broadcastConfigChangedIntent(phoneId);
                        }
                    }
                    break;
```

上面EVENT_BIND_DEFAULT_TIMEOUT事件比较简单， 首先unBind服务，再通过broadcastConfigChangedIntent通知监听者。

所以应该是在绑定服务成功后，就获取运营商信息，BIND_TIMEOUT_MILLIS是绑定服务的超时时间。下面看bindToConfigPackage：

```java
/** Binds to the default or carrier config app. */
private boolean bindToConfigPackage(String pkgName, int phoneId, int eventId) {
    log("Binding to " + pkgName + " for phone " + phoneId);
    Intent carrierService = new Intent(CarrierService.CARRIER_SERVICE_INTERFACE);
    carrierService.setPackage(pkgName);
    // 很显然绑定成功后的逻辑在CarrierServiceConnection类里面。
    mServiceConnection[phoneId] = new CarrierServiceConnection(phoneId, eventId);
    try {
        return mContext.bindService(carrierService, mServiceConnection[phoneId],
                Context.BIND_AUTO_CREATE);
    } catch (SecurityException ex) {
        return false;
    }
}
```

绑定服务成功后给mHandler发送了EVENT_CONNECTED_TO_DEFAULT事件：

```java
case EVENT_CONNECTED_TO_DEFAULT:
    // 移除超时事件。
    removeMessages(EVENT_BIND_DEFAULT_TIMEOUT);
    carrierId = getCarrierIdForPhoneId(phoneId);
    conn = (CarrierServiceConnection) msg.obj;
    // If new service connection has been created, unbind.
    if (mServiceConnection[phoneId] != conn || conn.service == null) {
        mContext.unbindService(conn);
        break;
    }
    try {
        ICarrierService carrierService = ICarrierService.Stub
                .asInterface(conn.service);
        // 从运营商服务中获取到配置。
        config = carrierService.getCarrierConfig(carrierId);
        iccid = getIccIdForPhoneId(phoneId);
        // 保存配置到xml文件
        saveConfigToXml(DEFAULT_CARRIER_CONFIG_PACKAGE, iccid, config);
        // 更新内存中的配置
        mConfigFromDefaultApp[phoneId] = config;
        sendMessage(obtainMessage(EVENT_LOADED_FROM_DEFAULT, phoneId, -1));
    } catch (Exception ex) {
        // The bound app could throw exceptions that binder will pass to us.
        loge("Failed to get carrier config: " + ex.toString());
    } finally {
        mContext.unbindService(mServiceConnection[phoneId]);
    }
    break;
```

无论是从本地xml文件中获取到运营商配置，还是从服务中获取到运营商配置，最终都会回到mHandler的EVENT_LOADED_FROM_DEFAULT事件：

```java
case EVENT_LOADED_FROM_DEFAULT:
    // If we attempted to bind to the app, but the service connection is null, then
    // config was cleared while we were waiting and we should not continue.
    // 我们是从服务中获取到的配置，但是服务却断开了，当然不能继续了。
    if (!msg.getData().getBoolean("loaded_from_xml", false)
            && mServiceConnection[phoneId] == null) {
        break;
    }
    carrierPackageName = getCarrierPackageForPhoneId(phoneId);
    // 特定的运营商app存在，那么需要从特定的运营商中再次获取配置。
    if (carrierPackageName != null) {
        log("Found carrier config app: " + carrierPackageName);
        sendMessage(obtainMessage(EVENT_FETCH_CARRIER, phoneId));
    } else {
        broadcastConfigChangedIntent(phoneId);
    }
    break;
```

获取特定运营商配置信息和获取默认配置信息的逻辑是一样的，代码就不具体列举了：

> 1. 检查本地缓存文件，缓存存在>>通知变更

> 2. 缓存不存在，那么从特定运营商的服务中获取对应的配置信息，然后缓存到本地，再通知变更。

#### 4. 安装卸载更新运营商信息

理解了上面的更新一个SIM卡的运营商信息的流程，下面就比较简单了。安装卸载会给mHandler发送EVENT_PACKAGE_CHANGED事件：

```java
case EVENT_PACKAGE_CHANGED:
    carrierPackageName = (String) msg.obj;
    // Only update if there are cached config removed to avoid updating config
    // for unrelated packages.
    // 清除掉被卸载的运营商app对应的保存在本地的xml文件。
    if (clearCachedConfigForPackage(carrierPackageName)) {
        int numPhones = TelephonyManager.from(mContext).getPhoneCount();
        for (int i = 0; i < numPhones; ++i) {
            updateConfigForPhoneId(i);
        }
    }
    break;
```

上面的逻辑和开机更新运营商信息差不多，再结合前面的分析，我们知道大体的流程如下：

> 1. 默认运营商配置更新

>> 1.1 检查本地缓存文件， 缓存存在，不需要更新。

> 2. 特定运营商配置更新

>> 2.1 检查本地缓存文件，缓存不存在，需要从服务重新获取配置信息。同时发现特定运营商app不存在，清除特定运营商配置缓存。

>> 2.2 检查特定运营商app，不存在，通知更新。

#### 5. 主动更新

上面所说的都是carrier_config在接收到外部某个事件后，进行更新的逻辑。

carrier_config同样支持主动更新。它为主动更新提供了2个接口：

> 1. notifyConfigChangedForSubId(int subId) 主要是提供给 telephony services 更新它当前的运营商配置用的。

> 2. updateConfigForPhoneId(int phoneId, String simState) 根据提供的sim状态，决定清除该卡的运营商配置还是加载该卡的运营商配置。

#### 6. 获取运营商配置的接口

仅仅是从内存缓存中读取配置信息，首先是读取内存预设好的配置，如果默认运营商配置存在，那么默认配置会覆盖预设配置，然后，如果特定运营商配置存在，那么特定配置覆盖默认配置。

```java
PersistableBundle getConfigForSubId(int subId) {
    try {
        mContext.enforceCallingOrSelfPermission(READ_PRIVILEGED_PHONE_STATE, null);
        // SKIP checking run-time READ_PHONE_STATE since using PRIVILEGED
    } catch (SecurityException e) {
        mContext.enforceCallingOrSelfPermission(READ_PHONE_STATE, null);
    }
    int phoneId = SubscriptionManager.getPhoneId(subId);
    // 优先级是 预设配置 < 默认配置 < 特定配置
    PersistableBundle retConfig = CarrierConfigManager.getDefaultConfig();
    if (SubscriptionManager.isValidPhoneId(phoneId)) {
        PersistableBundle config = mConfigFromDefaultApp[phoneId];
        if (config != null)
            retConfig.putAll(config);
        config = mConfigFromCarrierApp[phoneId];
        if (config != null)
            retConfig.putAll(config);
    }
    return retConfig;
}
```

#### 7. 服务的调用

##### 1) 因为服务加入到了ServiceManager里面，所以对于系统应用，可以通过ServiceManager.getService的方法获取到该服务的接口。

```java
IBinder carrierConfig = ServiceManager.getService(Context.CARRIER_CONFIG_SERVICE);
ICarrierConfigLoader loader = ICarrierConfigLoader.Stub.asInterface(carrierConfig);
try {
    PersistableBundle defaultConfig = loader.getConfigForSubId(SubscriptionManager.getDefaultSubId());
} catch (RemoteException e) {
    e.printStackTrace();
}
```

##### 2) 因为该服务也注册到系统服务里面了，所以第三方应用可以获取该服务接口。

```java
CarrierConfigManager ccm = (CarrierConfigManager) context.getSystemService(Context.CARRIER_CONFIG_SERVICE);
PersistableBundle defaultConfig = ccm.getConfigForSubId(SubscriptionManager.getDefaultSubId());
```


#### 总结

最后需要说明的是，carrier_config 服务自身有一套很完善的更新逻辑，调用者一般情况下不需要主动去触发更新。因为Telephony已经管理好了。
