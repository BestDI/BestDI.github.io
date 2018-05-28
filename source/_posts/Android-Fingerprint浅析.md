---
title: Android Fingerprint浅析
date: 2018-04-21 23:24:00
tags: [Android]
categories: [Android]
---

### 一，Fingerprint总体架构图

Fingerprint表层模块架构如下，只涉及Application,Framework,Frigerprintd以及FingerprintHal，不包含指纹的认证驱动；

![Fp_communication](/images/2018_0423/fp_communication.png)

<!-- more --> 

### 二，Fingerprint Framework初始化流程

2.1. 在Zygote初始化进程中，会启动各种Service，包括FingerprintServe；
`FingerprintService`的启动在SystemServer.java的`startOtherService()`过程中：

```java
/**
 * API23
 * Starts a miscellaneous grab bag of stuff that has yet to be refactored
 * and organized.
 */
private void startOtherServices() {
	// disableNonCoreServices -> SystemProperties.getBoolean("config.disable_noncore", false);
	// 配置项，是否加载非核心服务
    if (!disableNonCoreServices) {
        try {
            Slog.i(TAG, "Media Router Service");
            mediaRouter = new MediaRouterService(context);
            ServiceManager.addService(Context.MEDIA_ROUTER_SERVICE, mediaRouter);
        } catch (Throwable e) {
            reportWtf("starting MediaRouterService", e);
        }

        mSystemServiceManager.startService(TrustManagerService.class);

        // 启动FingerprintService
        mSystemServiceManager.startService(FingerprintService.class);

        try {
            Slog.i(TAG, "BackgroundDexOptService");
            BackgroundDexOptService.schedule(context, 0);
        } catch (Throwable e) {
            reportWtf("starting BackgroundDexOptService", e);
        }

    }
    ......
}

/* Android N */
private void startOtherService() {
	// 启动FingerprintService
	// FEATURE_FINGERPRINT = "android.hardware.fingerprint"
	// mPackageManager.hasSystemFeature(<fileName>) <- mAvailableFeatures(ArryayList) <- SystemConfig去加载etc目录下的配置。
	if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_FINGERPRINT)) {
    traceBeginAndSlog("StartFingerprintSensor");
    mSystemServiceManager.startService(FingerprintService.class);
    traceEnd();
	}
}
	
```

2.2. 在Android N版本以上，加入了SystemFeature的判断，如果需要支持的话，需要在`framework/native/data/etc`目录下添加`android.hardware.fingerprint.xml`来支持该功能，mPackageManager.hasSystemFeature(<fileName>)是最终通过获取配置来判断是否具有此功能。

```xml
<!-- This is the standard set of features for a biometric fingerprint sensor. -->
<permissions>
    <feature name="android.hardware.fingerprint" />
</permissions>
```

2.3. 启动FingerprintService，并添加到ServiceManager中，

![](/images/2018_0423/fp_service2.png)

> 将FingerprintService添加到ServiceManager中后，在SystemServiceRegistry.java中静态代码块中注册服务的时候，可以从ServiceManager中获取FingerprintService的Binder对象，从而可以构造出FingerprintManager对象，这样app端就可以通过Context来获取FingerprintManager对象。 

![](/images/2018_0423/fp_service1.png)

这样，app端通过Context获取FingerprintManager，通过调用FingerprintManager的接口来实现相应的功能，FingerprintManager转调FingerprintService中方法，FingerprintService负责管理整个注册，识别、删除指纹、检查权限等流程的逻辑，FingerprintService调用fingerprintd的接口，通过fingerprintd和FingerprintHal层进行通信。

在FingerprintService的getFingerprintDaemon方法中有如下步骤：  
1. 获取fingerprintd  
2. 向fingerprintd注册回调函数mDaemonCallback  
3. 调用获取fingerprintd的openhal函数  
4. 建立fingerprint文件系统节点，设置节点访问权限，调用fingerprintd的setActiveGroup，将路径传下去。此路径一半用来存储指纹模板的图片等

```java
public IFingerprintDaemon getFingerprintDaemon() {
    if (mDaemon == null) {
        // .1 获取fingerprintd
        mDaemon = IFingerprintDaemon.Stub.asInterface(ServiceManager.getService(FINGERPRINTD));
        if (mDaemon != null) {
            try {
                mDaemon.asBinder().linkToDeath(this, 0);
                // .2 向fingerprintd注册回调函数mDaemonCallback
                mDaemon.init(mDaemonCallback);
                // .3 调用获取fingerprintd的openhal函数
                mHalDeviceId = mDaemon.openHal();
                /* .4 建立fingerprint文件系统节点，设置节点访问权限，
                调用fingerprintd的setActiveGroup，
                将路径传下去。此路径一半用来存储指纹模板的图片等*/
                if (mHalDeviceId != 0) {
                    updateActiveGroup(ActivityManager.getCurrentUser());
                } else {
                    Slog.w(TAG, "Failed to open Fingerprint HAL!");
                    mDaemon = null;
                }
            } catch (RemoteException e) {
                Slog.e(TAG, "Failed to open fingeprintd HAL", e);
                mDaemon = null; // try again later!
            }
        } else {
            Slog.w(TAG, "fingerprint service not available");
        }
    }
    return mDaemon;
}
```

1. FingerprintService在framework模块负责指纹的大部分逻辑，FingerprintService会在开机的时候初始化；  
1. application调用framework通过FingerprintManager接口即可实现；  
1. framework中FingerManager和FingerprintService的通信使用Binder机制实现，表现即使用aidl这个接口定义语言实现  
1. framework调往fingerprintd的同样属于Binder通信，两者分属于不同的进程。不过这部分跟java层Binder处理有点不一样，是java层往native层的调用。

### 三，Fingerprintd初始化

![](/images/2018_0423/fingerprintd_1.png)

 fingerprintd如果划分的比较细的话，可以分为四个部分：
 
1. `fingerprintd.cpp`负责将fingerprintd加入到ServiceManager中，以便FingerprintService能够获取
2. `IFingerprintDaemon.h/IFingerprintDaemon.cpp`负责java层到fingerprintd的Binder通信
3. `FingerprintDaemonProxy.h/FingerprintDaemonProxy.cpp`负责fingerprintd和Fignerprint hal层的通信
4. `IFingerprintDaemonCallback.h/IFingerprintDaemonCallback.cpp`负责将指纹的回调结果传给java层

fingerprintd在init.rc有相应的开机启动脚本，所以一开机就会跑它的main函数。fingerprintd作为一个独立的进程运行，负责将Framework和Hal层的通信连接起来。

![](/images/2018_0423/fpd_main.png)

fingerprintd 的main函数就是将fingerprintd添加到servicemanager中管理。然后开了一个线程，等待binder消息。

### 四，IFingerprintDaemon是如何跟framework通信的

```C
#ifndef IFINGERPRINT_DAEMON_H_
#define IFINGERPRINT_DAEMON_H_

#include <binder/IInterface.h>
#include <binder/Parcel.h>

namespace android {

class IFingerprintDaemonCallback;

/*
* Abstract base class for native implementation of FingerprintService.
*
* Note: This must be kept manually in sync with IFingerprintDaemon.aidl
*/
class IFingerprintDaemon : public IInterface, public IBinder::DeathRecipient {
    public:
        enum {
           AUTHENTICATE = IBinder::FIRST_CALL_TRANSACTION + 0,
           CANCEL_AUTHENTICATION = IBinder::FIRST_CALL_TRANSACTION + 1,
           ENROLL = IBinder::FIRST_CALL_TRANSACTION + 2,
           CANCEL_ENROLLMENT = IBinder::FIRST_CALL_TRANSACTION + 3,
           PRE_ENROLL = IBinder::FIRST_CALL_TRANSACTION + 4,
           REMOVE = IBinder::FIRST_CALL_TRANSACTION + 5,
           GET_AUTHENTICATOR_ID = IBinder::FIRST_CALL_TRANSACTION + 6,
           SET_ACTIVE_GROUP = IBinder::FIRST_CALL_TRANSACTION + 7,
           OPEN_HAL = IBinder::FIRST_CALL_TRANSACTION + 8,
           CLOSE_HAL = IBinder::FIRST_CALL_TRANSACTION + 9,
           INIT = IBinder::FIRST_CALL_TRANSACTION + 10,
           POST_ENROLL = IBinder::FIRST_CALL_TRANSACTION + 11,
           ENUMERATE = IBinder::FIRST_CALL_TRANSACTION + 12,
        };

        IFingerprintDaemon() { }
        virtual ~IFingerprintDaemon() { }
        virtual const android::String16& getInterfaceDescriptor() const;

        // Binder interface methods
        virtual void init(const sp<IFingerprintDaemonCallback>& callback) = 0;
        virtual int32_t enroll(const uint8_t* token, ssize_t tokenLength, int32_t groupId,
                int32_t timeout) = 0;
        virtual uint64_t preEnroll() = 0;
        virtual int32_t postEnroll() = 0;
        virtual int32_t stopEnrollment() = 0;
        virtual int32_t authenticate(uint64_t sessionId, uint32_t groupId) = 0;
        virtual int32_t stopAuthentication() = 0;
        virtual int32_t remove(int32_t fingerId, int32_t groupId) = 0;
        virtual int32_t enumerate() = 0;
        virtual uint64_t getAuthenticatorId() = 0;
        virtual int32_t setActiveGroup(int32_t groupId, const uint8_t* path, ssize_t pathLen) = 0;
        virtual int64_t openHal() = 0;
        virtual int32_t closeHal() = 0;

        // DECLARE_META_INTERFACE - C++ client interface not needed
        static const android::String16 descriptor;
        static void hal_notify_callback(const fingerprint_msg_t *msg);
};

// ----------------------------------------------------------------------------

class BnFingerprintDaemon: public BnInterface<IFingerprintDaemon> {
    public:
       virtual status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply,
               uint32_t flags = 0);
    private:
       bool checkPermission(const String16& permission);
};

} // namespace android

#endif // IFINGERPRINT_DAEMON_H_
```

java层到fingerprintd的通信这里同样是采用binder方式，注意到上面IFingerprintDaemon.h NOTE，需要手动保证IFingerprintDaemon.h文件与IFingerprintDaemon.aidl文件一致，由于java层aidl文件编译时会自动编译成IFingerprintDaemon.java文件。

当添加接口来调用指纹底层暴露的接口，在IFingerprintDaemon.h文件中添加类似上面35行到68行的枚举，枚举的值需要与java层aidl自动生成的java文件中的枚举保持一致。另外还需要在上面68行处加上描述这些接口的纯虚函数（c++中的纯虚函数类似java的抽象方法，用于定义接口的规范，在C++中，一个具有纯虚函数的基类被称为抽象类）。
如下面截图对比，我们发现IFingerprintDaemon.cpp和java层aidl生成的IFingerprintDaemon.java在onTransact是基本一致的。这样我们也就明白了为什么上面说需要手动和IFingerprintDaemon.aidl保持同步了，这样方式类似我们平时在三方应用使用aidl文件，需要保持client端和server端aidl文件一致。

> 可以看到onTransact有四个参数  
> code ， data ，replay ， flags  
> code 是一个整形的唯一标识，用于区分执行哪个方法，客户端会传递此参数，告诉服务端执行哪个方法  
> data客户端传递过来的参数  
> replay服务器返回回去的值  
> flags标明是否有返回值，0为有（双向），1为没有（单向）

![](/images/2018_0423/IFp_Daemon.png)

IFingerprintDaemon.aidl文件生成的IFingerprintDaemon.java文件 

![](/images/2018_0423/IFp_java.png)

### 五，fingerprintd进程是如何和Fingerprint Hal层是如何传递数据的

> 说到Hal层，即硬件抽象层，Android系统为HAL层中的模块接口定义了规范，所有工作于HAL的模块必须按照这个规范来编写模块接口，否则将无法正常访问硬件。 

![](/images/2018_0423/hal.png)

指纹的HAL层规范fingerprint.h在/hardware/libhardware/include/hardware/下可以看到。
我们注意到在fingerprint.h中定义了两个结构体，分别是fingerprint_device_t和fingerprint_module_t,如下图。

![](/images/2018_0423/fp_h.png)

fingerprint_device_t结构体，用于描述指纹硬件设备；fingerprint_module_t结构体，用于描述指纹硬件模块。在FingerprintDaemonProxy.cpp就是通过拿到fingerprint_device_t这个结构体来和Fingerprint HAL层通信的。
当需要添加接口调用指纹底层时，在这个fingerprint.h中同样需要添加函数指针，然后通过FingerprintDaemonProxy.cpp中拿到这个fingerprint_device_t来调用fingerprint.h中定义的函数指针，也就相当于调用指纹HAL层。
我们重点看一下它的openHal（）函数。

![](/images/2018_0423/fp_openHalNew.png)

openHal的方法这里主要看上面三个部分： 
1. 根据名称获取指纹hal层模块。hw_module这个一般由指纹芯片厂商根据 fingerprint.h实现，hw_get_module是由HAL框架提供的一个公用的函数，这个函数的主要功能是根据模块ID(module_id)去查找注册在当前系统中与id对应的硬件对象，然后载入(load)其相应的HAL层驱动模块的*so文件。 
2. 调用fingerprint_module_t的open函数 
3. 向hal层注册消息回调函数，主要回调 注册指纹进度，识别结果，错误信息等等 
4. 判断向hal层注册消息回调是否注册成功

目前关于指纹的上层流程大致就到这儿，指纹底层就不怎么介绍了，术业有专攻，和专业做指纹的还是有不少差距。

### 六，FingerprintManager相关阅读

FingerprintManager中，具有一个Handler，用来分发Error/Help Msg，  
其中，`IFingerprintServiceReceiver`用来接收FingerprintManagerService等从native层面识别成功的结果，  
然后通过mHandler对象，进行结果的分发。

```
/**
 * Communication channel from the FingerprintService back to FingerprintManager.
 * @hide
 */
oneway interface IFingerprintServiceReceiver {
}
```

![](/images/2018_0423/fingerprintserviceReceiver.png)