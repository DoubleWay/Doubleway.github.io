---
layout:     post
title:      K560_MTK6761_Q_Go_Settings模块 wiko features(二)
subtitle:   
date:       2020-04-28
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - Setting模块
---

### 主屏幕设置（Home settings）

```java
<Preference
    android:key="homescreen_management"
    android:title="@string/wios_feature_display_home_screen_management_title"  
    android:icon="@drawable/ic_home_settings_wios">
    <intent
       android:action="android.intent.action.MAIN"
      android:targetPackage="com.wiko.launcherq"
      android:targetClass="com.wiko.settings.SettingsActivity" />
</Preference>
```

点击后通过intent，打开com.wiko.launcher.lightq（WikoLauncherQ_GO）应用，路径在

`vendor/tinno/myos/apps/WikoLauncherQ_GO`，在订单目录下决定是否添加

`vendor/tinno/product/k560/wik_fr/packages/apps/wiko_common.mk`

```java
 ApeSaleTrackerWiko \
    WikoDynomicPreload \
    WikoLauncherQ_GO \
```



### 指示灯（indicator lamp）

指示灯主要指是ApeHighLights这个应用，从配置文件可以看到它的targetClass是HintlightsActivity

```java
<Preference
    android:key="indicator_lamp"
   android:title="@string/wios_feature_display_indicator_lamp_title"
   android:icon="@drawable/ic_indicator_lamp_wios" >
              <intent android:targetPackage="com.tinno.hightlights"
               android:targetClass="com.tinno.hightlights.HintlightsActivity"/>
```

在HintlightsActivity做的事比较少，主要就是替换HintLightsSettings这个fragment。

```java
getFragmentManager().beginTransaction().replace(android.R.id.content,new HintLightsSettings()).commit();
```

`vendor/tinno/common/packages/apps/ApeHighLights/src/com/tinno/hightlights/HintLightsSettings.java`

```java
 // Send Broadcast where Received in NotifcationManagerService to update light status.
mHandler.post(new Runnable() {
    @Override
    public void run() {
        // TODO Auto-generated method stub
        Intent intent = new Intent("action.hintlights.SWITCH_CHANGED");
        intent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
        getActivity().sendBroadcastAsUser(intent, UserHandle.ALL);
        Log.i(TAG, "action.hintlights.SWITCH_CHANGED");
    }
});
```

这个类主要就是管理指示灯的界面，初始化CheckBoxPreference的状态，监听CheckBoxPreference的点击事件，然后发送intent为action.hintlights.SWITCH_CHANGED的广播，在服务里面更改light status。

HintLights的主要功能也是由express2.0来实现的，

`vendor/tinno/common/express20/modules/hintlights/src/main/java/express20/hintlights/LightsController.java`

```java
public static final String ACTION_HINTLIGHTS_SWITCH_CHANGED ="action.hintlights.SWITCH_CHANGED";
```

```java
else if (action.equals(ACTION_HINTLIGHTS_SWITCH_CHANGED)) {
    mUserSwitch = Settings.System.getInt(mContentResolver,  LIGHTS_SWITCH, 0x0000);
    updateLightsOnce();
}
```

```java
// Update hint light through calling a method of class Light which called
// method setLightLocked() in the end.
private void updateLightsOnce() {
    Utils.getMethodInvokeNoArgs(mHintLight, "turnOff");
    //return (obj == null) ? "" : (String)obj;
   // mHintLight.turnOff();
}
```

根据注释知道，通过Light最后会调用setLightLocked()方法，他们在：

`frameworks/base/services/core/java/com/android/server/lights`

但是因为express2.0，通过hideApi的方式调用，所以最后在

`vendor/tinno/common/express20/modules/hintlights/hideApi/src/express20/hide`路径下。

### （导航栏）navigation_bar_settings

```java
<Preference
    android:key="navigation_bar_settings"
   android:title="@string/nav_bar_title"
   android:icon="@drawable/ic_navigation_bar_wios" 
   >
</Preference>
```

`src/com/android/settings/wiosfeature/Utils.java`在这个类中，定义了这个Preference的字符串：

```java
public static final String KEY_WIOS_FEATURE_DISPLAY_NAV_BUTTON_SETTING = "navigation_bar_settings";
```

在WiosFeatureDashboardFragment的showDialogToDownLoadApp方法中：

```java
case Utils.KEY_WIOS_FEATURE_DISPLAY_NAV_BUTTON_SETTING:
       android.util.Log.e("laiyan","---->");
       //TINNO BEGIN, add for KFGAQWKA-387
       //Common.startNavSelect(getActivity());
       NavigationBarSettingsDialog.show(this);
       //TINNO END
       break;
```

在NavigationBarSettingsDialog.java中，监听了导航栏切换的点击事件

```java
public void onClick(View v) {
    resetRadio();
    int id = v.getId();
    int index = 0;
    switch (id) {
        case R.id.key_back_home_recent:
        case R.id.key_back_home_recent_rb:
            index = NAVIGATION_TYPE_1;
            mRadio1.setChecked(true);
            setNavBarDisplayType(index);
            Log.d(TAG, "R.id.key_back_home_recent");
            break;
        case R.id.key_recent_home_back:
        case R.id.key_recent_home_back_rb:
            index = NAVIGATION_TYPE_2;
            mRadio2.setChecked(true);
            setNavBarDisplayType(index);
            Log.d(TAG, "R.id.key_recent_home_back");
            break;
```

然后会通过ContentResolver将状态保存

```java
private void setNavBarDisplayType(int type) {
    Settings.System.putInt(getContext().getContentResolver(), NAV_BAR_DIAPLAY_TYPE, type);
}
```

可以通过`adb shell settings get system statusbar_nav_bar_type`来查看这个属性的值

