---
title: 禁止WebView长按复制
date: 2017-12-13 23:37:03
tags: [Android,WebView]
categories: [Android,WebView]
---

#### Android WebView安全相关，禁止长按复制

```java
/** 通过消费WebView的长按事件来禁止复制 */
webview.setOnLongClickListener(new View.OnLongClickListener() {

         @Override
         public boolean onLongClick(View v) {
             return true;
         }
});
```