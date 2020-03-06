---
layout:     post
title:      TelePhony--IccFileHandler
subtitle:   
date:       2020-03-06
author:     DoubleWay
header-img: img/post-bg-alibaba.jpg
catalog: 	 true
tags:
    - Android
    - Telephony
---

## TelePhony--IccFileHandler

通过之前我们可以知道，UiccCardApplicattion在构造方法和update方法中会根据需要创建IccFileHandler的各种子类：

```java
    mIccFh = createIccFileHandler(as.app_type);
    mIccRecords = createIccRecords(as.app_type, mContext, mCi);
    if (mAppState == AppState.APPSTATE_READY) {
        queryFdn();
        queryPin1State();
    }
    mCi.registerForNotAvailable(mHandler, EVENT_RADIO_UNAVAILABLE, null);
}
```
会调用createIccFileHandler()方法来进行创建：

```java
private IccFileHandler createIccFileHandler(AppType type) {
        switch (type) {
            case APPTYPE_SIM:
                /* SPRD: Add for bug693265, AndroidO porting for USIM/SIM phonebook @{ */
                //return new SIMFileHandler(this, mAid, mCi);
                return TeleFrameworkFactory.getInstance().createSIMFileHandler(this, mAid, mCi);
                /* @} */
            case APPTYPE_RUIM:
                return new RuimFileHandler(this, mAid, mCi);
            case APPTYPE_USIM:
                /* SPRD: Add for bug693265, AndroidO porting for USIM/SIM phonebook @{ */
                //return new UsimFileHandler(this, mAid, mCi);
                return TeleFrameworkFactory.getInstance().createUsimFileHandler(this, mAid, mCi);
                /* @} */
            case APPTYPE_CSIM:
                return new CsimFileHandler(this, mAid, mCi);
            case APPTYPE_ISIM:
                return new IsimFileHandler(this, mAid, mCi);
            default:
                return null;
        }
    }
```

可以看到会传入不同的type，然后根据不同的type来创建不同的IccFileHandler子类，他们都继承自IccFileHandler，不同的type也代表了不同的SIM卡，如RUIM/USIM/CSIM等，他们主要的不同就是其内部的数据储存分区不同，因为数据储存分区有差别所以在读取数据时的操作还是有所区别，所以要创建不同的FileHandler

### IccFileHandler的功能

因为他们都继承自IccFileHandler，所以先来看看IccFileHandler的public方法：

`\frameworks\opt\telephony\src\java\com\android\internal\telephony\uicc\IccFileHandler.java` 

```java
public void loadEFLinearFixed(int fileid, String path, int recordNum, Message onLoaded)
public void loadEFLinearFixed(int fileid, int recordNum, Message onLoaded)
public void loadEFImgLinearFixed(int recordNum, Message onLoaded)
public void getEFLinearRecordSize(int fileid, String path, Message onLoaded)
public void loadEFLinearFixedAll(int fileid, String path, Message onLoaded)
public void loadEFTransparent(int fileid, Message onLoaded)
public void loadEFImgTransparent(int fileid, int highOffset, int lowOffset,
   int length, Message onLoaded)
public void updateEFLinearFixed(int fileid, String path, int recordNum, byte[] data,String pin2, Message onComplete)    
```

从这些方法可以看出IccFileHandler的主要功能是提供对SIM卡文件系统的读写操作，当调用这些方法时要传入读写数据的地址，以及读写完成后的回调函数，IccFileHandler会在读取完成后通知调用者，并传递一个返回值。例如

```java
   public void loadEFLinearFixed(int fileid, String path, int recordNum, Message onLoaded) {
        String efPath = (path == null) ? getEFPath(fileid) : path;
       //读取Records的size
        Message response
                = obtainMessage(EVENT_GET_RECORD_SIZE_DONE,
                        new LoadLinearFixedContext(fileid, recordNum, efPath, onLoaded));
      //调用RILJ向Modem读取SIM卡当前分区的长度
        mCi.iccIOForApp(COMMAND_GET_RESPONSE, fileid, efPath,
                        0, 0, GET_RESPONSE_EF_SIZE_BYTES, null, null, mAid, response);
    }
```
在读取的过程中并不是直接读取相应的内容，而是先读取当前记录分区的长度，知道长度之后才去读取当前记录的内容，在发送信息以后：
    public void handleMessage(Message msg) {
        AsyncResult ar;
        IccIoResult result;
        Message response = null;
        String str;
        LoadLinearFixedContext lc;

```java
    byte data[];
    int size;
    int fileid;
    int recordSize[];
    String path = null;

    try {
        switch (msg.what) {
        .................................................
         case EVENT_GET_RECORD_SIZE_IMG_DONE:
         case EVENT_GET_RECORD_SIZE_DONE:
             //得到从modem拿到的原始数据
            ar = (AsyncResult)msg.obj;
            lc = (LoadLinearFixedContext) ar.userObj;
            result = (IccIoResult) ar.result;
            response = lc.mOnLoaded;

            if (processException(response, (AsyncResult) msg.obj)) {
                loge("exception caught from EVENT_GET_RECORD_SIZE");
                break;
            }

            data = result.payload;
            path = lc.mPath;

            if (TYPE_EF != data[RESPONSE_DATA_FILE_TYPE]) {
                throw new IccFileTypeMismatch();
            }

            if (EF_TYPE_LINEAR_FIXED != data[RESPONSE_DATA_STRUCTURE]) {
                throw new IccFileTypeMismatch();
            }
            //得到当前记录的长度
            lc.mRecordSize = data[RESPONSE_DATA_RECORD_LENGTH] & 0xFF;

            size = ((data[RESPONSE_DATA_FILE_SIZE_1] & 0xff) << 8)
                   + (data[RESPONSE_DATA_FILE_SIZE_2] & 0xff);

            lc.mCountRecords = size / lc.mRecordSize;

             if (lc.mLoadAll) {
                 lc.results = new ArrayList<byte[]>(lc.mCountRecords);
             }

             if (path == null) {
                 path = getEFPath(lc.mEfid);
             }
              //加上要读取的记录长度再次读取数据
             mCi.iccIOForApp(COMMAND_READ_RECORD, lc.mEfid, path,
                     lc.mRecordNum,
                     READ_RECORD_MODE_ABSOLUTE,
                     lc.mRecordSize, null, null, mAid,
                     obtainMessage(EVENT_READ_RECORD_DONE, lc));
             break;
```
在他获取到了当前记录的长度后，才会再次调用iccIOForApp，真正去读取数据，发送EVENT_READ_RECORD_DONE信息：

```java
  public void handleMessage(Message msg) {
               ......................
            case EVENT_READ_IMG_DONE:
            case EVENT_READ_RECORD_DONE:

                ar = (AsyncResult)msg.obj;
                lc = (LoadLinearFixedContext) ar.userObj;
                result = (IccIoResult) ar.result;
                response = lc.mOnLoaded;
                path = lc.mPath;

                if (processException(response, (AsyncResult) msg.obj)) {
                    break;
                }

                if (!lc.mLoadAll) {
                    sendResult(response, result.payload, null);
                } else {
                    lc.results.add(result.payload);

                    lc.mRecordNum++;
                      //循环读取记录
                    if (lc.mRecordNum > lc.mCountRecords) {
                        sendResult(response, lc.results, null);
                    } else {
                        if (path == null) {
                            path = getEFPath(lc.mEfid);
                        }

                        mCi.iccIOForApp(COMMAND_READ_RECORD, lc.mEfid, path,
                                    lc.mRecordNum,
                                    READ_RECORD_MODE_ABSOLUTE,
                                    lc.mRecordSize, null, null, mAid,
                                    obtainMessage(EVENT_READ_RECORD_DONE, lc));
                    }
                }

            break;
                ...................
        }
```
它的各个子类都继承自IccFileHandler，因为不同的SIM卡的区别就是储存信息的分区不同，所以他们唯一的不同就是重写了getEFPath(）方法，如CsimFileHandler的getEFPath方法：

```java
protected String getEFPath(int efid) {
        switch(efid) {
        case EF_SMS:
        case EF_CST:
        case EF_FDN:
        case EF_MSISDN:
        case EF_RUIM_SPN:
        case EF_CSIM_LI:
        case EF_CSIM_MDN:
        case EF_CSIM_IMSIM:
        case EF_CSIM_CDMAHOME:
        case EF_CSIM_EPRL:
        case EF_CSIM_MIPUPP:
            return MF_SIM + DF_ADF;
        }
        String path = getCommonIccEFPath(efid);
        if (path == null) {
            // The EFids in UICC phone book entries are decided by the card manufacturer.
            // So if we don't match any of the cases above and if its a UICC return
            // the global 3g phone book path.
            return MF_SIM + DF_TELECOM + DF_PHONEBOOK;
        }
        return path;
    }
```

