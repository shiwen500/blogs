<h2 align = "center">iphonesubinfo相关研究</h2>

------

概述：在PhoneApp启动时，会构建PhoneGlobals，PhoneGlobals构建时，调用PhoneFactory. makeDefaultPhones构建Phone对象，同时也会对ProxyController进行初始化，ProxyController会构造PhoneSubInfoController。

PhoneSubInfoController就是本文的主角，它是服务iphonesubinfo的实现者，并且在PhoneSubInfoController创建时，通过ServiceManager将iphonesubinfo服务的binder接口添加尽了系统服务中。

总的来说，iphonesubinfo服务是提供SIM卡信息查询的接口。

### 服务的主要责任

服务主要是对外提供SIM卡信息的查询功能，包括但不限于：deviceId(IMEI for GSM),nai(可为空),imei,deviceSvn(IMEI/SV for GSM),SubscriberId(IMSI for GSM), IccId, Line1Number(电话号码), Msisdn,VoiceMailNumber,CompleteVoiceMailNumber,Isim信息，Usim信息.

下面主要对GSM类型的手机进行分析：

#### deviceId 设备唯一标识

即SIM卡槽的IMEI码，另外在CDMA手机上，是MEID码；每一个卡槽都有一个IMEI码，换卡IMEI不变。

#### NAI 接入点网络 Network Access ID

NAI的全称是Network Access Identifier，为接入点网络，手机通过NAI上网接入网络。用户通过移动终端选择不同的NAI 可实现不同的网络接入方式，并可访问不同业务。

目前中国电信主要的NAI 为CTNET、CTWAP。

#### IMEI GSM手机上的设备唯一标识码

如果没有设置，可能为空；在支持GSM的手机上都会设置该码。

#### DeviceSvn  IMEI/SV

手机SIM卡槽的软件版本号，例如IMEI/SV；每一个卡槽对应一个IMEI/SV

#### SubscriberId 唯一的SubscriberId

SIM卡的唯一订阅ID，GSM上是IMEI码。

#### ICCID 由一串十进制数字和一串十六进制数字组成

iccId 是不包含十六进制字符的iccId, fullIccId是包含十六进制的iccId。

#### Line1Number 手机号码

Line1Number 表示卡的手机号码，在GSm卡上是 MSIDN 码， 在CDMA卡上是 MDN码。

#### MSIDN

在GSM卡上是手机号码，即和Line1Number的值一样；在CDMA卡就是独立的MSIDN码；所以如果需要MSIDN码，在CDMA卡上可以通过这个接口获取。

#### VoiceMailNumber 语音信箱

去除了网络部分的语音信箱

#### CompleteVoiceMailNumber 完整的语音信箱

包含网络部分

#### ISIM (IMS Subscriber Identity Module)

多媒体业务身份模块

>  1. IMPI (IMS private user identity) IMS服务的私有用户id

>  2. Domain (IMS home network domain name) IMS服务网络的域名

>  3. IMPU (an array of IMS public user identities) 一组共有用户id

>  4. IST (IMS Service Table ) IMS 服务表

>  5. PCSCF (IMS Proxy Call Session Control Function) IMS 代理呼叫会话控制功能

>  6. ChallengeResponse (the response of ISIM Authetification through RIL) ISIM通过RIL进行验证的结果

>  7. (MTK添加) GBABP (GBA bootstrapping parameters) GBA引导的参数

>  8. (MTK添加) PSISMSC （the Public Service Identity of the SM-SC） SM-SC的公共服务id

#### USIM（Universal Subscriber Identity Module）

通用业务身份模块

>  1. getUsimService(int service, String callingPackage) 服务是否可用，传入的service是frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/UsimServiceTable.java表中的枚举的索引值。

>  2. Gbabp (GBA bootstrapping parameters) GBA引导参数

>  3. PSISMSC （the Public Service Identity of the SM-SC） SM-SC的公共服务id

>  4. SMSP 短信息中心

>  5. MNC 移动网号

### 服务的调用

下面是一个调用服务的例子

```java
        IPhoneSubInfo iPhoneSubInfo = IPhoneSubInfo.Stub.asInterface(ServiceManager.getService("iphonesubinfo"));
        try {
            android.util.Log.d("Seven", "My package name: " + getPackageName());
            String number = iPhoneSubInfo.getLine1NumberForSubscriber(
                    SubscriptionManager.getDefaultSubId(),
                    getPackageName()
            );
            android.util.Log.d("Seven", "My default phone number is: " + number);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
```

由于该服务没有注册到SystemService里面，所以，对于非依赖系统编译的第三方应用，无法直接访问该服务。因为只能通过ServiceManager.getService（）的方式获取Binder接口。

另外需要说明的是：一个sim卡插入手机内，会在数据库中生成一条Subscription记录，subId就是该条记录的ID。每个手机卡的subId都是唯一的，即使把手机卡拔了，该条记录也一直存在。

#### slotId, phoneId, subId 的关系

slotId 和 phoneId 一一对应， 假设有2个卡槽， 那么slotId 为 0, 1；卡槽1的phoneId 为 0, 卡槽2的phoneId为 1。

一个卡槽对应着多个subId，因为该卡槽可能曾经插过多张不同的卡。

所以我们可以通过phoneId查询subId。
