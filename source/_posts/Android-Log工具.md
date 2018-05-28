---
title: Android Log工具类
date: 2018-05-04 22:25:03
tags: [Android]
categories: [Android]
---

> **MainLog.java**,使用方法为 `private MainLog LOG = new MainLog(clazz)`

<!-- more --> 

```java
import org.slf4j.helpers.MarkerIgnoringBase;

public class MainLog extends MarkerIgnoringBase {

    private final static int LOG_ERROR = 1;
    private final static int LOG_WARN = 2;
    private final static int LOG_INFO = 3;
    private final static int LOG_DEBUG = 4;
    private final static int LOG_TRACE = 5;

    /**
     * 可以在Build.gradle文件中区分debug、release、beta的log level
     */
    private static int LOG_LEVEL = BuildConfig.LOG_LEVEL;
    private org.slf4j.Logger log = null;

    private MainLog() {

    }

    public MainLog(Class<?> t) {

        if (t == null) {
            throw new IllegalArgumentException("owner must not be null");
        }
        this.log = org.slf4j.LoggerFactory.getLogger(t);
    }


    @Override
    public void debug(String format, Object param1, Object param2) {

        if (!this.isDebugEnabled()) {
            return;
        }
        log.debug(format, param1, param2);
    }

    @Override
    public void debug(String format, Object arg1) {

        if (!this.isDebugEnabled()) {
            return;
        }
        log.debug(format, arg1);
    }

    @Override
    public void debug(String format, Object[] argArray) {

        if (!this.isDebugEnabled()) {
            return;
        }
        log.debug(format, argArray);

        StringBuilder sb = new StringBuilder();
        for (Object o : argArray) {
            sb.append(((o == null) ? null : o.toString()));
            sb.append("\n");
        }
    }

    @Override
    public void debug(String msg, Throwable t) {

        if (!this.isDebugEnabled()) {
            return;
        }
        log.debug(msg, t);

        StringBuilder sb = new StringBuilder();
        if (t != null) {
            for (StackTraceElement element : t.getStackTrace()) {
                sb.append(((element == null) ? null : element.toString()));
                sb.append("\n");
            }
        }
    }

    @Override
    public void debug(String msg) {

        if (!this.isDebugEnabled()) {
            return;
        }
        log.debug(msg);
    }

    @Override
    public void error(String format, Object arg1, Object arg2) {

        if (!this.isErrorEnabled()) {
            return;
        }
        log.error(format, arg1, arg2);
    }

    @Override
    public void error(String format, Object arg) {

        if (!this.isErrorEnabled()) {
            return;
        }
        log.error(format, arg);
    }

    @Override
    public void error(String format, Object[] argArray) {

        if (!this.isErrorEnabled()) {
            return;
        }
        log.error(format, argArray);

        StringBuilder sb = new StringBuilder();
        for (Object o : argArray) {
            sb.append(((o == null) ? null : o.toString()));
            sb.append("\n");
        }
    }

    @Override
    public void error(String msg, Throwable t) {

        if (!this.isErrorEnabled()) {
            return;
        }
        log.error(msg, t);

        StringBuilder sb = new StringBuilder();
        if (t != null) {
            for (StackTraceElement element : t.getStackTrace()) {
                sb.append(((element == null) ? null : element.toString()));
                sb.append("\n");
            }
        }
    }

    @Override
    public void error(String msg) {

        if (!this.isErrorEnabled()) {
            return;
        }
        log.error(msg);
    }

    @Override
    public void info(String format, Object arg1, Object arg2) {

        if (!this.isInfoEnabled()) {
            return;
        }
        log.info(format, arg1, arg2);
    }

    @Override
    public void info(String format, Object arg) {

        if (!this.isInfoEnabled()) {
            return;
        }
        log.info(format, arg);
    }

    @Override
    public void info(String format, Object[] argArray) {

        if (!this.isInfoEnabled()) {
            return;
        }
        log.info(format, argArray);

        StringBuilder sb = new StringBuilder();
        for (Object o : argArray) {
            sb.append(((o == null) ? null : o.toString()));
            sb.append("\n");
        }
    }

    @Override
    public void info(String msg, Throwable t) {

        if (!this.isInfoEnabled()) {
            return;
        }
        log.info(msg, t);

        StringBuilder sb = new StringBuilder();
        if (t != null) {
            for (StackTraceElement element : t.getStackTrace()) {
                sb.append(((element == null) ? null : element.toString()));
                sb.append("\n");
            }
        }
    }

    @Override
    public void info(String msg) {

        if (!this.isInfoEnabled()) {
            return;
        }
        log.info(msg);
    }

    @Override
    public void trace(String format, Object param1, Object param2) {

        if (!this.isTraceEnabled()) {
            return;
        }
        log.trace(format, param1, param2);
    }

    @Override
    public void trace(String format, Object param1) {

        if (!this.isTraceEnabled()) {
            return;
        }
        log.trace(format, param1);
    }

    @Override
    public void trace(String format, Object[] argArray) {

        if (!this.isTraceEnabled()) {
            return;
        }
        log.trace(format, argArray);

        StringBuilder sb = new StringBuilder();
        for (Object o : argArray) {
            sb.append(((o == null) ? null : o.toString()));
            sb.append("\n");
        }
    }

    @Override
    public void trace(String msg, Throwable t) {

        if (!this.isTraceEnabled()) {
            return;
        }
        log.trace(msg, t);

        StringBuilder sb = new StringBuilder();
        if (t != null) {
            for (StackTraceElement element : t.getStackTrace()) {
                sb.append(((element == null) ? null : element.toString()));
                sb.append("\n");
            }
        }
    }

    @Override
    public void trace(String msg) {

        if (!this.isTraceEnabled()) {
            return;
        }
        log.trace(msg);
    }

    @Override
    public void warn(String format, Object arg1, Object arg2) {

        if (!this.isWarnEnabled()) {
            return;
        }
        log.warn(format, arg1, arg2);
    }

    @Override
    public void warn(String format, Object arg) {

        if (!this.isWarnEnabled()) {
            return;
        }
        log.warn(format, arg);
    }

    @Override
    public void warn(String format, Object[] argArray) {

        if (!this.isWarnEnabled()) {
            return;
        }
        log.warn(format, argArray);

        StringBuilder sb = new StringBuilder();
        for (Object o : argArray) {
            sb.append(((o == null) ? null : o.toString()));
            sb.append("\n");
        }
    }

    @Override
    public void warn(String msg, Throwable t) {

        if (!this.isWarnEnabled()) {
            return;
        }
        log.warn(msg, t);

        StringBuilder sb = new StringBuilder();
        if (t != null) {
            for (StackTraceElement element : t.getStackTrace()) {
                sb.append(((element == null) ? null : element.toString()));
                sb.append("\n");
            }
        }
    }

    @Override
    public void warn(String msg) {

        if (!this.isWarnEnabled()) {
            return;
        }
        log.warn(msg);
    }

    @Override
    public boolean isDebugEnabled() {

        return LOG_LEVEL >= LOG_DEBUG;
    }

    @Override
    public boolean isErrorEnabled() {

        return LOG_LEVEL >= LOG_ERROR;
    }

    @Override
    public boolean isInfoEnabled() {

        return LOG_LEVEL >= LOG_INFO;
    }

    @Override
    public boolean isTraceEnabled() {

        return LOG_LEVEL >= LOG_TRACE;
    }

    @Override
    public boolean isWarnEnabled() {

        return LOG_LEVEL >= LOG_WARN;
    }
}
```