---
title: ScreenLogger了解一下
date: 2018-07-24 22:44:23
tags: [Android]
categories: [Android]
---


### ScreenLogger -- 一个简单可用的屏幕日志插件[![](https://jitpack.io/v/BestDI/ScreenLogger.svg)](https://jitpack.io/#BestDI/ScreenLogger)

#### How to Use it?

1. Add it in your root build.gradle at the end of repositories:

```groovy
allprojects {
    repositories {
        ...
        maven { url 'https://jitpack.io' }
    }
}
```

Add the dependency
```groovy
dependencies {
    implementation 'com.github.BestDI:ScreenLogger:v1.0'
}
```

<!-- more -->

2. init in your application:
```kotlin
override fun onCreate() {
    super.onCreate()

    ProcessLifecycleOwner.get().lifecycle.apply {
        val isEnable = true // 可以通过isEnable的状态控制是否打开ScreenLog
        addObserver(LoggerLifecycleObserver(this@MainApplication, isEnable))
    }
}
```

now, you could use it:
```kotlin
Logger.warn("TAG", "hello from button")
```

or you could use:
`typealias ScreenLogger = Logger`

#### use result

1. request permission(above android 6.0)
![](/images/2018_0724/permission.png)

2. tracking log
![](/images/2018_0724/track.png)

3. log details
![](/images/2018_0724/logdetails.png)

> 使用Kotlin编写的屏幕Log，帮助开发定位API，脱离IDE看日志的痛点。