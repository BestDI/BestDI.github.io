---
title: Java计算两个日期之间的天数
date: 2017-12-14 22:31:21
tags: [Android,Java]
categories: [Android,Java]
---

#### Java中计算两个日期之间的天数,比如说**2017/12/06**与**2017/12/04**之间的天数差为2

```java
/**
 * Get a diff between two dates
 *
 * @param oldDate
 *            the old date
 * @param newDate
 *            the new date
 * @return the diff value, in the days
 */
public static long getDateDiff(SimpleDateFormat format, String oldDate, String newDate) {
	try {
		return TimeUnit.DAYS.convert(
			format.parse(newDate).getTime() - format.parse(oldDate).getTime(),
			TimeUnit.MILLISECONDS);
	} catch (Exception e) {
		e.printStackTrace();
		return 0;
	}
}
```

<!-- more --> 

#### 使用方式：

```java
SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy");
long dateDiff = getDateDiff(sdf, "4/12/2017", "6/12/2017");
System.out.println(" days diff is : " + dateDiff);
```

#### 控制台将输出:

```bash
days diff is : 2
```

参考[link](https://stackoverflow.com/questions/21285161/android-difference-between-two-dates)