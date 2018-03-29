---
title: Android反编译APK
date: 2017-12-25 22:23:59
tags: [Android]
categories: [Android]
---

**反编译可以理解为逆向工程（Reverse Engineering）**
通过反编译APK，可以更好的理解APK打包过程，可以验证特性和动态替换资源。使用工具[ClassyShark](https://github.com/google/android-classyshark)

## Base

### APK(Application Package)

**APK实质上为压缩包，可以直接解压，解压后可以获得的信息**
> * AndroidManifest.xml：清单文件
> * classes.dex：Dex格式编译文件，classes压缩包
> * res：不需要编译的文件，一般都是系统资源文件
> * assets：AssetManager
> * META-INF：Jar包元数据，也包含应用签名

<!-- more --> 

**安装应用**

> adb install -r *.apk
> -r 表示强制安装，覆盖当前版本

**查看手机中的所有应用信息**

> adb shell pm list packages -f

**导出手机中的apk(root 手机)**

> adb pull -p /data/app/me.chunyu.Pedometer-1/base.apk base.apk

### AAPT(Android Assets Packaging Tool)

**Android 打包工具**
> 在Android sdk的build-tools文件夹中

**获取apk信息**
> * aapt list base.apk // 内容
> * aapt dump badging base.apk // 属性
> * aapt dump permissions base.apk // 权限
> * aapt dump resources base.apk // 资源

**获取二进制xml信息**
> aapt dump xmltree base.apk AndroidManifest.xml

## 反编译

### dex2jar

**[dex2jar](https://github.com/pxb1988/dex2jar)是dex转换jar的工具，还需要使用Java包解析工具[JD-GUI](http://jd.benow.ca/)**
> 使用步骤：
> 1. apk解压后，可以获得classes.dex(有可能不止一个)
> 2. 使用命令**d2j-dex2jar classes.dex**进行格式转换，得到**classes-dex2jar.jar**
> 3. 得到的jar可以使用**JD-GUI**查看


### apktool

**进行资源的反编译，[ApkTool Download Page](https://ibotpeaches.github.io/Apktool/install/)**
> 使用步骤：
> 1. apktool d base.apk
> 2. 执行完成后，就会生成包含资源文件的test文件夹：
> |.
> | original
> | res
> | smali
> | AndroidManifest.xml
> | apktool.yml





## 重打包

**1. 既然已经顺利进行了反编译，修改之后，可以使用命令进行重新打包**

> ```cmd
apktool b test -o new_test.apk
// 执行完成之后，会生成一个new_test.apk```

**2. 重新打包之后的apk需要重新签名**
> ```
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore 签名文件名 -storepass 签名密码 待签名的APK文件名 签名的别名
// 其中jarsigner命令文件是存放在jdk的bin目录下的，需要将bin目录配置在系统的环境变量中才可以在任何位置执行此命令```

**3. 签名完成之后，需要对APK进行一次对齐操作**

> ```
zipalign 4 new_test.apk new_test_aligned.apk
// 对齐操作使用的是zipalign工具，该工具在<Android SDK>/build-tools/<version>目录下```

**4. 验证apk签名是否成功**

> ``` jarsigner -verify -verbose -certs new_test_aligned.apk ```