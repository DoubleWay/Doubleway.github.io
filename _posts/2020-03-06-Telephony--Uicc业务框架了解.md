```
---
layout:     post
title:      Telephony--Uicc业务框架了解
subtitle:   
date:       2020-03-06
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - Telephony
---
```

## Telephony--Uicc业务框架了解

手机使用的SIM卡统称UICC（Universal Integrated Circuit Card）有数据存储的能力（保存通讯录），并有唯一的ID（ICCID）都具备运营标识（MCC/MNC）MCC（Mobile country code）MNC（Mobile NetWork code），其本身也是一个文件系统，因为储存数据的位置不同分为不同的种类，通常分为SIM/USIM两种。

UICC系统主要涉及以下几个java类：

1. **UiccController.java** 

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/UiccController.java`

   设计为单例模式。监听RIL中的SIM卡状态，并把SIM卡状态的变化通知给其他类。在UICC框架中，它属于核心部分，除了分发SIM卡状态变化，还对外提供接口用于获取UiccCard，IccFileHandler，IccRecords，UiccCardApplication的对象。

2. **UiccCard.java**

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/UiccCard.java`

   它代表了具体的卡，一个UiccCard对象就表示了一张SIM卡（PhoneID）。SIM卡中的状态或者数据都可以在这里获取，比如，属性mCardState保存了SIM卡状态，mUniversalPinState保存了PIN码状态，mCatService代表了STK应用信息，mUiccApplications中包含了SIM卡的具体数据。。。等等，总结起来，UiccCard是SIM卡的大管家，它既代表了SIM卡，又控制了UiccApplications，CatService的生命周期。

3.  **UiccCardApplication.java**

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/UiccCardApplication.java`

   这是SIM卡应用（不是STK）。应该是对应了上面说到的逻辑模块，一张SIM卡可以有多个逻辑模块，也就有多个UiccCardApplication对象。它控制了IccRecords和IccFileHandler的生命周期。而IccRecords和

4.  **IccFileHandler.java**

    `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/IccFileHandler.java`

   读取SIM卡中（逻辑模块）的文件系统，也就是SIM卡中的具体数据。根据 UICC卡的种类不同，衍生了几个对应的子类SIMFileHandler，UsimFileHandler，RuimFileHandler，CsimFileHandler，IsimFileHandler

5. **IccRecords.java**

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/IccRecords.java`

   模板类，通过IccFileHandler来操作SIM卡中的文件系统个，获取并保存SIM中的具体数据，根据UICC卡的种类不同，衍生了几个对应的子类SIMRecords，RuimRecords，IsimUiccRecords。

6. **CatService.java**

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/cat/CatService.java`

   负责STK业务

7.   **IccCardProxy.java**

   `/frameworks/opt/telephony/src/java/com/android/internal/telephony/uicc/`

   封装了对UICC的一系列操作（状态和数据），并对外提供接口。一张卡（PhoneID）对应一个IccCardProxy对象。实际上，我们可以使用UiccController获取其他的对象从而实现对UICC的操作，但是需要传入一系列参数并保证卡状态正确，否则需要做判断处理，使用IccCardProxy操作只需要知道PhoneID。并发出ACTION_SIM_STATE_CHANGED广播，通知应用层以及没有监听UiccController得知SIM卡状态发生变化的其他类。

   

   ![Uicc整体框架](C:\Users\791112716\Downloads\Uicc整体框架.PNG)

   

