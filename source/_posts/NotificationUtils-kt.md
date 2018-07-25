---
title: NotificationUtils.kt
date: 2018-07-25 22:36:05
tags: [android,kotlin]
categories: [android,kotlin]
---

> 用于App是否打开通知选项，并且可跳转至相关Setting页面进行设置

```kotlin
import android.app.AppOpsManager
import android.app.NotificationManager
import android.content.Context
import android.content.Intent
import android.net.Uri
import android.os.Build
import java.lang.reflect.InvocationTargetException

// NotificationUtils is a class that could judge notification status or not , 
// and could enableNotification
object NotificationUtils {

    private const val CHECK_OP_NO_THROW = "checkOpNoThrow"
    private const val OP_POST_NOTIFICATION = "OP_POST_NOTIFICATION"

    @JvmStatic
    fun enableNotification(context: Context) =
            Intent().apply {
                when {
                    Build.VERSION.SDK_INT >= Build.VERSION_CODES.O -> {
                        //android 8.0引导
                        action = "android.settings.APP_NOTIFICATION_SETTINGS"
                        putExtra("android.provider.extra.APP_PACKAGE", context.packageName)
                    }
                    Build.VERSION.SDK_INT in Build.VERSION_CODES.LOLLIPOP..Build.VERSION_CODES.N_MR1 -> {
                        //android 5.0-7.0
                        action = "android.settings.APP_NOTIFICATION_SETTINGS"
                        putExtra("app_package", context.packageName)
                        putExtra("app_uid", context.applicationInfo.uid)
                    }
                    Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP -> {
                        action = "android.settings.APPLICATION_DETAILS_SETTINGS"
                        data = Uri.fromParts("package", context.packageName, null)
                    }
                }
            }

    @JvmStatic
    fun areNotificationsEnabled(context: Context) = with(context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            areNotificationsEnabled()
        } else {
            isNotificationEnabled(context)
        }
    }

    private fun isNotificationEnabled(context: Context): Boolean {

        val mAppOps = context.getSystemService(Context.APP_OPS_SERVICE) as AppOpsManager
        val appInfo = context.applicationInfo
        val pkg = context.applicationContext.packageName
        val uid = appInfo.uid

        val appOpsClass: Class<*>? /* Context.APP_OPS_MANAGER */

        try {
            appOpsClass = Class.forName(AppOpsManager::class.java.name)
            val checkOpNoThrowMethod = appOpsClass!!.getMethod(CHECK_OP_NO_THROW, Integer.TYPE, Integer.TYPE, String::class.java)
            val opPostNotificationValue = appOpsClass.getDeclaredField(OP_POST_NOTIFICATION)
            val value = opPostNotificationValue.get(Int::class.java) as Int

            return checkOpNoThrowMethod.invoke(mAppOps, value, uid, pkg) as Int == AppOpsManager.MODE_ALLOWED
        } catch (e: ClassNotFoundException) {
            e.printStackTrace()
        } catch (e: NoSuchMethodException) {
            e.printStackTrace()
        } catch (e: NoSuchFieldException) {
            e.printStackTrace()
        } catch (e: InvocationTargetException) {
            e.printStackTrace()
        } catch (e: IllegalAccessException) {
            e.printStackTrace()
        }

        return false
    }
}
```