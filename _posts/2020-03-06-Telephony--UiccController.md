---
layout:     post
title:      Telephony--UiccController
subtitle:   
date:       2020-03-06
author:     DoubleWay
header-img: img/post-bg-BJJ.jpg
catalog: 	 true
tags:
    - Android
    - Telephony
---

## Telephony--UiccController

### UiccController提供的功能

UiccController继承了Handler，是一个Handler的子类，具有分发和处理消息的能力

```java
public class UiccController extends Handler 
```

来看它的一些public方法：

```java
 public UiccCard getUiccCard(int phoneId)
 public UiccCard[] getUiccCards方法，在()
 public IccRecords getIccRecords(int phoneId, int family)
 public IccFileHandler getIccFileHandler(int phoneId, int family) 
 public void registerForIccChanged(Handler h, int what, Object obj) 
 public void unregisterForIccChanged(Handler h)
 public UiccCardApplication getUiccCardApplication(int phoneId, int family)
```
从这一些方法可以看出UiccController的主要功能为：

1. 向外提供UiccCard，IccRecord，IccFileHanlder，UiccCardApplication等对象

2. 监听SIM卡的变化
    这里可以注意一下getUiccCard和getUiccCards方法，在android5.0后谷歌加入了双卡的功能，而体现到具体的代码中就有UICC系统中的UiccCard对象，以及Phone应用中的Phone对象等。其会根据SIM卡的数量分别创建一个或两个实例，而对应的get方法也就需要传入对应的phoneId或slotId，比如此处就要传入phoneId获取对应的UiccCard 对象。 

###UiccController的创建流程

首先Phone对象在在开机时通过PhoneGlobals创建的

`/packages/services/Telephony/src/com/android/phone/PhoneApp.java`

```java
public void onCreate() {
        if (UserHandle.myUserId() == 0) {
           // We are running as the primary user, so should bring up the
            // global phone state.
            mPhoneGlobals = new PhoneGlobals(this);
            mPhoneGlobals.onCreate();

            mTelephonyGlobals = new TelephonyGlobals(this);
            mTelephonyGlobals.onCreate();
        }
    }
```

`/packages/services/Telephony/src/com/android/phone/PhoneGlobals.java`

看PhoneGlobals的onCreate方法：

```java
if (mCM == null) {
            // Initialize the telephony framework
           PhoneFactory.makeDefaultPhones(this);

            // Start TelephonyDebugService After the default phone is created.
            Intent intent = new Intent(this, TelephonyDebugService.class);
            startService(intent);

            mCM = CallManager.getInstance();
            for (Phone phone : PhoneFactory.getPhones()) {
                mCM.registerPhone(phone);
            }
```

`/frameworks/opt/telephony/src/java/com/android/internal/telephony/PhoneFactory.java`

进入PhoneFactory的makeDefaultPhones方法：

```java
 public static void makeDefaultPhone(Context context) {

                           .......................
    
 int numPhones = TelephonyManager.getDefault().getPhoneCount();
               // Start ImsResolver and bind to ImsServices.
               String defaultImsPackage = sContext.getResources().getString(
                        com.android.internal.R.string.config_ims_package);
                Rlog.i(LOG_TAG, "ImsResolver: defaultImsPackage: " + defaultImsPackage);
                sImsResolver = new ImsResolver(sContext, defaultImsPackage, numPhones);
               sImsResolver.populateCacheAndStartBind();

                int[] networkModes = new int[numPhones];
                sPhones = new Phone[numPhones];
                sCommandsInterfaces = new RIL[numPhones];
                sTelephonyNetworkFactories = new TelephonyNetworkFactory[numPhones];

               for (int i = 0; i < numPhones; i++) {
                    // reads the system properties and makes commandsinterface
                    // Get preferred network type.
                networkModes[i] = RILConstants.PREFERRED_NETWORK_MODE;

                Rlog.i(LOG_TAG, "Network Mode set to " + Integer.toString(networkModes[i]));
                    sCommandsInterfaces[i] = new RIL(context, networkModes[i],
                           cdmaSubscription, i);
                }
                Rlog.i(LOG_TAG, "Creating SubscriptionController");
                SubscriptionController.init(context, sCommandsInterfaces);
 // Instantiate UiccController so that all other classes can just
               // call getInstance()
               sUiccController = UiccController.make(context, sCommandsInterfaces);

                       ............................

}
```

这里面通过TelephonyManager.getDefault().getPhoneCount()获取了当前需要创建的数量，根据这个数量创建了networkModes，sPhones，sCommandsInterfaces，sTelephonyNetworkFactories几个数组，然后根据sCommandsInterfaces，创建UiccController对象sUiccController。

这里的sCommandsInterfaces，其实是RIL的实例（sCommandsInterfaces[i] = new RIL（）），用来像UiccController报告SIM卡和radio层的变化。

接下来看make方法中具体的创建过程：

`/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/UiccController.java`

```java
public static UiccController make(Context c, CommandsInterface[] ci) {
        synchronized (mLock) {
           if (mInstance != null) {
                throw new RuntimeException("MSimUiccController.make() should only be called once");
           }
            mInstance = new UiccController(c, ci);
           return (UiccController)mInstance;
       }
   }
```

可以看到UiccController只能调用一次，是单例模式，全局只有一个实例

```java
private UiccController(Context c, CommandsInterface []ci) {
        if (DBG) log("Creating UiccController");
        mContext = c;
        mCis = ci;
        for (int i = 0; i < mCis.length; i++) {
            Integer index = new Integer(i);
            mCis[i].registerForIccStatusChanged(this, EVENT_ICC_STATUS_CHANGED, index);
            // TODO remove this once modem correctly notifies the unsols
            // If the device is unencrypted or has been decrypted or FBE is supported,
           // i.e. not in cryptkeeper bounce, read SIM when radio state isavailable.
            // Else wait for radio to be on. This is needed for the scenario when SIM is locked --
            // to avoid overlap of CryptKeeper and SIM unlock screen.
            if (!StorageManager.inCryptKeeperBounce()) {
                mCis[i].registerForAvailable(this, EVENT_ICC_STATUS_CHANGED, index);
            } else {
                mCis[i].registerForOn(this, EVENT_ICC_STATUS_CHANGED, index);
            }
            mCis[i].registerForNotAvailable(this, EVENT_RADIO_UNAVAILABLE, index);
            mCis[i].registerForIccRefresh(this, EVENT_SIM_REFRESH, index);
        }

        mLauncher = new UiccStateChangedLauncher(c, this);
    }
```

可以再看在UiccController的构造方法中，循环取出RIL的实例，将Uicccontroller自身注册到其中，注册了registerForIccStatusChanged和registerForOn等监听器，这两个监听器分别监听的是SIM卡和恶radio的状态，这里是使用了观察者模式监听了EVENT_ICC_STATUS_CHANGED状态 。

### UiccController的更新机制

上面讲到，UiccController将自己注册到RIL实例中，对SIM卡变化等进行了监听。

`/frameworks/opt/telephony/src/java/com/android/internal/telephony/RIL.java`

在里面可以发现： 

```
switch (rr.mRequest) {
            case RIL_REQUEST_ENTER_SIM_PUK:
           case RIL_REQUEST_ENTER_SIM_PUK2:
                if (mIccStatusChangedRegistrants != null) {
                    if (RILJ_LOGD) {
                      riljLog("ON enter sim puk fakeSimStatusChanged: reg count="
                               + mIccStatusChangedRegistrants.size());
                    }
                   mIccStatusChangedRegistrants.notifyRegistrants();
              }

              break;
         case RIL_REQUEST_SHUTDOWN:
             setRadioState(RadioState.RADIO_UNAVAILABLE);
              break;
     }
```

像出现了puk码错误时，会调用notifyRegistrants方法，因为UiccController是handler的子类，所以会调用handleMessage方法：

```java
public void handleMessage (Message msg) {
        synchronized (mLock) {
            Integer index = getCiIndex(msg);

           if (index < 0 || index >= mCis.length) {
               Rlog.e(LOG_TAG, "Invalid index : " + index + " received with event " + msg.what);
               return;
            }

           AsyncResult ar = (AsyncResult)msg.obj;
           switch (msg.what) {
               case EVENT_ICC_STATUS_CHANGED:
                   if (DBG) log("Received EVENT_ICC_STATUS_CHANGED, calling getIccCardStatus");
                   mCis[index].getIccCardStatus(obtainMessage(EVENT_GET_ICC_STATUS_DONE, index));
                   break;
               case EVENT_GET_ICC_STATUS_DONE:
                   if (DBG) log("Received EVENT_GET_ICC_STATUS_DONE");
                   onGetIccCardStatusDone(ar, index);
                   break;
               case EVENT_RADIO_UNAVAILABLE:
                   if (DBG) log("EVENT_RADIO_UNAVAILABLE, dispose card");
                   if (mUiccCards[index] != null) {
                       mUiccCards[index].dispose();
                   }
                   mUiccCards[index] = null;
                   mIccChangedRegistrants.notifyRegistrants(new AsyncResult(null, index, null));
                   break;
               case EVENT_SIM_REFRESH:
                   if (DBG) log("Received EVENT_SIM_REFRESH");
                   onSimRefresh(ar, index);
                   break;
               default:
                   Rlog.e(LOG_TAG, " Unknown Event " + msg.what);
           }
       }
   }
```

这里因为前面注册的是EVENT_ICC_STATUS_CHANGED，所以会通过对应的index再次使用相应的RIL对象去查询当前SIM卡的状态，注意Message.what=EVENT_GET_ICC_STATUS_DONE，同样传入index参数，这是双卡特别参数。在RIL查询完SIM卡状态后会封装msg.obj再次发回来，进入case EVENT_GET_ICC_STATUS_DONE:  

```java
 private synchronized void onGetIccCardStatusDone(AsyncResult ar, Integer index) {
        if (ar.exception != null) {
           Rlog.e(LOG_TAG,"Error getting ICC status. "
                    + "RIL_REQUEST_GET_ICC_STATUS should "
                    + "never return an error", ar.exception);
            return;
        }
        if (!isValidCardIndex(index)) {
            Rlog.e(LOG_TAG,"onGetIccCardStatusDone: invalid index : " + index);
            return;
        }

        IccCardStatus status = (IccCardStatus)ar.result;

        if (mUiccCards[index] == null) {
            //Create new card
            mUiccCards[index] = new UiccCard(mContext, mCis[index], status, index);
        } else {
            //Update already existing card
            mUiccCards[index].update(mContext, mCis[index] , status);
        }

        if (DBG) log("Notifying IccChangedRegistrants");
        mIccChangedRegistrants.notifyRegistrants(new AsyncResult(null, index, null));

    }
```

在这里通过ar.result获得了记录SIM卡实际状态信息的 IccCardStatus实例，并以此来创建或更新对应的UiccCard对象。同时也通过mIccChangedRegistrants.notifyRegistrants();通知UiccController中的SIM卡状态监听者。
1、UiccCard对象是在UiccController更新SIM卡状态时创建的;
2、其他对象可以通过registerForIccChanged()方法向UiccController申请监听SIM卡状态监听。
