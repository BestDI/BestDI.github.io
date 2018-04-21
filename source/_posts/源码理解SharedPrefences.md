---
title: 源码理解SharedPrefences
date: 2018-04-21 20:16:57
tags:
categories:
---

### 一、前言

> `SharedPreferences`会从`shared_prefs`目录下的`<name>.xml`文件, 并将其以`get/put`的形式提供读写服务. 其中涉及到如下几个问题:
> 1. 如何从磁盘读取配置到内存
> 2. `getXxx`如何从内存中获取配置
> 3. 最终配置如何从内存回写到磁盘
> 4. 多线程/多进程是否会有问题
> 5. 以及最佳实践

<!-- more --> 

### 二、前言总结

> `SharedPreferences`是线程安全的. 内部由大量`synchronized`关键字保障,
> `SharedPreferences`不是进程安全的;
> 
> 初始化SP对象时，`getSharedPreferences`会读取磁盘文件, 后续的`getSharedPreferences`获取`SP`会从内存缓存中获取. 如果第一次调用`getSharedPreferences`时还没从磁盘加载完毕就调用`getXxx/putXxx`, 则`getXxx/putXxx`操作会卡主, 直到数据从磁盘加载完毕后返回
> 所有的`getXxx`都是从内存中取的数据；
> 
> `apply`是同步回写内存, 然后把异步回写磁盘的任务放到一个单线程的队列中等待调度.`commit`和前者一样, 只不过要等待异步磁盘任务结束后才返回;
> 
> `MODE_MULTI_PROCESS`是在每次`getSharedPreferences`时检查磁盘上配置文件上次修改时间和文件大小, 一旦所有修改则会重新从磁盘加载文件. 所以并不能保证多进程数据的实时同步,
> 从**Android N**开始, 不再支持`MODE_WORLD_READABLE & MODE_WORLD_WRITEABLE`一旦指定, 会抛异常。

### 三、最佳实践

> 不要多进程使用, 很小几率会造成数据全部丢失, 现象是配置文件被删除;
> 不要依赖`MODE_MULTI_PROCESS`. 这个标记就像`MODE_WORLD_READABLE/MODE_WORLD_WRITEABLE`未来会被废弃;
> 每次`apply / commit`都会把全部的数据一次性写入磁盘, 所以单个的配置文件不应该过大, 影响整体性能.

### 四、源码全面分析

#### 4.1 SharedPreferences 对象的获取

常见的获取方式如下：
```
PreferenceManager#getDefaultSharedPreferences()
ContextImpl#getSharedPreferences()
```

以上述`PreferenceManager#getDefaultSharedPreferences()`为例来看看源码:

```
// PreferenceManager.java
public static SharedPreferences getDefaultSharedPreferences(Context context) {
    return context.getSharedPreferences(getDefaultSharedPreferencesName(context),
            getDefaultSharedPreferencesMode());
}
```

其实上述的获取方式中, 最终都是调用到了`ContextImpl#getSharedPreferences()`源码:

```
// ContextImpl.java
public SharedPreferences getSharedPreferences(String name, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) { // 线程同步
        if (sSharedPrefs == null) {
            sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
        }
        final String packageName = getPackageName();
        ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
            sSharedPrefs.put(packageName, packagePrefs);
        }
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }
        // 从缓存中获取sp对象，sp是null时才会去创建
        sp = packagePrefs.get(name);
        if (sp == null) {
            File prefsFile = getSharedPrefsFile(name); // data/data/<packageName>/shared_prefs/<name>.xml
            sp = new SharedPreferencesImpl(prefsFile, mode);
            packagePrefs.put(name, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}
```

可见`Android SDK`是先取了缓存,如果缓存未命中,才构造对象.
也就是说, 多次`getSharedPreferences`几乎是没有代价的. 
同时, 实例的构造被`synchronized`关键字包裹, 因此构造过程是多线程安全的.

#### 4.2 SharedPreferences 的构造

```
// SharedPreferencesImpl.java
SharedPreferencesImpl(File file, int mode) {
    mFile = file; //代表我们磁盘上的配置文件
    mBackupFile = makeBackupFile(file); // 备份文件, 用户写入失败时进行恢复. 其路径是 mFile 加后缀 ‘.bak’
    mMode = mode;
    mLoaded = false;
    mMap = null; // 用于在内存中缓存我们的配置数据, 也就是 getXxx 数据的来源
    startLoadFromDisk();
}
```

进一步查看`startLoadFromDisk`

```
// SharedPreferencesImpl.java
private void startLoadFromDisk() {
    synchronized (this) {
        mLoaded = false; // 初始化结束的标记，在get/put中能看到
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            synchronized (SharedPreferencesImpl.this) {
                loadFromDiskLocked();
            }
        }
    }.start();
}
```

此处开启线程从文件读取, 其源码如下:

```
// SharedPreferencesImpl.java
private void loadFromDisk() {
    synchronized (SharedPreferencesImpl.this) {
        if (mLoaded) {
            return;
        }
        // 备份文件对原文件进行覆盖；
		// 读写之前创建，读写成功删除。
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    ... 略去无关代码 ...

    str = new BufferedInputStream(
            new FileInputStream(mFile), 16*1024);
    // 使用XMLPullParser的方式去解析文件
    map = XmlUtils.readMapXml(str);

    synchronized (SharedPreferencesImpl.this) {
        mLoaded = true;// 关键标记,修改状态
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<String, Object>();
        }
        // 唤醒等待的其他线程，例如在读写之前会存在一个等待加载的过程
        notifyAll();
    }
}
```

loadFromDisk 这个函数很关键. 它就是实际从磁盘读取配置文件的函数. 可见, 它做了如下几件事:
1. 如果有 ‘备份’ 文件, 则直接使用备份文件回滚覆盖原文件(读写之前创建，读写成果即删除的特性).
2. 把配置从磁盘读取到内存的并保存在`mMap`字段中(看代码最后 mMap = map)
3. 标记读取完成, 这个字段后面`awaitLoadedLocked`会用到. 记录读取文件的时间, 后面`MODE_MULTI_PROCESS`中会用到
4. `notifyAll`通知已经读取完毕, 激活所有等待加载的其他线程.

![](/images/2018_0421/sharedPreferences.png)

#### 4.3 getXxx 的流程

```
// SharedPreferencesImpl.java
public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

可见, 所有的`get`操作都是线程安全的. 并且`get`仅仅是从内存中`(mMap)`获取数据, 所以无性能问题.

考虑到 配置文件的加载 是在单独的线程中异步进行的(参考 ‘SharedPreferences 的构造’), 所以这里的 awaitLoadedLocked 是在等待配置文件加载完毕. 也就是说如果我们第一次构造 SharedPreferences 后就立刻调用 getXxx 方法, 很有可能读取配置文件的线程还未完成, 所以这里要等待该线程做完相应的加载工作. 来看看 awaitLoadedLocked 的源码:

```
// SharedPreferencesImpl.java
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            wait();
        } catch (InterruptedException unused) {
        }
    }
}
```

很明显, 如果加载还未完成(mLoaded == false), `getXxx`会卡在`awaitLoadedLocked`, 一旦加载配置文件的线程工作完毕, 则这个加载线程会通过`notifyAll`会通知所有在 awaitLoadedLocked 中等待的线程, getXxx 就能够返回了. 不过大部分情况下, mLoaded == true. 这样的话 awaitLoadedLocked 会直接返回

#### 4.4 putXxx 的流程

set 比 get 稍微麻烦一点儿, 因为涉及到 Editor 和 MemoryCommitResult 对象

先来看看 edit() 方法的实现:

```
// SharedPreferencesImpl.java
public Editor edit() {
    // TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    //      context.getSharedPreferences(..).edit().putString(..).apply()
    //
    // ... all without blocking.
    synchronized (this) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```

#### 4.5 Editor

Editor 没有构造函数, 只有两个属性被初始化:

```
// SharedPreferencesImpl.java
public final class EditorImpl implements Editor {
    private final Map<String, Object> mModified = Maps.newHashMap();
    private boolean mClear = false;

    ... 略去方法定义 ...
    public Editor putString(String key, @Nullable String value) { ... }
    public boolean commit() { ... }
    ...
}
```

mModified 是我们每次 putXxx 后所改变的配置项
mClear 标识要清空配置项, 但是只清了 SharedPreferences.mMap. 所以不要写这样的愚蠢代码:

```
sharedPreferences.edit()
        .putBoolean(&quot;foo&quot;, true)        // foo 无法被 clear 掉
        .clear()
        .putBoolean(&quot;bar&quot;, true)
        .commit()
```

edit() 会保障配置已从磁盘读取完毕, 然后仅仅创建了一个对象. 接下来看看 putXxx 的真身:

```
// SharedPreferencesImpl.java
public Editor putString(String key, @Nullable String value) {
    synchronized (this) {
        mModified.put(key, value);
        return this;
    }
}
```

很简单, 仅仅是把我们设置的配置项放到了 mModified 属性里保存. 等到 apply 或者 commit 的时候回写到内存和磁盘. 咱们分别来看看

#### 4.6 apply

apply 是各种 ‘最佳实践’ 推荐的方式, 那么它到底是怎么异步工作的呢? 我们来看个究竟:

```
// SharedPreferencesImpl.java
public void apply() {
    final MemoryCommitResult mcr = commitToMemory(); // 将修改放入内存中，包括放入mMap
    final Runnable awaitCommit = new Runnable() {
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };

    QueuedWork.add(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            public void run() {
                awaitCommit.run();
                QueuedWork.remove(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it&#039;s hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```

可以看出大致的脉络:
1. commitToMemory 应该是把修改的配置项回写到内存
2. QueuedWork.add(awaitCommit) 貌似没什么卵用
3. SharedPreferencesImpl.this.enqueueDiskWrite 把配置项加入到一个异步队列中, 等待调度

我们来看看 commitToMemory 的实现(略去大量无关代码):

```
// SharedPreferencesImpl.java
private MemoryCommitResult commitToMemory() {
    MemoryCommitResult mcr = new MemoryCommitResult();
    synchronized (SharedPreferencesImpl.this) {

        ... 略去无关 ...

        mcr.mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        synchronized (this) {
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // &quot;this&quot; is the magic value for a removal mutation. In addition,
                // setting a value to &quot;null&quot; for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    mMap.remove(k);
                } else {
                    mMap.put(k, v);
                }
            }

            mModified.clear();
        }
    }
    return mcr;
}
```

总结来说:
1. 把 Editor.mModified 中的配置项回写到 SharedPreferences.mMap 中, 完成了内存的同步;
2. 把 SharedPreferences.mMap 保存在了 mcr.mapToWriteToDisk 中. 而后者就是即将要回写到磁盘的数据源;

我们再来回头看看 apply 方法:

```
// SharedPreferencesImpl.java
public void apply() {
    final MemoryCommitResult mcr = commitToMemory();

    ... 略无关 ...

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
}
```

commitToMemory 完成了内存的同步回写
enqueueDiskWrite 完成了硬盘的异步回写, 我们接下来具体看看
enqueueDiskWrite

```
// SharedPreferencesImpl.java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                final Runnable postWriteRunnable) {
    final Runnable writeToDiskRunnable = new Runnable() {
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr);
                }

                ...
            }
        };

    ...

    QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
}
```

QueuedWork.singleThreadExecutor 实际上就是 ‘一个线程的线程池’, 如下:

```
// QueuedWork.java
public static ExecutorService singleThreadExecutor() {
    synchronized (QueuedWork.class) {
        if (sSingleThreadExecutor == null) {
            // TODO: can we give this single thread a thread name?
            sSingleThreadExecutor = Executors.newSingleThreadExecutor();
        }
        return sSingleThreadExecutor;
    }
}
```

回到 enqueueDiskWrite 中, 这里还有一个重要的函数叫做 writeToFile:

writeToFile

```
// SharedPreferencesImpl.java
private void writeToFile(MemoryCommitResult mcr) {
    // Rename the current file so it may be used as a backup during the next read
    if (mFile.exists()) {
        if (!mBackupFile.exists()) {
            if (!mFile.renameTo(mBackupFile)) {
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
        try {
            final StructStat stat = Os.stat(mFile.getPath());
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
        }
        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();
        return;
    }

    // Clean up an unsuccessfully written file
    mFile.delete();
}
```

代码大致分为三个过程:
1. 先把已存在的老的配置文件重命名(加 ‘.bak’ 后缀), 然后删除老的配置文件. 这相当于做了灾备
2. 向 mFile 中一次性写入所有配置项. 即 mcr.mapToWriteToDisk(这就是 commitToMemory 所说的保存了所有配置项的字段) 一次性写入到磁盘. 如果写入成功则删除灾备文件, 同时记录了这次同步的时间
3. 如果上述过程 [2] 失败, 则删除这个半成品的配置文件

好了, 我们来总结一下 apply:
1. 通过 commitToMemory 将修改的配置项同步回写到内存 SharedPreferences.mMap 中. 此时, 任何的 getXxx 都可以获取到最新数据了
2. 通过 enqueueDiskWrite 调用 writeToFile 将所有配置项一次性异步回写到磁盘. 这是一个单线程的线程池

![](/images/2018_0421/apply.png)

#### 4.7 commit

相对于apply的代码，commit的明了很多：


```
// SharedPreferencesImpl.java
public boolean commit() {
    MemoryCommitResult mcr = commitToMemory();
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

![](/images/2018_0421/commit.png)

apply/commit的异同可以看到 ‘等待异步任务返回’ 的线, 对比 apply 的时序图, 一眼就看出差别

`registerOnSharedPreferenceChangeListener`

最后需要提一下的就是 listener:
* 对于 apply, listener 回调时内存已经完成同步, 但是异步磁盘任务不保证是否完成
* 对于 commit, listener 回调时内存和磁盘都已经同步完毕


#### 4.8 各种标记的作用

* **MODE_PRIVATE/MODE_WORLD_READABLE/MODE_WORLD_WRITEABLE**:
指的是, 在保存文件的时候设置的文件属性. PRIVATE 就只有自己和所属组的读写权限, READABLE/WRITEABLE 是对 other 用户和组的读写权限. 主要源码位于: FileUtils.setPermissions

* **MODE_MULTI_PROCESS**:
阅读过本文的话你会知道, 一个 prefs 实例通常有两种获得途径, 一个是第一次被 new 创建出来的, 这种方式会实际的读取磁盘文件. 还一种是后续从缓存(sSharedPrefsCache) 中取出来了.
而这个标记的意思就是: 使用 getSharedPrefercences 获取实例时, 无论是从磁盘读文件构造对象还是从缓存获取, 都会检查实例的 ‘内存中保存的时间’ 和 ‘磁盘上文件的最后修改时间’, 如果内存中保存的时间和磁盘上文件的最后修改时间, 则重新加载文件. 可以认为如果实例是从磁盘读取构造出来的, 那么他的 ‘内存中保存的时间’ 和 ‘文件的最后修改时间’ 一定是一样的, 而从缓存中来的实例就不一样了, 因为它可能很早就被创建(那个时候就已经读取了磁盘的文件并记录了当时文件的最后修改时间), 在随后的期间里其他进程很可能修改过磁盘上的配置文件导致最后修改时间变化, 这时候当我们从缓存中再次获取这个实例的时候, 系统会帮你检查这个文件在这段时间是否被修改过(‘内存中保存的时间’ 和 ‘磁盘上文件的最后修改时间’ 是否一致), 如果被修改过, 则重新从磁盘读取配置文件, 保证获取实例的内容是最新的.
