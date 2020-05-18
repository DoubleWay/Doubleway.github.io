---
layout:     post
title:      Android_device_unlock脚本及说明
subtitle:   
date:       2020-05-18
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - Unlock
---

## Android_device_unlock脚本及说明

### MTK_ANDROID_Q项目 unlock

 方法1：

```xml
Flash user/eng/userdebug full load and bootup to home screen
        
Settings -> System -> Developer options -> OEM unlocking
       
"adb reboot bootloader" or "press volume up key + power key and select fastboot"

fastboot flashing unlock (press volumn up key)
```

方法2

```
rebuild lk to unlock devices

goto lk (vendor/mediatek/proprietary/bootable/bootloader/lk)

add config into "project".mk (ex: project/k79v1_64_tee.mk)

MTK_BUILD_DEFAULT_UNLOCK = yes

rebuild lk 单独烧录lk  (lk需同一codebase编译生成)
```

### SPRD_ANDROID_Q项目 unlock 

#### 解锁设备

1. fastboot和signidentifier_unlockbootloader.sh赋予读写权限

2. 手机开发者选项打开OEM和userdebug

3. adb reboot bootloader 进入fastboot模式

4. sudo ./fastboot oem get_identifier_token   获得一串数字，如3139303135323634353530353232

5. ./signidentifier_unlockbootloader.sh 4a414a4e31393335303030353932 rsa4096_vbmeta.pem sign.bin  `如果失败，尝试替换本地目录下rsa4096_vbmeta.pem和signidentifier_unlockbootloader.sh文件`
   `/home/android/work/v800fr/vendor/sprd/proprietories-source/packimage_scripts/signimage/sprd/config/rsa4096_vbmeta.pem`

   `/home/android/work/v800fr/vendor/sprd/proprietories-source/packimage_scripts/signidentifier_unlockbootloader.sh`

6. sudo ./fastboot flashing unlock_bootloader sign.bin

#### 刷google镜刷机工具刷镜像

跑cts-on-gsi只需要GSI（如果fastboot和手机中不匹配，就用out目录下的fastboot）

##### 命令行刷镜像

- adb reboot bootloader
- sudo  ./fastboot reboot bootloader
- sudo  ./fastboot -S 20M flash system system.img
- sudo  ./fastboot reboot bootloader
- sudo  ./fastboot flashing lock(上锁)
- sudo  ./fastboot reboot;(手机会自动清楚数据，开机)

##### 刷机工具刷镜像

用ResearchDownload_R19.17.4301刷机工具,在main 设置中只选system和vbmeta分区,	system分区选对应的google_system.img ,VBMETA分区选项目软件包中vbmeta-gsi.img,    这个可以用out目录下的vbmeta-gsi.img 

### HMD设项目unlock 

- adb reboot bootloader

- fastboot flash unlock 0191022102822.bin （HMD的私钥对设备sn的签名.由HMD给出）

- 按音量键选择解锁, 然后按Power确认执行解锁.

  解锁需求
  1、将sn2发给HMD,拿到新的签名执行解锁;
  2、将sn2写为0191022102822,然后可以用 0191022102822.bin 执行解锁操作;
  (vendor/tinno/$project/trunk/0191022102822.bin)