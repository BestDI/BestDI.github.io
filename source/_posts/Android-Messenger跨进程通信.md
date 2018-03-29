---
title: Android Messenger跨进程通信
date: 2018-01-03 00:41:33
tags: [Android]
categories: [Android]
---

<!-- http://blog.csdn.net/lmj623565791/article/details/47017485 -->

* [Google API](https://developer.android.com/reference/android/os/Messenger.html) Reference to a Handler, which others can use to send messages to it. This allows for the implementation of message-based communication across processes, by creating a Messenger pointing to a Handler in one process, and handing that Messenger to another process. 
*Messenger允许跨进程基于消息的通信，通过在一个进程创建指向Handler的Messenger，然后在另外一个进程处理。*
Note: the implementation underneath is just a simple wrapper around a [Binder](https://developer.android.com/reference/android/os/Binder.html) that is used to perform the communication. 
*Messenger是Binder的简单包装。*
* 参见 [Google Remote Messenger Sample](https://developer.android.com/reference/android/app/Service.html#RemoteMessengerServiceSample)


<!-- more -->
> 跨进程简单理解模型：
> ![](/images/2018_0103/client_server.png)

## 一、 Demo演示


同一个App中，将Service设置为remote，实现一个App多个进程。

### Server端

#### Service代码

```java
import android.app.Service;
import android.content.Intent;
import android.os.Handler;
import android.os.IBinder;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.support.annotation.Nullable;

/**
 * Created by Mia.
 */
public class ServerService extends Service {

    public static final int MSG_DIFF = 0x110;

    private Messenger messenger = new Messenger(new Handler() {

        @Override
        public void handleMessage(Message msgFromClient) {
            // Message.obtain(Message),通过已有Message对象创建同样属性的Message对象
            // copies the values of an existing message (including its target) into the new one.
            Message msgToClient = Message.obtain(msgFromClient);
            switch (msgFromClient.what) {
                case MSG_DIFF:
                    msgToClient.what = MSG_DIFF;
                    try {
                        Thread.sleep(2000);
                        msgToClient.arg2 = msgFromClient.arg1 - msgFromClient.arg2;
                        msgFromClient.replyTo.send(msgToClient);
                    } catch (InterruptedException | RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
            }
            super.handleMessage(msgFromClient);
        }
    });

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

#### 配置在AndroidManifest.xml

```xml
<!-- 设置remote之后，不仅仅是进程id不同了，就连应用程序包名也不一样了，包名后面还跟上了:remote标识。 -->

<service  
    android:name="*.ServerService"  
    android:process=":remote" >  
</service> 
```

### Client端

#### Activity

```java
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity{
    private static final String TAG = "MainActivity";

    private Button mBtnAdd;
    private LinearLayout mLyContainer;
    //显示连接状态
    private TextView mTvState;

    private Messenger mService;
    private boolean isConn;


    private Messenger mMessenger = new Messenger(new Handler(){
        @Override
        public void handleMessage(Message msgFromServer)
        {
            switch (msgFromServer.what)
            {
                case ServerService.MSG_DIFF:
                    TextView tv = (TextView) mLyContainer.findViewById(msgFromServer.arg1);
                    tv.setText(tv.getText() + "=>" + msgFromServer.arg2);
                    break;
            }
            super.handleMessage(msgFromServer);
        }
    });


    private ServiceConnection mConn = new ServiceConnection(){
        @Override
        public void onServiceConnected(ComponentName name, IBinder service)
        {
            mService = new Messenger(service);
            isConn = true;
            mTvState.setText("connected!");
        }

        @Override
        public void onServiceDisconnected(ComponentName name)
        {
            mService = null;
            isConn = false;
            mTvState.setText("disconnected!");
        }
    };

    private int mA;

    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //开始绑定服务
        bindServiceInvoked();

        mTvState = (TextView) findViewById(R.id.id_tv_callback);
        mBtnAdd = (Button) findViewById(R.id.id_btn_add);
        mLyContainer = (LinearLayout) findViewById(R.id.id_ll_container);

        mBtnAdd.setOnClickListener(new View.OnClickListener(){
            @Override
            public void onClick(View v){
                try{
                    int a = mA++;
                    int b = (int) (Math.random() * 100);

                    //创建一个tv,添加到LinearLayout中
                    TextView tv = new TextView(MainActivity.this);
                    tv.setText(a + " + " + b + " = caculating ...");
                    tv.setId(a);
                    mLyContainer.addView(tv);

                    Message msgFromClient = Message.obtain(null, ServerService.MSG_DIFF, a, b);
                    msgFromClient.replyTo = mMessenger;
                    if (isConn)
                    {
                        //往服务端发送消息
                        mService.send(msgFromClient);
                    }
                } catch (RemoteException e){
                    e.printStackTrace();
                }
            }
        });
    }

    private void bindServiceInvoked(){
        Intent intent = new Intent();
        intent.setAction("com.zhy.aidl.calc");
        bindService(intent, mConn, Context.BIND_AUTO_CREATE);
        Log.e(TAG, "bindService invoked !");
    }

    @Override
    protected void onDestroy(){
        super.onDestroy();
        unbindService(mConn);
    }
}
```

#### Layout.xml

#### 效果图

## 二、 源码分析

**一句话总结：**Messenger的内部实现的，实际上也是依赖于aidl文件实现的。

### 客户端向服务端通信

#### Server

1.服务端中的Service#onBind():

```java
public IBinder onBind(Intent intent){
    return mMessenger.getBinder();
}
```
2.其mMessenger.getBinder()具体实现:

```java
public IBinder getBinder() {
    return mTarget.asBinder();
}
```

3.其mTarget对象即是new Messenger(Handler)的Handler

```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
```

4.进入target.getIMessenger(),观察其实现细节：

```java
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}

private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```
mTarget实质上是一个MessengerImpl对象，那么asBinder实际上是返回this，也就是MessengerImpl对象；
这是个内部类，可以看到继承自IMessenger.Stub，然后实现了一个send方法，该方法就是将接收到的消息通过 Handler.this.sendMessage(msg);
发送到handleMessage方法。
传统写aidl文件，aapt给我们生成什么，生成IXXX.Stub类，然后我们服务端继承IXXX.Stub实现接口中的方法。
没错，其实这里内部其实也是依赖一个aidl生成的类，这个aidl位于：frameworks/base/core/java/android/os/IMessenger.aidl.

```java
package android.os;  

import android.os.Message;  

/** @hide */  
oneway interface IMessenger {  
    void send(in Message msg);  
}  
```

Messenger并没有什么神奇之处，实际上，就是依赖该aidl文件生成的类，继承了IMessenger.Stub类，
实现了send方法，send方法中参数会通过客户端传递过来，最终发送给handler进行处理。

#### Client

客户端首先通过onServiceConnected拿到sevice（Ibinder）对象IMessenger.Stub.asInterface(service)拿到接口对象进行调用；
而，我们的代码中是mService = new Messenger(service);

```java
public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

其实质与平常写法一致！
客户端与服务端通信，实际上和我们平时的写法没有任何区别，通过编写aidl文件，服务端onBind利用Stub编写接口实现返回；客户端利用回调得到的IBinder对象，使用IMessenger.Stub.asInterface(target)拿到接口实例进行调用（内部实现）。


### 服务端向客户端通信

客户端send方法发送的是一个Message，这个Message.replyTo指向的是一个mMessenger，是我们自己初始化的，用于服务端向客户端通信。
那么将消息发送到服务端，肯定是通过序列化与反序列化拿到Message对象，我们看下Message的反序列化的代码：

#### Message

```java
private void readFromParcel(Parcel source) {
    what = source.readInt();
    arg1 = source.readInt();
    arg2 = source.readInt();
    if (source.readInt() != 0) {
        obj = source.readParcelable(getClass().getClassLoader());
    }
    when = source.readLong();
    data = source.readBundle();
    replyTo = Messenger.readMessengerOrNullFromParcel(source);
    sendingUid = source.readInt();
}
```

主要看replyTo，调用的是Messenger.readMessengerOrNullFromParcel

```java
public static Messenger readMessengerOrNullFromParcel(Parcel in) {
    IBinder b = in.readStrongBinder();
    return b != null ? new Messenger(b) : null;
}

public static void writeMessengerOrNullToParcel(Messenger messenger,
        Parcel out) {
    out.writeStrongBinder(messenger != null ? messenger.mTarget.asBinder()
            : null);
}
```

通过上面的writeMessengerOrNullToParcel可以看到，它将客户端的messenger.mTarget.asBinder()对象进行了恢复，客户端的message.mTarget.asBinder()是什么？
客户端也是通过Handler创建的Messenger，于是asBinder返回的是：

```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}

private final class MessengerImpl extends IMessenger.Stub {
     public void send(Message msg) {
         msg.sendingUid = Binder.getCallingUid();
         Handler.this.sendMessage(msg);
     }
}

public IBinder getBinder() {
     return mTarget.asBinder();
}
```

那么asBinder，实际上就是MessengerImpl extends IMessenger.Stub中的asBinder了。

#### IMessenger.Stub

```java
@Override 
public android.os.IBinder asBinder(){
return this;
}
```

那么其实返回的就是MessengerImpl对象自己。到这里可以看到message.mTarget.asBinder()其实返回的是客户端的MessengerImpl对象。

最终，发送给客户端的代码是这么写的：

```java
msgfromClient.replyTo.send(msgToClient);
public void send(Message message) throws RemoteException {
    mTarget.send(message);
}
```

这个mTarget实际上就是对客户端的MessengerImpl对象的封装，那么send(message)（屏蔽了transact/onTransact的细节），这个message最终肯定传到客户端的handler的handleMessage方法中。

> **一句话总结：**
> * 客户端通过bindService拿到服务端的Messenger对象；
> * 客户端给服务端发送消息的时候，使用Messager.replyTo携带客户端的Messenger对象到服务端；
> * 这样，客户端/服务端互相持有对方Binder对象（MessengerImpl）；
> * 实质还是利用了AIDL的方式。