---
title: AndFix原理及源码解析
date: 2018-04-01 23:10:15
tags: [Android,三方框架]
categories: [Android,三方框架]
---

#### 一、概述

> [**AndFix**](https://github.com/alibaba/AndFix) 是阿里系的一个轻量级热修复，热更新框架，只支持简单的method修改，没有res修复等臃肿功能。具体的实现是通过native进行原函数的指针替换，具有一次修复，持续有效的特性（当然是不清除app所有数据前提下）。`AndFix has a native method art_replaceMethod in ART or dalvik_replaceMethod in Dalvik.`

![方法替换原理](/images/2018_0401/principle.png)

<!-- more --> 

#### 二、使用Demo，及步骤列举

> AndFix的具体修复流程正如官方文档所描述的，是基于相同版本进行的修复，而AndFix所提供的apkpatch.bat是分析和对比，通过指定对应的apk、keystore生成.patch文件（热修复所需要的，可以通过网络路径存放等等方式）

![Alt text](/images/2018_0401/process.png)

##### 1. 使用demo

> 1. 初始化PatchManager
> ```
patchManager = new PatchManager(context);
patchManager.init(appversion);//current version
> ```
> 2. 加载patch文件
> ```
patchManager.loadPatch();// You should load patch as early as possible, generally, in the initialization phase of your application(such as Application.onCreate()).
> ```
> 3. Add patch,
> ``` 
patchManager.addPatch(path);//path of the patch file that was downloaded
> ```

##### 2. 步骤列举 

> 1. .patch文件分析，使用apkpatch.bat生成的目录如下：
> ```
.
├── 1.txt
├── diff.dex
├── out.apatch
└── smali
    └── com
        └── tong
            └── android_hot_fix
                └── CalcUtils_CF.smali
> ```
> 2. 其中，.patch文件其实是个压缩包，其中包含的内容如下：
> ```
> .
├── classes.dex // 具体的修复class
└── META-INF
    ├── CERT.RSA // 签名整数
    ├── CERT.SF // 证书相关INFO
    ├── MANIFEST.MF // Manifest相关
    └── PATCH.MF // patch生成相关信息
> ```
> 3. dex->jar classes.dex之后，可以看到生成的class文件中，会有`@MethodReplace(clazz="com.tong.android_hot_fix.CalcUtils", method="calc")` 这种注解，注解的作用是在method加载的时候，使用此标记去判断需要替换的方法，深层次是用natvie层的C进行指针映射到修复后函数。

#### 三、源码解析

1、 初始化PatchManager对象
```
public PatchManager(Context context) {
    mContext = context;
    mAndFixManager = new AndFixManager(mContext);
    // mPatchDir是.patch的复制保存路径，AndFix是从此目录去查找.patch，此目录为../packageName/file/aptch
    mPatchDir = new File(mContext.getFilesDir(), DIR);
    mPatchs = new ConcurrentSkipListSet<Patch>();
    mLoaders = new ConcurrentHashMap<String, ClassLoader>();
}
```

初始化PatchManager过程中，初始化了AndFixManager，
```
public AndFixManager(Context context) {
	mContext = context;
	mSupport = Compat.isSupport();// 去判断当前设备是否支持热修复功能
	if (mSupport) {
		// SecurityChecker主要做.patch签名验证等功能
		mSecurityChecker = new SecurityChecker(mContext);
		mOptDir = new File(mContext.getFilesDir(), DIR);
		if (!mOptDir.exists() && !mOptDir.mkdirs()) {// make directory fail
			mSupport = false;
			Log.e(TAG, "opt dir create error.");
		} else if (!mOptDir.isDirectory()) {// not directory
			mOptDir.delete();
			mSupport = false;
		}
```

2、 `patchManager.loadPatch()`加载.patch文件
```
/**
 * load patch,call when application start
 */
public void loadPatch() {
    mLoaders.put("*", mContext.getClassLoader());// wildcard
    Set<String> patchNames;
    List<String> classes;
    for (Patch patch : mPatchs) {
        patchNames = patch.getPatchNames();
        for (String patchName : patchNames) {
            classes = patch.getClasses(patchName);
            // 调用AndFixManager.fix()
            mAndFixManager.fix(patch.getFile(), mContext.getClassLoader(),
                    classes);
        }
    }
}
```
可以跟随看到AndFixManager的方法实现：
```
/**
 * fix
 * 
 * @param file
 *            patch file
 * @param classLoader
 *            classloader of class that will be fixed
 * @param classes
 *            classes will be fixed
 */
public synchronized void fix(File file, ClassLoader classLoader,
		List<String> classes) {
	if (!mSupport) {
		return;
	}
	// 进行Security验证
	if (!mSecurityChecker.verifyApk(file)) {// security check fail
		return;
	}
	try {
		// 1.从file/patch目录下读取.patch文件
		File optfile = new File(mOptDir, file.getName());
		boolean saveFingerprint = true;
		if (optfile.exists()) {
			// need to verify fingerprint when the optimize file exist,
			// prevent someone attack on jailbreak device with
			// Vulnerability-Parasyte.
			// btw:exaggerated android Vulnerability-Parasyte
			// http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
			if (mSecurityChecker.verifyOpt(optfile)) {
				saveFingerprint = false;
			} else if (!optfile.delete()) {
				return;
			}
		}
		// 加载其压缩包中的.dex文件
		final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
				optfile.getAbsolutePath(), Context.MODE_PRIVATE);
		if (saveFingerprint) {
			mSecurityChecker.saveOptSig(optfile);
		}
		ClassLoader patchClassLoader = new ClassLoader(classLoader) {
			@Override
			protected Class<?> findClass(String className)
					throws ClassNotFoundException {
				Class<?> clazz = dexFile.loadClass(className, this);
				if (clazz == null
						&& className.startsWith("com.alipay.euler.andfix")) {
					return Class.forName(className);// annotation’s class
													// not found
				}
				if (clazz == null) {
					throw new ClassNotFoundException(className);
				}
				return clazz;
			}
		};
		Enumeration<String> entrys = dexFile.entries();
		Class<?> clazz = null;
		while (entrys.hasMoreElements()) {
			String entry = entrys.nextElement();
			if (classes != null && !classes.contains(entry)) {
				continue;// skip, not need fix
			}
			clazz = dexFile.loadClass(entry, patchClassLoader);
			if (clazz != null) {
				// fixClass
				fixClass(clazz, classLoader);
			}
		}
	} catch (IOException e) {
		Log.e(TAG, "pacth", e);
	}
}

/**
 * fix class
 * 
 * @param clazz
 *            class
 */
private void fixClass(Class<?> clazz, ClassLoader classLoader) {
	Method[] methods = clazz.getDeclaredMethods();
	// 方法注解，表明需要替换的方法
	MethodReplace methodReplace;
	String clz;
	String meth;
	for (Method method : methods) {
		methodReplace = method.getAnnotation(MethodReplace.class);
		if (methodReplace == null)
			continue;
		// 从注解获取class和需要替换的方法名
		clz = methodReplace.clazz();
		meth = methodReplace.method();
		if (!isEmpty(clz) && !isEmpty(meth)) {
			replaceMethod(classLoader, clz, meth, method);
		}
	}
}

/**
 * replace method
 * 
 * @param classLoader classloader
 * @param clz class
 * @param meth name of target method 
 * @param method source method
 */
private void replaceMethod(ClassLoader classLoader, String clz,
		String meth, Method method) {
	try {
		String key = clz + "@" + classLoader.toString();
		Class<?> clazz = mFixedClass.get(key);
		if (clazz == null) {// class not load
			Class<?> clzz = classLoader.loadClass(clz);
			// initialize target class
			clazz = AndFix.initTargetClass(clzz);
		}
		if (clazz != null) {// initialize class OK
			mFixedClass.put(key, clazz);
			Method src = clazz.getDeclaredMethod(meth,
					method.getParameterTypes());
			AndFix.addReplaceMethod(src, method);
		}
	} catch (Exception e) {
		Log.e(TAG, "replaceMethod", e);
	}
}

public static void addReplaceMethod(Method src, Method dest) {
	try {
		replaceMethod(src, dest);
		initFields(dest.getDeclaringClass());
	} catch (Throwable e) {
		Log.e(TAG, "addReplaceMethod", e);
	}
}
// 最终调用native进行方法替换
private static native void replaceMethod(Method dest, Method src);
```

native进行方法替换
```
// andfix.cpp
static void replaceMethod(JNIEnv* env, jclass clazz, jobject src,
		jobject dest) {
	// 对不同的jvm实现进行判断区分
	if (isArt) {
		art_replaceMethod(env, src, dest);
	} else {
		dalvik_replaceMethod(env, src, dest);
	}
}
```

可以查看art_method_replace.cpp中，是对不同版本的设备进行区分，ART可能会有不同版本的差异，需要单独处理
```
extern void __attribute__ ((visibility ("hidden"))) art_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
    if (apilevel > 23) {
        replace_7_0(env, src, dest);
    } else if (apilevel > 22) {
		replace_6_0(env, src, dest);
	} else if (apilevel > 21) {
		replace_5_1(env, src, dest);
	} else if (apilevel > 19) {
		replace_5_0(env, src, dest);
    }else{
        replace_4_4(env, src, dest);
    }
}
```

以replace_7_0为例，对函数指针的替换，用新的method替换旧的method内容：
```
void replace_7_0(JNIEnv* env, jobject src, jobject dest) {
	art::mirror::ArtMethod* smeth =
			(art::mirror::ArtMethod*) env->FromReflectedMethod(src);

	art::mirror::ArtMethod* dmeth =
			(art::mirror::ArtMethod*) env->FromReflectedMethod(dest);

//	reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->class_loader_ =
//			reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->class_loader_; //for plugin classloader
	reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->clinit_thread_id_ =
			reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->clinit_thread_id_;
	reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->status_ =
			reinterpret_cast<art::mirror::Class*>(smeth->declaring_class_)->status_ -1;
	//for reflection invoke
	reinterpret_cast<art::mirror::Class*>(dmeth->declaring_class_)->super_class_ = 0;

	smeth->declaring_class_ = dmeth->declaring_class_;
	smeth->access_flags_ = dmeth->access_flags_  | 0x0001;
	smeth->dex_code_item_offset_ = dmeth->dex_code_item_offset_;
	smeth->dex_method_index_ = dmeth->dex_method_index_;
	smeth->method_index_ = dmeth->method_index_;
	smeth->hotness_count_ = dmeth->hotness_count_;

	smeth->ptr_sized_fields_.dex_cache_resolved_methods_ =
			dmeth->ptr_sized_fields_.dex_cache_resolved_methods_;
	smeth->ptr_sized_fields_.dex_cache_resolved_types_ =
			dmeth->ptr_sized_fields_.dex_cache_resolved_types_;

	smeth->ptr_sized_fields_.entry_point_from_jni_ =
			dmeth->ptr_sized_fields_.entry_point_from_jni_;
	smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_ =
			dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_;

	LOGD("replace_7_0: %d , %d",
			smeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_,
			dmeth->ptr_sized_fields_.entry_point_from_quick_compiled_code_);

}
```

3、 `addPatch()`具体实现：
```
/**
 * add patch at runtime
 *
 * @param path .patch修复包的存放路径，patch path
 * @throws IOException
 */
public void addPatch(String path) throws IOException {
    File src = new File(path);
    File dest = new File(mPatchDir, src.getName());
    if (!src.exists()) {
        throw new FileNotFoundException(path);
    }
    if (dest.exists()) {
        Log.d(TAG, "patch [" + path + "] has be loaded.");
        return;
    }
    // 具体作用是将.patch文件copy到file文件目录下，删除patch文件后不影响修复效果
    FileUtil.copyFile(src, dest);// copy to patch's directory
    // 先进行.patch文件copy，热修复将一直使用的是file目录下的修复包
    Patch patch = addPatch(dest);
    if (patch != null) {
        loadPatch(patch);
    }
}
```

#### 四、归纳总结

> 从阅读源码的过程中，可以发现AndFix的具有的一些特点，轻量，支持Android多版本，进行方法热修复，单纯的热修复功能，AndFix会将.patch做保存，会有一种一次修复持久有效的功效（即使删除下载之后的.patch文件）。相对缺少一些对于res布局文件修复的功能。
> 阅读的过程中，会有一些体会是，如果我去实现一个热修复框架应该考虑什么点，注意一些什么？去深入探索一下触手所不能及的地方，去看看，自己动手找找，可能会记忆更深刻。

#### 参照

* [热修复之 Method Hook 原理](https://juejin.im/entry/595f4a7d51882568b462fc4f)
* [Android热补丁笔记](http://jackieming.com/blog/2017/04/02/Android%E7%83%AD%E8%A1%A5%E4%B8%81%E7%AC%94%E8%AE%B0/)
* [MethodHook](https://github.com/pqpo/MethodHook)