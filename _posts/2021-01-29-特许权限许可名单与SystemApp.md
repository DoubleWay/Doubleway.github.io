---
layout:     post
title:      特许权限许可名单与SystemApp
subtitle:   
date:       2021-01-29
author:     DoubleWay
header-img: img/post-sample-image.jpg
catalog: 	 true
tags:
    - Android
    - apk
---

# 特许权限许可名单

------

客户提了一个需求，预制一个launcher apk到系统应用，不可删除，且应用能够通过google play store升级。

最开始选择就预制到system/app目录下面，这个比较简单，在应用目录下的**Android.mk**文件中通过 **LOCAL_PRIVILEGED_MODULE := true**来进行控制，如果不设置或者设置为**false**，安装位置为**system/app**；如果设置为**true**，安装位置为**system/priv-app**。

应用要能够通过应用商店进行升级，这个就必须要使用应用自身的签名 ，因为如果前后两个程序版本所采用的签名不同，即使包名相同，也不会被视为同一个程序的不同版本，不能覆盖安装，**LOCAL_CERTIFICATE := PRESIGNED**，表示使用应用本身的签名，表示之前应用已经签名了。Google apk就是这样

按理来说这样已经就满足客户提出的需求了，但是客户又提了新的需求，给了我们一个链接，https://source.android.google.cn//devices/tech/config/perms-allowlist，希望我们能按照这个参考文档里的方式预置应用，进入链接可以看到，这就是官方关于特许权限应用许可的指导文档。需要预置到**system/priv-app**下面。

这就引出了我一直以来的一个疑惑，**system/app**和**system/priv-app**到底有什么区别

要知道两者的区别，首先了解一下什么是**system app**

------



## 什么是**system app**

在PackageManagerService中，对是否是system app是这样判断的：

```java
private static boolean isSystemApp(PackageParser.Package pkg) {
        return (pkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0;
    }
```

```java
 private static boolean isSystemApp(PackageSetting ps) {
        return (ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0;
    }
```

从代码可以看到，具有**ApplicationInfo.FLAG_SYSTEM**标志的，就被视为**system app**

### 一般那些app是**system app**呢

- 特定的shared uid的app都属于system app，例如：android.uid.system`，`android.uid.phone`，`android.uid.log`，`android.uid.nfc`，`android.uid.bluetooth`，`android.uid.shell等，这些app都被赋予了**ApplicationInfo.FLAG_SYSTEM**标志,在PackageManagerService中

  ```java
  mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                  ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
          mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                  ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        ................................................
          mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                  ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
  ```

- 特定目录下的app，例如`/vendor/overlay`，`/system/framework`，`/system/priv-app`，`/system/app`，`/vendor/app`，`/oem/app`下面的应用，这个在PackageManagerService中也有相关的定义：

  ```java
  // Collect vendor/product overlay packages. (Do this before scanning any apps.)
              // For security and version matching reason, only consider
              // overlay packages if they reside in the right directory.
              scanDirTracedLI(new File(VENDOR_OVERLAY_DIR),
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM
                      | SCAN_AS_VENDOR,
                      0);  // /vendor/overlay
              scanDirTracedLI(new File(PRODUCT_OVERLAY_DIR),
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM
                      | SCAN_AS_PRODUCT,
                      0);  // /product/overlay
               // Collect privileged system packages.
              final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
              scanDirTracedLI(privilegedAppDir,
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM
                      | SCAN_AS_PRIVILEGED,
                      0); // /system/priv-app
  .........................
             // Collect ordinary system packages.
              final File systemAppDir = new File(Environment.getRootDirectory(), "app");
              scanDirTracedLI(systemAppDir,
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM,
                      0); //system/app
  // Collect all OEM packages.
              final File oemAppDir = new File(Environment.getOemDirectory(), "app");
              scanDirTracedLI(oemAppDir,
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM
                      | SCAN_AS_OEM,
                      0); // /oem/app
  ......................
  ```

  函数中的**PackageParser.PARSE_IS_SYSTEM**最终会被转换成**ApplicationInfo.FLAG_SYSTEM**

  ------

  

  ## 什么是特权app

  那什么又叫特权app呢，特权app 也叫**privileged app**，能够使用特许权限的应用，才能叫特权应用，就和文章开头的特许权限许可联系起来了。从之前的链接文档可以知道

  特权应用是位于系统映像某个分区上 `priv-app` 目录下的系统应用。各 Android 版本中，该分区为：

  - Android 8.1 及更低版本 - `/system`
  - Android 9 及更高版本 - `/system, /product, /vendor`

  从这个路径可以知道，特权app首先必须是System app。也就是说 System app分为**普通的system app**和**特权的system app**。`/system/app`和`/system/priv-app`区别就是，`/system/app`下面的是**普通的system app**，而`/system/priv-app`下面是特权app，他能够声明获得更多的权限。

从Android 8.0开始，特权应用如果使用**系统**的特许权限，那么需要把这个特许权限加入到白名单中，这个就叫特许权限许可名单。

### 系统的特许权限

什么是系统的特许权限呢，系统的特许权限必须在**`frameworks/base/core/res/AndroidManifest.xml`定义**，并且等级为**`signature|privileged`**

```xml
<permission android:name="android.permission.ACCESS_IMS_CALL_SERVICE"
        android:label="@string/permlab_accessImsCallService"
        android:description="@string/permdesc_accessImsCallService"
        android:protectionLevel="signature|privileged" />
 <permission android:name="android.permission.ACCESS_UCE_PRESENCE_SERVICE"
        android:permissionGroup="android.permission-group.PHONE"
        android:protectionLevel="signature|privileged"/>
.......................
```

如果应用使用了这些权限，就需要配置权限白名单，如果没有配置，那么设备将无法启动。而像normal和dangerous级别的权限，这些权限应用需要去访问其对应受保护的资源时只需要在androidManifest.xml中添加相同的uses-permission就行了。

------



## 如何配置特许权限许可名单

如何配置特许权限许可名单呢，这个在官方指导文档里面也说明了：

- 对于已包含在 Android 开源项目 (AOSP) 树中的应用，请将其权限列在 `/etc/permissions/privapp-permissions-platform.xml` 中。
- 对于 Google 应用，请将其权限列在 `/etc/permissions/privapp-permissions-google.xml` 中。
- 对于其他应用，请使用以下格式的文件：`/etc/permissions/privapp-permissions-DEVICE_NAME.xml`

`privapp-permissions.xml` 文件只有在与特权应用位于同一分区时才能授予或拒绝授予该应用权限。例如，如果 `/vendor` 分区上的应用请求特许权限，则只能由同样位于 `/vendor` 上的 `privapp-permissions.xml` 文件来同意或拒绝该请求。

而像我们将应用预置到system下面的话，就要将`privapp-permissions.xml`文件放到system分区下面。就在自己的订单仓库下面配置就行，例如我们配置的就叫**`privapp-permissions-ObaBox.xml`**然后在配置文件中添加需要的权限：

```xml
<!--
    This XML file declares which signature|privileged permissions should be
    granted to privileged apps that come with the platform
    -->
 <permissions>
<privapp-permissions package="com.android.xxxxxxx"> <!-- 应用的包名 -->
    <permission name="android.permission.BACKUP"/>
    <permission name="android.permission.CRYPT_KEEPER"/>
    <!-- don't allow application to interact across users -->
    <deny-permission name="android.permission.INTERACT_ACROSS_USERS"/>
    <permission name="android.permission.MANAGE_USERS"/>
    <permission name="android.permission.MODIFY_PHONE_STATE"/>
    <permission name="android.permission.READ_PRIVILEGED_PHONE_STATE"/>
    <permission name="android.permission.RECEIVE_EMERGENCY_BROADCAST"/>
</privapp-permissions>
</permissions>  
  ..........
```

详细的配置可以参考谷歌的官方文档：https://source.android.google.cn//devices/tech/config/perms-allowlist