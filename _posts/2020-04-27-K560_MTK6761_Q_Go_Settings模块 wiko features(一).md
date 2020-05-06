---
layout:     post
title:      K560_MTK6761_Q_Go_Settings模块 wiko features(一)
subtitle:   
date:       2020-04-27
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - Setting模块
---


## wiko features

设置里的wiko features主要就是一些wiko的需求功能。

从刚才的top_level_settings.xml资源文件可以看到，和wiko features相关的主要是WiosFeatureDashboardFragment和TopLevelWiosFeaturePreferenceController。

##### WiosFeatureDashboardFragment：

```java
@Override
protected int getPreferenceScreenResId() {
    return R.xml.wios_feature_fragment;
}
```

可以看到，wiko features菜单里面记载的布局文件是wios_feature_fragment.xml。

```java
<PreferenceScreen    
     xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:settings="http://schemas.android.com/apk/res-auto"
        android:title="@string/wios_feature_settings">

   <PreferenceCategory
       android:key="display_category"
      android:title="@string/wiko_feature_category_display">
      <!-- Single Hand Mode -->
      <Preference
          android:key="single_hand"
         android:title="@string/single_hand_settings_title"
         android:icon="@drawable/ic_one_handed_wios"
         android:fragment="com.tinno.SingleHandSettings"/>
      
                   。。。。。。。。。。。。。。。。。。。。。。。。。
      <!-- Home Screen Management -->
      <Preference
          android:key="homescreen_management"
          android:title="@string/wios_feature_display_home_screen_management_title"  
          android:icon="@drawable/ic_home_settings_wios">
          <intent
             android:action="android.intent.action.MAIN"
            android:targetPackage="com.wiko.launcherq"
            android:targetClass="com.wiko.settings.SettingsActivity" />
      </Preference>
         。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。
      <!-- Indicator Lamp -->
      <Preference
          android:key="indicator_lamp"
         android:title="@string/wios_feature_display_indicator_lamp_title"
         android:icon="@drawable/ic_indicator_lamp_wios" >
                  <intent android:targetPackage="com.tinno.hightlights"
                   android:targetClass="com.tinno.hightlights.HintlightsActivity"/>
      </Preference>
             。。。。。。。。。。。。。。。。。。。。。。。。。。。
   </PreferenceCategory>
</PreferenceScreen>
```

布局显示的就是wiko features的功能列表，如果你想去掉列表里面的某项功能，一种方法就是直接修改资源文件，直接屏蔽功能的preference。当然，这样修改是有风险的，如果是自己项目拉出的分支，不会影响到其他项目可以这样修改。

还有种方法就是可以增加宏控

```java
private static List<AbstractPreferenceController> buildPreferenceControllers(Context context, Lifecycle lifecycle) {
    final List<AbstractPreferenceController> controllers = new ArrayList<>();
    // Display
    controllers.add(new SingleHandController(context));
    //controllers.add(new SingleHandSwitchController(context,lifecycle));
    controllers.add(new AmbientDisplayController(
            context,
            new AmbientDisplayConfiguration(context)));
    // controllers.add(new AppZoomController(context));
    controllers.add(new HomeScreenManagementController(context));
    controllers.add(new HomeScreenManagementGoController(context));
    controllers.add(new GalleryLockScreenController(context));
    controllers.add(new IndicatorLampController(context));
    controllers.add(new NavButtonSettingController(context));
```

WiosFeatureDashboardFragment里面有一个buildPreferenceControllers方法，作用是获得各个Preference的Controller添加到controllers中，当然直接屏蔽controllers.add方法就想去掉是没有用的（我试过），一般是进入你想去掉的某项功能controller，比如IndicatorLampController：

```java
@Override
public boolean isAvailable() {
    return Utils.DISPLAY_INDICATOR_LAMP_SUPPORT;
}
```

里面的isAvailable（）方法就是控制perference的显示与否，一般修改就在这里加个宏控就行了：

```java
public boolean isAvailable() {
    if(SystemProperties.getBoolean("ro.feature.show_IndicatorLamp", true)) {
        return Utils.DISPLAY_INDICATOR_LAMP_SUPPORT;
    }
}
```

### 单手模式（one hand mode ）：

部分wiko的项目会要求移植和集成单手模式的功能，因为Q的项目，所以是以express2.0方式来实现的。

以K560_MTK6761_Q_Go项目为例，

##### 在自己的订单仓库下的configs文件李添加了一个功能开关，

```java
# TINNO BEGIN
#FEATURE_SINGLE_HAND hsg 20190904
FEATURE_SINGLE_HAND := yes
# TINNO END
```

这个开关的功能是是否启用wiko菜单里面的单手模式功能

vendor/tinno/common/external/SingleHand/configs.mk

```java
ifeq ($(strip $(FEATURE_SINGLE_HAND)), yes)
    PRODUCT_PRODUCT_PROPERTIES += ro.feature.singlehand=true 
    PRODUCT_PACKAGES += singlehand
endif
```

vendor/tinno/common/express20/modules/express20_interface/src/main/java/express20/Features.java

```java
public static final boolean MBA_FTR_SingleHand_REQC1206 = get("ro.feature.singlehand", false);
```

vendor/tinno/common/external/SingleHand/packages/apps/Settings/common/com/tinno/SingleHandController.java

```java
@Override
public boolean isAvailable() {
    return express20.Features.MBA_FTR_SingleHand_REQC1206;
}
```

##### 然后对单手模式需要的一些资源文件进行了overlay

`vendor/tinno/common/external/CustomReqDll/singlehand/`

`vendor/tinno/common/external/SingleHand`

**SingleHandSettings.java**：单手模式整体界面状态，生命周期等

**SingleHandController.java**：控制器，是否显示，Summary变化

**FasterAnimationsContainer**：动画效果

在`vendor/tinno/common/external/SingleHand`里面进行修改，在MtkSettings单独编译也能编到，因为在当前目录下的configs.mk文件中定义了

```java
TINNO_SINGLEHAND_PATH := vendor/tinno/common/external/SingleHand
BUILD_TINNO_SINGLEHAND := $(TINNO_SINGLEHAND_PATH)/common.mk
```

在MtkSettings的Android.mk中：

```java
# TINNO BEGIN

# FEATURE_SINGLE_HAND hsg 20190711

LOCAL_STATIC_JAVA_LIBRARIES += singlehand.hideapi
include $(BUILD_TINNO_SINGLEHAND)

# TINNO END
```

##### 在AndroidManifest.xml中添加定义的SingleHandSettings

MtkSettings里面的任何文件，activity或者fragment等都需要在AndroidManifest定义，还有看需不需要在**display_settings**里面添加，有些项目（除wiko）是在settings->display里面显示的。

看需求，需不需要将功能加入qs，如果需要的话

http://192.168.10.207/#/c/83693/

`vendor/mediatek/proprietary/packages/apps/SystemUI/res/values/config.xml`

```java
<!-- The default tiles to display in QuickSettings -->
    <string name="quick_settings_tiles_default" translatable="false">
        wifi,singlehand,bt,simdataconnection,dnd,flashlight,rotation,battery,cell,airplane,cast
    </string>
```

添加你需要添加的快捷开关的字符串，它是根据一个字符串来读取的。

`mediatek/proprietary/packages/apps/SystemUI/src/com/ape/systemui/qs/tiles/SinglehandTile.java`

添加功能的tile文件。

`vendor/mediatek/proprietary/packages/apps/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java`

```java
// TINNO BEGIN
            // FEATURE_SINGLE_HAND hsg 20190711
            case "singlehand":
                return new SinglehandTile(mHost);
            // TINNO END
```

增加需求功能的判断

##### 具体功能实现提取出来在express2.0中实现

`vendor/tinno/common/express20/modules/singlehand/src/main/java/express20/singlehand`