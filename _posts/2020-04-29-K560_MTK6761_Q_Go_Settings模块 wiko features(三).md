### 项目：K560_MTK6761_Q_Go

##### 修改设置-->Network&internet-->SIM cards

路径：

/home/android/work2/k560_wik_fr/android/vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/sim/SimSettings.java

##### 手机界面，kernal version中的时区信息CST要改为WIB 

修改：

/home/android/work2/k560_wik_fr/android/vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/KernelVersionPreferenceController.java

/home/android/work2/k560_wik_fr/android/vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/firmwareversion/KernelVersionPreferenceController.java