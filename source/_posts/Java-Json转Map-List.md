---
title: Java Json转Map/List
date: 2017-12-28 23:08:41
tags: [Java]
categories: [Java]
---

### Java Json转Map/List

------

#### Json格式

> 将格式如下的Json，转化成Map
``` java
{
“key1":"value1",
“key2":"value2",
“key3":"value3"
} 
```
> 将格式如下的Json，转化成List
> 
``` java
{
"key":{"key01":"value01","key02":"value02","key03":"value03"},
"key1":{"key01":"value01","key02":"value02","key03":"value03"}
}
```

<!-- more --> 

------

#### code

> 直接撸代码如下：

```java
/**
 * Json转Map
 *
 * @param jsonStr
 * @return
 */
public static Map<String, Object> getMapFromJSON(String jsonStr) {
    JSONObject jo;
    try {
        jo = new JSONObject(jsonStr);
        Iterator<String> iterator = jo.keys();
        String key;
        Object value;
        HashMap<String, Object> hashMap = new HashMap<>();
        while (iterator.hasNext()) {
            key = iterator.next();
            value = jo.get(key);
            hashMap.put(key, value);
        }
        return hashMap;
    } catch (JSONException e) {
        e.printStackTrace();
    }
    return null;
}

/**
 * Json转换成List<Map<>>
 *
 * @param jsonStr
 * @return
 */
public static List<Map<String, Object>> getListFromJSON(String jsonStr) {
    List<Map<String, Object>> list = null;
    try {
        JSONArray jsonArray = new JSONArray(jsonStr);
        JSONObject jsonObject;
        list = new ArrayList<>();
        int length = jsonArray.length();
        for (int i = 0; i < length; i++) {
            jsonObject = (JSONObject) jsonArray.get(i);
            list.add(getMapFromJSON(jsonObject.toString()));
        }
    } catch (JSONException e) {
        e.printStackTrace();
    }
    return list;
}
```