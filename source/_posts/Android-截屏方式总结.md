---
title: Android 截屏方式总结
date: 2017-12-18 20:18:19
tags: [Android]
categories: [Android]
---

## 一. Android如何禁止Application截屏？
```java 
getWindow().setFlags(LayoutParams.FLAG_SECURE, LayoutParams.FLAG_SECURE);
```

> 如果一遍不行，那就来两遍。[FLAG_SECURE.StackOverflow](https://stackoverflow.com/questions/28606689/how-to-prevent-screen-capture-in-android)

<!-- more --> 

## 二. 如何监听截屏

> 使用ContentObserver进行相册的监听,[原链](https://stackoverflow.com/questions/31360296/listen-for-screenshot-action-in-android)

```java
/** 定义 */
public abstract class ScreenShotContentObserver extends ContentObserver {

    private Context context;
    private boolean isFromEdit = false;
    private String previousPath;

    public ScreenShotContentObserver(Handler handler, Context context) {
        super(handler);
        this.context = context;
    }

    @Override
    public boolean deliverSelfNotifications() {
        return super.deliverSelfNotifications();
    }

    @Override
    public void onChange(boolean selfChange) {
        super.onChange(selfChange);
    }

    @Override
    public void onChange(boolean selfChange, Uri uri) {
        Cursor cursor = null;
        try {
            cursor = context.getContentResolver().query(uri, new String[]{
                    MediaStore.Images.Media.DISPLAY_NAME,
                    MediaStore.Images.Media.DATA
            }, null, null, null);
            if (cursor != null && cursor.moveToLast()) {
                int displayNameColumnIndex = cursor.getColumnIndex(MediaStore.Images.Media.DISPLAY_NAME);
                int dataColumnIndex = cursor.getColumnIndex(MediaStore.Images.Media.DATA);
                String fileName = cursor.getString(displayNameColumnIndex);
                String path = cursor.getString(dataColumnIndex);
                if (new File(path).lastModified() >= System.currentTimeMillis() - 10000) {
                    if (isScreenshot(path) && !isFromEdit && !(previousPath != null && previousPath.equals(path))) {
                        onScreenShot(path, fileName);
                    }
                    previousPath = path;
                    isFromEdit = false;
                } else {
                    cursor.close();
                    return;
                }
            }
        } catch (Throwable t) {
            isFromEdit = true;
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
        super.onChange(selfChange, uri);
    }

    private boolean isScreenshot(String path) {
        return path != null && path.toLowerCase().contains("screenshot");
    }

    protected abstract void onScreenShot(String path, String fileName);

}
```

```java
/** 使用 */
public class MainActivity extends AppCompatActivity {

    private ScreenShotContentObserver screenShotContentObserver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        screenShotContentObserver = new ScreenShotContentObserver(handler, this) {
            @Override
            protected void onScreenShot(String path, String fileName) {
                File file = new File(path); //this is the file of screenshot image
            }
        };
    }

    @Override
    public void onResume() {
        super.onResume();
        getContentResolver().registerContentObserver(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                true,
                screenShotContentObserver
        );
    }

    @Override
    public void onPause() {
        super.onPause();

        try {
            getContentResolver().unregisterContentObserver(screenShotContentObserver);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        try {
            getContentResolver().unregisterContentObserver(screenShotContentObserver);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

## 三. 如何截屏

### 常见截屏方式

> 1.获取当前Window的DrawingCache的方式，即DecorView的DrawingCache

```java
/**
   * shot the current screen ,with the status but the status is trans *
   *
   * @param ctx current activity
   */
  public static Bitmap shotActivity(Activity ctx) {

    View view = ctx.getWindow().getDecorView();
    view.setDrawingCacheEnabled(true);
    view.buildDrawingCache();

    Bitmap bp = Bitmap.createBitmap(view.getDrawingCache(), 0, 0, view.getMeasuredWidth(),
        view.getMeasuredHeight());

    view.setDrawingCacheEnabled(false);
    view.destroyDrawingCache();

    return bp;
  }
```

> 2. 获取某个View的DrawingCache

```java
public static Bitmap getViewBp(View v) {
        if (null == v) {
            return null;
        }
        v.setDrawingCacheEnabled(true);
        v.buildDrawingCache();
        if (Build.VERSION.SDK_INT >= 11) {
            v.measure(MeasureSpec.makeMeasureSpec(v.getWidth(),
                    MeasureSpec.EXACTLY), MeasureSpec.makeMeasureSpec(
                    v.getHeight(), MeasureSpec.EXACTLY));
            v.layout((int) v.getX(), (int) v.getY(),
                    (int) v.getX() + v.getMeasuredWidth(),
                    (int) v.getY() + v.getMeasuredHeight());
        } else {
            v.measure(MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED),
                    MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));
            v.layout(0, 0, v.getMeasuredWidth(), v.getMeasuredHeight());
        }
        Bitmap b = Bitmap.createBitmap(v.getDrawingCache(), 0, 0, v.getMeasuredWidth(), v.getMeasuredHeight());

        v.setDrawingCacheEnabled(false);
        v.destroyDrawingCache();
        return b;
    }
```

### 使用三方Library

> [tarek360.Instacapture](https://github.com/tarek360/Instacapture),使用Kotlin/Java实现的一个三方库，可以针对常见的View和一般视图实现快速截图。可以截图的包括：
> 1. Google Map
> 2. Dialog,context menus,toasts
> 3. TextureView
> 4. GLSurfaceView
>
> 并且可以设置非截图View，无权限要求。

### ScrollView截屏

> 由于ScrollView只有一个直接子View，可以按照如下方式获取，[原链](https://stackoverflow.com/questions/43817116/take-a-full-screenshot-of-scrollview-android):

```java
Bitmap bitmap = getBitmapFromView(scrollview, scrollview.getChildAt(0).getHeight(), scrollview.getChildAt(0).getWidth());

//create bitmap from the ScrollView 
private Bitmap getBitmapFromView(View view, int height, int width) {
    Bitmap bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(bitmap);
    Drawable bgDrawable = view.getBackground();
    if (bgDrawable != null)
        bgDrawable.draw(canvas);
    else
        canvas.drawColor(Color.WHITE);
    view.draw(canvas);
    return bitmap;
}
```

### ListView

> ListView中存在的问题，条目过多，有很多未可见[原链](http://stackoverflow.com/questions/12742343/android-get-screenshot-of-all-listview-items)。

```java
public static Bitmap getWholeListViewItemsToBitmap() {

    ListView listview    = MyActivity.mFocusedListView;
    ListAdapter adapter  = listview.getAdapter(); 
    int itemscount       = adapter.getCount();
    int allitemsheight   = 0;
    List<Bitmap> bmps    = new ArrayList<Bitmap>();

    for (int i = 0; i < itemscount; i++) {

        View childView      = adapter.getView(i, null, listview);
        childView.measure(MeasureSpec.makeMeasureSpec(listview.getWidth(), MeasureSpec.EXACTLY), 
                MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));

        childView.layout(0, 0, childView.getMeasuredWidth(), childView.getMeasuredHeight());
        childView.setDrawingCacheEnabled(true);
        childView.buildDrawingCache();
        bmps.add(childView.getDrawingCache());
        allitemsheight+=childView.getMeasuredHeight();
    }

    Bitmap bigbitmap    = Bitmap.createBitmap(listview.getMeasuredWidth(), allitemsheight, Bitmap.Config.ARGB_8888);
    Canvas bigcanvas    = new Canvas(bigbitmap);

    Paint paint = new Paint();
    int iHeight = 0;

    for (int i = 0; i < bmps.size(); i++) {
        Bitmap bmp = bmps.get(i);
        bigcanvas.drawBitmap(bmp, 0, iHeight, paint);
        iHeight+=bmp.getHeight();

        bmp.recycle();
        bmp=null;
    }


    return bigbitmap;
}
```

### RecyclerView

> ListView的替代品RecyclerView,[原链](https://stackoverflow.com/questions/30085063/take-a-screenshot-of-recyclerview-in-full-length)

```java
public Bitmap getScreenshotFromRecyclerView(RecyclerView view) {
        RecyclerView.Adapter adapter = view.getAdapter();
        Bitmap bigBitmap = null;
        if (adapter != null) {
            int size = adapter.getItemCount();
            int height = 0;
            Paint paint = new Paint();
            int iHeight = 0;
            final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

            // Use 1/8th of the available memory for this memory cache.
            final int cacheSize = maxMemory / 8;
            LruCache<String, Bitmap> bitmaCache = new LruCache<>(cacheSize);
            for (int i = 0; i < size; i++) {
                RecyclerView.ViewHolder holder = adapter.createViewHolder(view, adapter.getItemViewType(i));
                adapter.onBindViewHolder(holder, i);
                holder.itemView.measure(View.MeasureSpec.makeMeasureSpec(view.getWidth(), View.MeasureSpec.EXACTLY),
                        View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED));
                holder.itemView.layout(0, 0, holder.itemView.getMeasuredWidth(), holder.itemView.getMeasuredHeight());
                holder.itemView.setDrawingCacheEnabled(true);
                holder.itemView.buildDrawingCache();
                Bitmap drawingCache = holder.itemView.getDrawingCache();
                if (drawingCache != null) {

                    bitmaCache.put(String.valueOf(i), drawingCache);
                }
//                holder.itemView.setDrawingCacheEnabled(false);
//                holder.itemView.destroyDrawingCache();
                height += holder.itemView.getMeasuredHeight();
            }

            bigBitmap = Bitmap.createBitmap(view.getMeasuredWidth(), height, Bitmap.Config.ARGB_8888);
            Canvas bigCanvas = new Canvas(bigBitmap);
            bigCanvas.drawColor(Color.WHITE);

            for (int i = 0; i < size; i++) {
                Bitmap bitmap = bitmaCache.get(String.valueOf(i));
                bigCanvas.drawBitmap(bitmap, 0f, iHeight, paint);
                iHeight += bitmap.getHeight();
                bitmap.recycle();
            }
        }
        return bigBitmap;
    }
```

### WebView

> [原链](https://stackoverflow.com/questions/9745988/how-can-i-programmatically-take-a-screenshot-of-a-webview-capturing-the-full-pa)

```java
w = new WebView(this);
    w.setWebViewClient(new WebViewClient()
    {
            public void onPageFinished(WebView view, String url)
            {
                    Picture picture = view.capturePicture();
                    Bitmap  b = Bitmap.createBitmap( picture.getWidth(),
                    picture.getHeight(), Bitmap.Config.ARGB_8888);
                    Canvas c = new Canvas( b );

                    picture.draw( c );
                    FileOutputStream fos = null;
                    try {

                        fos = new FileOutputStream( "mnt/sdcard/yahoo.jpg" );
                            if ( fos != null )
                            {
                                b.compress(Bitmap.CompressFormat.JPEG, 100, fos);

                                fos.close();
                            }
                        }
                   catch( Exception e )
                   {

                   }
          }
      });
```