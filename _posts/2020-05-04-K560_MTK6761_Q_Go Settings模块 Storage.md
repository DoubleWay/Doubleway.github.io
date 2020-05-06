---
layout:     post
title:      K560_MTK6761_Q_Go Settings模块 Storage
subtitle:   
date:       2020-05-04
author:     DoubleWay
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - Android
    - Setting模块
---

## Storage

从一级菜单的布局文件top_level_settings.xml文件可以看到：

```xml
<Preference
    android:key="top_level_storage"
    android:title="@string/storage_settings"
    android:summary="@string/summary_placeholder"
    android:icon="@drawable/ic_homepage_storage"
    android:order="-60"
    android:fragment="com.android.settings.deviceinfo.StorageSettings"
    settings:controller="com.android.settings.deviceinfo.TopLevelStoragePreferenceController"/>
```

storage指定的fragment是StorageSettings，指定的controller是TopLevelStoragePreferenceController。

在TopLevelStoragePreferenceController里面主要就是控制storage的可见否与更新summary，在Controller里面更新summary的方法里，通过线程工具在使用的storage发生改变后，会同步更新一级菜单里storage这个preference的summary：

```java
ThreadUtils.postOnBackgroundThread(() -> {
    final NumberFormat percentageFormat = NumberFormat.getPercentInstance();
    final PrivateStorageInfo info = PrivateStorageInfo.getPrivateStorageInfo(
            mStorageManagerVolumeProvider);
    final double privateUsedBytes = info.totalBytes - info.freeBytes;

    ThreadUtils.postOnMainThread(() -> {
        preference.setSummary(mContext.getString(R.string.storage_summary,
                percentageFormat.format(privateUsedBytes / info.totalBytes),
                Formatter.formatFileSize(mContext, info.freeBytes)));
    });
});
```

info.totalBytes指的是设备总的内存空间，info.freeBytes是设备还剩余的空间，privateUsedBytes是已经使用来的设备内存，percentageFormat.format(privateUsedBytes / info.totalBytes),就是设置一级菜单中storage的preference的summary显示已经使用了百分之多少。

StorageSettings主要就是“ 显示内部存储(包括内置存储和私有存储)的面板卷)和可移动存储(公共卷)。”就是设置各个部分使用的内存。

```java
if (mInternalCategory.getPreferenceCount() == 2
        && mExternalCategory.getPreferenceCount() == 0) {
    // Only showing primary internal storage, so just shortcut
    if (!mHasLaunchedPrivateVolumeSettings) {
        mHasLaunchedPrivateVolumeSettings = true;
        final Bundle args = new Bundle();
        args.putString(VolumeInfo.EXTRA_VOLUME_ID, VolumeInfo.ID_PRIVATE_INTERNAL);
        if(Utils.isMonkeyRunning()){
            Log.e(TAG, "Monkey test running so finishing storage manager dashboard settings");
        }
        else {
            new SubSettingLauncher(getActivity())
                .setDestination(StorageDashboardFragment.class.getName())
                .setArguments(args)
                .setTitleRes(R.string.storage_settings)
                .setSourceMetricsCategory(getMetricsCategory())
                .launch();
        }
        finish();
    }
```

从setDestination(StorageDashboardFragment.class.getName())可以了解到，它加载了StorageDashboardFragment.class这个类。

`vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/StorageDashboardFragment.java`

在StorageDashboardFragment.java中加载的xml文件是storage_dashboard_fragment.xml

```java
@Override
protected int getPreferenceScreenResId() {
    return R.xml.storage_dashboard_fragment;
}
```

`vendor/mediatek/proprietary/packages/apps/MtkSettings/res/xml/storage_dashboard_fragment.xml`

```xml
<com.android.settings.deviceinfo.StorageItemPreference
    android:key="pref_photos_videos"
    android:title="@string/storage_photos_videos"
    android:icon="@drawable/ic_photo_library"
    settings:allowDividerAbove="true"
    android:order="2" />
<com.android.settings.deviceinfo.StorageItemPreference
    android:key="pref_music_audio"
    android:title="@string/storage_music_audio"
    android:icon="@drawable/ic_media_stream"
    android:order="3" />
```

从资源文件可以看出，这个就是显示storage各个部分使用内存详细情况。storage里面详细情况的展示分为两个部分，一部分是上面的磁盘展示，另一部分是下面的item展示。

```java
protected int getPreferenceScreenResId() {
    return R.xml.storage_dashboard_fragment;
}

@Override
protected List<AbstractPreferenceController> createPreferenceControllers(Context context) {
    final List<AbstractPreferenceController> controllers = new ArrayList<>();
    mSummaryController = new StorageSummaryDonutPreferenceController(context);
    controllers.add(mSummaryController);
```

#### 上面部分磁盘展示的controller就是**StorageSummaryDonutPreferenceController**，

`vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/storage/StorageSummaryDonutPreferenceController.java`

```java
public void updateState(Preference preference) {
    super.updateState(preference);
    StorageSummaryDonutPreference summary = (StorageSummaryDonutPreference) preference;
    summary.setTitle(convertUsedBytesToFormattedText(mContext, mUsedBytes));
    summary.setSummary(mContext.getString(R.string.storage_volume_total,
            Formatter.formatShortFileSize(mContext, mTotalBytes)));
    summary.setPercent(mUsedBytesStorageSummaryDonutPreference, mTotalBytes);
    Log.d("StorageSummaryDonutPreferenceController","mUsedBytes: "+mUsedBytes+" mTotalBytes: "+mTotalBytes);
    summary.setEnabled(true);
}
```

updateState函数主要是更改storage磁盘上面的summary字符串显示，mUsedBytes是已用的内存，mTotalBytes是全部的内存。而在updateState方法中更改的summary，StorageSummaryDonutPreference就是布局的主要显示。

`vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/storage/StorageSummaryDonutPreference.java`

```java
setLayoutResource(R.layout.storage_summary_donut);
```

加载的布局文件是**storage_summary_donut.xml**

```java
public void setPercent(long usedBytes, long totalBytes) {
    if (totalBytes == 0) {
        return;
    }

    mPercent = usedBytes / (double) totalBytes;
}

@Override
public void onBindViewHolder(PreferenceViewHolder view) {
    super.onBindViewHolder(view); donut.setPercentage(mPercent);
    view.itemView.setClickable(false);

    final DonutView donut = (DonutView) view.findViewById(R.id.donut);
    if (donut != null) {
        donut.setPercentage(mPercent);
    }

    final Button deletionHelperButton = (Button) view.findViewById(R.id.deletion_helper_button);
    if (deletionHelperButton != null) {
        deletionHelperButton.setOnClickListener(this);
    }
}
```

通过setPercent（）来计算内存空间占用的百分比，然后通过onBindViewHolder（）方法来加载布局， donut.setPercentage(mPercent)设置显示的百分比。

#### 然后是下面的item部分的显示

```java
StorageManager sm = context.getSystemService(StorageManager.class);
mPreferenceController = new StorageItemPreferenceController(context, this,
        mVolume, new StorageManagerVolumeProvider(sm));
controllers.add(mPreferenceController);
```

item部分的controller可以看到是StorageItemPreferenceController。

`vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/deviceinfo/storage/StorageItemPreferenceController.java`

```java
private StorageItemPreference mPhotoPreference;
private StorageItemPreference mAudioPreference;
private StorageItemPreference mGamePreference;
private StorageItemPreference mMoviesPreference;
private StorageItemPreference mAppPreference;
private StorageItemPreference mFilePreference;
private StorageItemPreference mSystemPreference;
```

可以看到，初始化了item就是StorageItemPreference，

```java
@Override
public void displayPreference(PreferenceScreen screen) {
    mScreen = screen;
    Log.d("doubleway","mScreen getKey : "+mScreen.getKey());
    mPhotoPreference = screen.findPreference(PHOTO_KEY);
    mAudioPreference = screen.findPreference(AUDIO_KEY);
    mGamePreference = screen.findPreference(GAME_KEY);
    mMoviesPreference = screen.findPreference(MOVIES_KEY);
    mAppPreference = screen.findPreference(OTHER_APPS_KEY);
    mSystemPreference = screen.findPreference(SYSTEM_KEY);
    mFilePreference = screen.findPreference(FILES_KEY);

    setFilesPreferenceVisibility();
}
```

然后通过findPreference方法加载不同的Key来显示item，mScreen是/storage_dashboard_fragment.xml。

还监听了item的点击事件;

```java
@Override
public boolean handlePreferenceTreeClick(Preference preference) {
    if (preference == null) {
        return false;
    }

    Intent intent = null;
    if (preference.getKey() == null) {
        return false;
    }
    switch (preference.getKey()) {
        case PHOTO_KEY:
            Log.d("doubleway","photo onclick");
            intent = getPhotosIntent();
            break;
        case AUDIO_KEY:
            intent = getAudioIntent();
```

```java
if (intent != null) {
    intent.putExtra(Intent.EXTRA_USER_ID, mUserId);

    launchIntent(intent);
    return true;
}
```

捕获到点击事件以后，获取item 的intent，然后加载itent。

以getPhotosIntent为例，来看他是怎样获得intent的。

```java
private Intent getPhotosIntent() {
    Bundle args = getWorkAnnotatedBundle(2);
    args.putString(
            ManageApplications.EXTRA_CLASSNAME, Settings.PhotosStorageActivity.class.getName());
    args.putInt(
            ManageApplications.EXTRA_STORAGE_TYPE,
            ManageApplications.STORAGE_TYPE_PHOTOS_VIDEOS);
    return new SubSettingLauncher(mContext)
            .setDestination(ManageApplications.class.getName())
            .setTitleRes(R.string.storage_photos_videos)
            .setArguments(args)
            .setSourceMetricsCategory(mMetricsFeatureProvider.getMetricsCategory(mFragment))
            .toIntent();
}
```

可以看到PhotosStorageActivity这个类是一个Settings的内部类，并没有具体实现。它setDestination的是ManageApplications.class。

```java
if (mListType == LIST_TYPE_OVERLAY && !Utils.isSystemAlertWindowEnabled(getContext())) {
    mRootView = inflater.inflate(R.layout.manage_applications_apps_unsupported, null);
    setHasOptionsMenu(false);
    return mRootView;
}

mRootView = inflater.inflate(R.layout.manage_applications_apps, null);
```

```java
mRecyclerView = mListContainer.findViewById(R.id.apps_list);
```

可以看到ManageApplications里面加载的布局根据判断加的是manage_applications_apps_unsupported或者是manage_applications_apps，然后下面的app列表是一个RecyclerView，apps_list。
