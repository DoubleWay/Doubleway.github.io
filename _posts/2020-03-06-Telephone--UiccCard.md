---
layout:     post
title:      Telephone--UiccCard
subtitle:   
date:       2020-03-06
author:     DoubleWay
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - Android
    - Telephony
---

## Telephone--UiccCard

### UiccCard的主要功能

先来看一下他的一些public方法：

```java
public void registerForAbsent(Handler h, int what, Object obj)

public boolean isApplicationOnIcc(IccCardApplicationStatus.AppType type)

public CardState getCardState()

public PinState getUniversalPinState() 

public UiccCardApplication getApplication(int family)

public UiccCardApplication getApplicationIndex(int index)
```

从这些方法可以看出，UiccCard的主要功能为：

1. 注册SIM卡是否存在的监听器
2. 获得SIM卡的状态对象 CardState
3. 创建并向外提供UiccCardApplication对象

也可以看出，一个UiccCard对象就代表了一张SIM卡（phoneID），SIM卡的状态和数据都可以在这里获取，在类中，属性mCardState保存了SIM卡的状态，属性mUniversalPinState保存了PIN码的状态，mCatService代表了STK应用信息，mUiccApplications包含了SIM卡的具体数据等。

### UiccCard的创建和更新过程

在UiccController中提到过，在接收到RIL传递的SIM卡状态的更新信息后，回创建或更新UiccCard对象

`\frameworks\opt\telephony\src\java\com\android\internal\telephony\uicc\UiccController.java`

```java
private synchronized void onGetIccCardStatusDone(AsyncResult ar, Integer index) {
   ...........................

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
第一次会调用构造方法创建UiccCard对象，先来看看UiccCard的构造方法：

`\frameworks\opt\telephony\src\java\com\android\internal\telephony\uicc\UiccCard.java`

```java
public UiccCard(Context c, CommandsInterface ci, IccCardStatus ics, int phoneId) {
        if (DBG) log("Creating");
        mCardState = ics.mCardState;
        mPhoneId = phoneId;
        update(c, ci, ics);
    }


```

可以看到构造函数里比较简单，初始化CardState对象和PhoneId对象后，就调用update函数更新自己的状态

前面可以看到UiccController中SIM卡已经存在状态有变化，会调用update方法，构造方法中也会调用update方法，所以这个update方法很重要。

```java
public void update(Context c, CommandsInterface ci, IccCardStatus ics) {
    synchronized (mLock) {
        CardState oldState = mCardState;
        mCardState = ics.mCardState;
        mUniversalPinState = ics.mUniversalPinState;
        mGsmUmtsSubscriptionAppIndex = ics.mGsmUmtsSubscriptionAppIndex;
        mCdmaSubscriptionAppIndex = ics.mCdmaSubscriptionAppIndex;
        mImsSubscriptionAppIndex = ics.mImsSubscriptionAppIndex;
        mContext = c;
        mCi = ci;

        //update applications
        if (DBG) log(ics.mApplications.length + " applications");
        for ( int i = 0; i < mUiccApplications.length; i++) {
            if (mUiccApplications[i] == null) {
                //Create newly added Applications 创建UiccCardApplication对象
                if (i < ics.mApplications.length) {
                    mUiccApplications[i] = new UiccCardApplication(this,
                            ics.mApplications[i], mContext, mCi);
                }
            } else if (i >= ics.mApplications.length) {
                //Delete removed applications
                mUiccApplications[i].dispose();
                mUiccApplications[i] = null;
            } else {
                //Update the rest 更新UiccCardApplication对象
                mUiccApplications[i].update(ics.mApplications[i], mContext, mCi);
            }
        }
         //创建CatService对象
        createAndUpdateCatServiceLocked();

        // Reload the carrier privilege rules if necessary.
        log("Before privilege rules: " + mCarrierPrivilegeRules + " : " + mCardState);
        if (mCarrierPrivilegeRules == null && mCardState == CardState.CARDSTATE_PRESENT) {
            mCarrierPrivilegeRules = new UiccCarrierPrivilegeRules(this,
                    mHandler.obtainMessage(EVENT_CARRIER_PRIVILEGES_LOADED));
        } else if (mCarrierPrivilegeRules != null
                && mCardState != CardState.CARDSTATE_PRESENT) {
            mCarrierPrivilegeRules = null;
        }

        sanitizeApplicationIndexesLocked();

        RadioState radioState = mCi.getRadioState();
        if (DBG) log("update: radioState=" + radioState + " mLastRadioState="
                + mLastRadioState);
        // No notifications while radio is off or we just powering up
        if (radioState == RadioState.RADIO_ON && mLastRadioState == RadioState.RADIO_ON) {
            if (oldState != CardState.CARDSTATE_ABSENT &&
                    mCardState == CardState.CARDSTATE_ABSENT) {
                if (DBG) log("update: notify card removed");
                //SIM卡被拔出，通知监听者，发送信息
                mAbsentRegistrants.notifyRegistrants();
                mHandler.sendMessage(mHandler.obtainMessage(EVENT_CARD_REMOVED, null));
            } else if (oldState == CardState.CARDSTATE_ABSENT &&
                    mCardState != CardState.CARDSTATE_ABSENT) {
                if (DBG) log("update: notify card added");
                //插入SIM卡，发出信息
                mHandler.sendMessage(mHandler.obtainMessage(EVENT_CARD_ADDED, null));
            }
        }
        mLastRadioState = radioState;
    }
}
```
可以看到在update方法中，先从IccCardStatus取出各个状态更新到UiccCard中，然后创建或更新UiccCardApplication对象，初始化CatService对象，还有就是监测到SIM卡的拔除和插入后，通过mHandler发送信息。

在通过mHandler发送信息后的具体操作：

```java
 protected Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg){
            switch (msg.what) {
                case EVENT_CARD_REMOVED:
                    onIccSwap(false);
                    break;
                case EVENT_CARD_ADDED:
                    onIccSwap(true);
                    break;
```

在收到消息后，无论是REMOVED还是ADDED都会调用onIccSwap（）方法。

```java
private void onIccSwap(boolean isAdded) {

    boolean isHotSwapSupported = mContext.getResources().getBoolean(
            R.bool.config_hotswapCapable);

    if (isHotSwapSupported) {
        log("onIccSwap: isHotSwapSupported is true, don't prompt for rebooting");
        return;
    }
    log("onIccSwap: isHotSwapSupported is false, prompt for rebooting");

    promptForRestart(isAdded);
}
```
在onIccSwap里面，会继续调用 promptForRestart(isAdded)方法

```java
private void promptForRestart(boolean isAdded) {
    synchronized (mLock) {
        final Resources res = mContext.getResources();
        final String dialogComponent = res.getString(
                R.string.config_iccHotswapPromptForRestartDialogComponent);
        if (dialogComponent != null) {
            Intent intent = new Intent().setComponent(ComponentName.unflattenFromString(
                    dialogComponent)).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                    .putExtra(EXTRA_ICC_CARD_ADDED, isAdded);
            try {
                mContext.startActivity(intent);
                return;
            } catch (ActivityNotFoundException e) {
                loge("Unable to find ICC hotswap prompt for restart activity: " + e);
            }
        }

        // TODO: Here we assume the device can't handle SIM hot-swap
        //      and has to reboot. We may want to add a property,
        //      e.g. REBOOT_ON_SIM_SWAP, to indicate if modem support
        //      hot-swap.
        //注册dialog的点击监听
        DialogInterface.OnClickListener listener = null;
```


```java
        // TODO: SimRecords is not reset while SIM ABSENT (only reset while
        //       Radio_off_or_not_available). Have to reset in both both
        //       added or removed situation.
        listener = new DialogInterface.OnClickListener() {
            @Override
            public void onClick(DialogInterface dialog, int which) {
                synchronized (mLock) {
                    if (which == DialogInterface.BUTTON_POSITIVE) {
                        if (DBG) log("Reboot due to SIM swap");
                        //点击事件，点击重启手机
                        PowerManager pm = (PowerManager) mContext
                                .getSystemService(Context.POWER_SERVICE);
                        pm.reboot("SIM is added.");
                    }
                }
            }

        };

        Resources r = Resources.getSystem();

        String title = (isAdded) ? r.getString(R.string.sim_added_title) :
            r.getString(R.string.sim_removed_title);
        String message = (isAdded) ? r.getString(R.string.sim_added_message) :
            r.getString(R.string.sim_removed_message);
        String buttonTxt = r.getString(R.string.sim_restart_button);
        //弹出dialog，是否重启手机
        AlertDialog dialog = new AlertDialog.Builder(mContext)
        .setTitle(title)
        .setMessage(message)
        .setPositiveButton(buttonTxt, listener)
        .create();
        dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
        dialog.show();
    }
}
```
可以看出UiccCard在创建和更新的过程中，完成了以下的事情：

1. 创建和更新UiccCardApplication对象
2. 初始化了CatService
3. 监听SIM卡的插入和拔出，并弹出提示框询问是否重启手机

UiccCard本身并不实现具体的功能，只是作为间接接口向UiccController提供UiccCardApplication对象和完成CatService的创建工作，以及当SIM卡被插入或者拔出时弹出提示框是否需要重启设备。