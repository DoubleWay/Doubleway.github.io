---
layout:     post
title:      TelePhony--UiccCardApplication
subtitle:   
date:       2020-03-06
author:     DoubleWay
header-img: img/post-bg-YesOrNo.jpg
catalog: 	 true
tags:
    - Android
    - Telephony
---

## TelePhony--UiccCardApplication

可以知道UiccCardApplication，是在UiccCard的update方法里面创建和更新的，还是先从它的public方法来看他的一些功能：

```java
public void registerForReady(Handler h, int what, Object obj) 
public void registerForLocked(Handler h, int what, Object obj)
     /**
     * Notifies handler of any transition into State.isPinLocked()
     */
public void registerForNetworkLocked(Handler h, int what, Object obj) 
public AppState getState() 
public AppType getType() 
public PinState getPin1State() 
public IccFileHandler getIccFileHandler()
public IccRecords getIccRecords()
public void supplyPin (String pin, Message onComplete)
public void supplyPuk (String puk, String newPin, Message onComplete)
public void supplyPin2 (String pin2, Message onComplete)
public void supplyPuk2 (String puk2, String newPin2, Message onComplete) 
public boolean getIccLockEnabled()
public boolean getIccFdnEnabled()
public void setIccLockEnabled (boolean enabled,String password, Message onComplete)
public void setIccFdnEnabled (boolean enabled,String password, Message onComplete) 
public void changeIccLockPassword(String oldPassword, String newPassword,Message onComplete)
public void changeIccFdnPassword(String oldPassword, String newPassword, Message onComplete)
```

从这些方法可以看出UiccCardApplication的主要功能为：

1. 注册PIN锁，网络锁等监听器
2. 查询当前UiccCardApplication的状态信息，如AppState，AppType，PinState等
3. 创建向外提供IccFileHandler，IccRecords对象
4. 设置，查询PIN，PUK状态和密码
5. 查询，设置Fdn的状态

### UiccCardApplication的创建过程

之前知道UiccCard在更新的过程中会创建和更新UiccCardApplication

```java
  mUiccApplications[i] = new UiccCardApplication(this,
                            ics.mApplications[i], mContext, mCi);
```

来看它的构造方法：

`\frameworks\opt\telephony\src\java\com\android\internal\telephony\uicc\UiccCardApplication.java` 

```java
public UiccCardApplication(UiccCard uiccCard,
                    IccCardApplicationStatus as,
                    Context c,
                    CommandsInterface ci) {
    if (DBG) log("Creating UiccApp: " + as);
    mUiccCard = uiccCard;
    mAppState = as.app_state;
    mAppType = as.app_type;
    mAuthContext = getAuthContext(mAppType);
    mPersoSubState = as.perso_substate;
    mAid = as.aid;
    mAppLabel = as.app_label;
    mPin1Replaced = (as.pin1_replaced != 0);
    mPin1State = as.pin1;
    mPin2State = as.pin2;

    mContext = c;
    mCi = ci;
          //创建IccFileHandler对象
    mIccFh = createIccFileHandler(as.app_type);
        //创建IccRecords对象
    mIccRecords = createIccRecords(as.app_type, mContext, mCi);
    if (mAppState == AppState.APPSTATE_READY) {
        //查询Fdn
        queryFdn();
        //查询Pin
        queryPin1State();
    }
    mCi.registerForNotAvailable(mHandler, EVENT_RADIO_UNAVAILABLE, null);
}
```
可以从代码中看到UiccCardApplication在构造函数中主要完成了以下几件事：

1. 根据Uicc卡的种类，创建SIM卡文件管理者IccFileHandler的子类对象

   SIMFileHandler/RuimFileHandler/UsimFileHandler/CsimFileHandler/IsimFileHandler

2. 根据Uicc卡的种类创建SIM卡信息IccRecords的子类对象

   SIMRecords/RuimRecords/IsimUiccRecords

3. 查询Fdn（FDN(Fixed dialer number)方式拨号。就是只允许呼出FDN菜单中自己输入的电话号码，设定指定拨号后，你的手机只能拨出有限的几个号码啦。也只能接听FDN中的号码。但是，紧急呼叫是不受该限制的。设定指定拨号需要你的PIN2码。一般移动公司是不会提供这个码的，需要个人和移动公司交流才能得到。）

4. 查询Pin

5. 取出IccCardApplicationStatus里面的各种状态更新到UiccCardApplication中

在SIM卡和radio发生变化的时候，会通过UiccController更新UiccCard，在由UiccCard更新UiccCardApplication，调用它的update方法：

```java
public void update (IccCardApplicationStatus as, Context c, CommandsInterface ci) {
    synchronized (mLock) {
        if (mDestroyed) {
            loge("Application updated after destroyed! Fix me!");
            return;
        }

        if (DBG) log(mAppType + " update. New " + as);
        mContext = c;
        mCi = ci;
        AppType oldAppType = mAppType;
        AppState oldAppState = mAppState;
        PersoSubState oldPersoSubState = mPersoSubState;
        mAppType = as.app_type;
        mAuthContext = getAuthContext(mAppType);
        mAppState = as.app_state;
        mPersoSubState = as.perso_substate;
        mAid = as.aid;
        mAppLabel = as.app_label;
        mPin1Replaced = (as.pin1_replaced != 0);
        mPin1State = as.pin1;
        mPin2State = as.pin2;

        if (mAppType != oldAppType) {
            if (mIccFh != null) { mIccFh.dispose();}
            if (mIccRecords != null) { mIccRecords.dispose();}
            //重新创建IccFileHandler对象和IccRecords对象
            mIccFh = createIccFileHandler(as.app_type);
            mIccRecords = createIccRecords(as.app_type, c, ci);
        }

        if (mPersoSubState != oldPersoSubState &&
                mPersoSubState == PersoSubState.PERSOSUBSTATE_SIM_NETWORK) {
            //通知网络锁的监听器
            notifyNetworkLockedRegistrantsIfNeeded(null);
        }

        if (mAppState != oldAppState) {
            if (DBG) log(oldAppType + " changed state: " + oldAppState + " -> " + mAppState);
            // If the app state turns to APPSTATE_READY, then query FDN status,
            //as it might have failed in earlier attempt.
            if (mAppState == AppState.APPSTATE_READY) {
                //重新查询Fdn和Pin
                queryFdn();
                queryPin1State();
            }
            //通知Pin锁的监听器
            notifyPinLockedRegistrantsIfNeeded(null);
            notifyReadyRegistrantsIfNeeded(null);
        }
    }
}
```
可以看到UiccCardAppliation在更新的过程中，完成下面几件事情：

1. 更新相关状态的成员变量，如mAppType，mPin1State等
2. 根据情况重新创建创建IccFileHandler对象和IccRecords对象，重新查询Fdn和Pin
3. 通知监听器