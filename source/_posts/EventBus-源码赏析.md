---
title: EventBus 源码赏析
date: 2018-03-29 22:04:31
tags: [Android,三方框架]
categories: [Android,三方框架]
---

#### 一 、简单概括
 
 
- EventBus是一个Android/Java平台基于订阅与发布的同通信框架，不支持跨进程。可以用于Activity/Fragment/Thread/Service之间通信解耦，也可以用于多线程通信。
- EventBus相对于BroadcastReceiver、Handler、接口回调的有点在于简单，事件的订阅和发布解耦，但是存在的问题也很明显，会有大量的Event类来管理
- Event bus for Android and Java that simplifies communication between Activities, Fragments, Threads, Services, etc. Less code, better quality.
- [GITHUB直连]

<!-- more --> 

#### 二、EventBus的基本使用

![整体思路](https://raw.githubusercontent.com/guoxiaoxing/android-open-framework-analysis/master/art/eventbus/event_bus_structure.png)

- 注册订阅者
- 发布、接收Event
- 取消注册订阅者

#### 三、源码解析核心类

> 分析如下几个类，基本可以理清EventBus的event register/post

- `EventBus.java` 对外使用的类和方法
- `Subscription.java` 其中关键的`
final Object subscriber; // 订阅者
final SubscriberMethod subscriberMethod; // 订阅者方法`
- `SubscriberMethod.java` 其中关键的`
final Method method; // 订阅者中的方法
final ThreadMode threadMode; // 回调的线程模式
final Class<?> eventType; // Event类型，订阅Map的Key
final int priority; // 优先级
final boolean sticky; // 是否粘性`

##### 1. 注册订阅者

`EventBus.getDefault().register(this)` 
getDefault() 为单例获取方法，也可以使用EventBusBuilder实现实例
```java

// 订阅队列，**Map的Key为Event实例**，会将一组注册相同Event的归纳仅一个集合
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
// 后续准备取消的事件队列
private final Map<Object, List<Class<?>>> typesBySubscriber;
// 粘性事件队列
private final Map<Class<?>, Object> stickyEvents;

public void register(Object subscriber) {
    // 1. 获取订阅者的类名。
    Class<?> subscriberClass = subscriber.getClass();
    // 2. 查找当前订阅者的所有响应函数，即使用注解@Subscribe(threadMode = ThreadMode.MAIN)标记的方法，并将其得以封装在SubscriberMethod对象中
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        // 3. 循环每个事件响应函数
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

如下是真正`subscribe()`的过程
```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
      // 事件类型（xxxEvent）
      Class<?> eventType = subscriberMethod.eventType;
      // 1.将订阅者和订阅者的方法归纳到另一个对象中
      Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
      // 2.拿到该事件类型的所有订阅信息。
      CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
      if (subscriptions == null) {
          // 如果未有相同事件注册过，则创建一个此Event订阅列表
          subscriptions = new CopyOnWriteArrayList<>();
          subscriptionsByEventType.put(eventType, subscriptions);
      } else {
          if (subscriptions.contains(newSubscription)) {
              throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
          }
      }
      
      int size = subscriptions.size();
      // 3. 按照事件优先级将其插入订阅者列表中，其实就是插入顺序
      for (int i = 0; i <= size; i++) {
          if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
              subscriptions.add(i, newSubscription);
              break;
          }
      }
  
      // 4. 得到当前订阅者订阅的所有事件队列，存放在typesBySubscriber中，用于后续取消事件订阅。
      List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
      if (subscribedEvents == null) {
          subscribedEvents = new ArrayList<>();
          typesBySubscriber.put(subscriber, subscribedEvents);
      }
      subscribedEvents.add(eventType);
  
      // 5. 是否是粘性事件，如果是粘性事件，则从stickyEvents队列中取出最后一个该类型的事件发送给订阅者。
      if (subscriberMethod.sticky) {
          if (eventInheritance) {
              // Existing sticky events of all subclasses of eventType have to be considered.
              // Note: Iterating over all events may be inefficient with lots of sticky events,
              // thus data structure should be changed to allow a more efficient lookup
              // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
              Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
              for (Map.Entry<Class<?>, Object> entry : entries) {
                  Class<?> candidateEventType = entry.getKey();
                  if (eventType.isAssignableFrom(candidateEventType)) {
                      Object stickyEvent = entry.getValue();
                      checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                  }
              }
          } else {
              Object stickyEvent = stickyEvents.get(eventType);
              checkPostStickyEventToSubscription(newSubscription, stickyEvent);
          }
      }
}  
```

##### 2. 发布事件

`EventBus.getDefault().post(new Event("Event Message"));`
其具体的串联过程如下：
`post()` -> `postSingleEvent)()` -> `postSingleEventForEventType()` -> `postToSubscription()` -> `invokeSubscriber()`  -> `subscription.subscriberMethod.method.invoke(subscription.subscriber, event);` -> 最终通过反射调用订阅者的onEvent()，得以将消息传递


```java
public void post(Object event) {
    // 1. 获取当前线程的PostingThreadState对象，该对象包含事件队列，保存在ThreadLocal中。ThreadLocal将需要被线程保护的对象放入其中，此对象将和线程绑定，意味着每个线程都有自己的事件队列
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    // 2. 将当前事件加入到该线程的事件队列中。
    eventQueue.add(event);

    // 3. 判断事件是否在分发中。如果没有则遍历事件队列进行实际分发。
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                // 4. 事件分发并从队列中移除
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

> PostingThreadState用来描述发送事件的线程的相关状态信息，包含事件队列，是否是主线程、订阅者、事件Event等信息。

1. 获取当前线程的PostingThreadState对象，该对象包含事件队列，保存在ThreadLocal中。
2. 将当前事件加入到该线程的事件队列中。
3. 判断事件是否在分发中。如果没有则遍历事件队列进行实际分发。
4. 进行事件分发。

然后调用postSingleEvent()进行事件分发。

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // 1. 如果事件允许继承，则查找该事件类型的所有父类和接口，依次进行循环。
    if (eventInheritance) {
        
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            // 2. 查找该事件的所有订阅者。
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

该方法主要做了以下事情：

1. 如果事件允许继承，则查找该事件类型的所有父类和接口，依次进行循环。
2. 查找该事件的所有订阅者。

然后调用postSingleEventForEventType()方法查询当前事件的所有订阅者，如下所示：

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        // 1. 获取当前事件的所有订阅者。
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        // 2. 遍历所有订阅者。
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                // 3. 根据订阅者所在线程，调用事件响应函数onEvent()。
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}    
```

该方法主要做了以下事情：

1. 获取当前事件的所有订阅者。
2. 遍历所有订阅者。
3. 根据订阅者所在线程，调用事件响应函数onEvent()。

调用postToSubscription()方法根据订阅者所在线程，调用事件响应函数onEvent()，这便涉及到接收事件Event的处理了，我们接着来看。

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
       switch (subscription.subscriberMethod.threadMode) {
           case POSTING:
               invokeSubscriber(subscription, event);
               break;
           case MAIN:
               if (isMainThread) {
                   invokeSubscriber(subscription, event);
               } else {
                   mainThreadPoster.enqueue(subscription, event);
               }
               break;
           case MAIN_ORDERED:
               if (mainThreadPoster != null) {
                   mainThreadPoster.enqueue(subscription, event);
               } else {
                   // temporary: technically not correct as poster not decoupled from subscriber
                   invokeSubscriber(subscription, event);
               }
               break;
           case BACKGROUND:
               if (isMainThread) {
                   backgroundPoster.enqueue(subscription, event);
               } else {
                   invokeSubscriber(subscription, event);
               }
               break;
           case ASYNC:
               asyncPoster.enqueue(subscription, event);
               break;
           default:
               throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
       }
}    
```

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onEvent(Event event) {
    ·······
}
```
如上所示，onEvent函数上是可以加Subscribe注解了，该注解标明了onEvent()函数在哪个线程执行。主要有以下几个线程：

- PostThread：默认的 ThreadMode，表示在执行 Post 操作的线程直接调用订阅者的事件响应方法，不论该线程是否为主线程（UI 线程）。当该线程为主线程
时，响应方法中不能有耗时操作，否则有卡主线程的风险。适用场景：对于是否在主线程执行无要求，但若 Post 线程为主线程，不能耗时的操作；
- MainThread：在主线程中执行响应方法。如果发布线程就是主线程，则直接调用订阅者的事件响应方法，否则通过主线程的 Handler 发送消息在主线程中处理—
—调用订阅者的事件响应函数。显然，MainThread类的方法也不能有耗时操作，以避免卡主线程。适用场景：必须在主线程执行的操作；
- BackgroundThread：在后台线程中执行响应方法。如果发布线程不是主线程，则直接调用订阅者的事件响应函数，否则启动唯一的后台线程去处理。由于后台线程
是唯一的，当事件超过一个的时候，它们会被放在队列中依次执行，因此该类响应方法虽然没有PostThread类和MainThread类方法对性能敏感，但最好不要有重度耗
时的操作或太频繁的轻度耗时操作，以造成其他操作等待。适用场景：操作轻微耗时且不会过于频繁，即一般的耗时操作都可以放在这里；
- Async：不论发布线程是否为主线程，都使用一个空闲线程来处理。和BackgroundThread不同的是，Async类的所有线程是相互独立的，因此不会出现卡线程的问
题。适用场景：长耗时操作，例如网络访问。

这里我线程执行和EventBus的成员变量对应，它们都实现了Runnable与Poster接口，Poster接口定义了事件排队功能，这些本质上都是个Runnable，放在线程池里执行，如下所示：

`
private final Poster mainThreadPoster;
private final BackgroundPoster backgroundPoster;
private final AsyncPoster asyncPoster;
private final SubscriberMethodFinder subscriberMethodFinder;
private final ExecutorService executorService;`

##### 3. 取消订阅

取消注册订阅者调用的是以下方法：

```java
EventBus.getDefault().unregister(this);
```
具体如下所示：

```java
public synchronized void unregister(Object subscriber) {
        
    // 1. 获取当前订阅者订阅的所有事件类型。
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        // 2. 遍历事件队列，解除事件注册。
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        // 3. 移除事件订阅者。
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

取消注册订阅者的流程也十分简单，如下所示：

1. 获取当前订阅者订阅的所有事件类型。
2. 遍历事件队列，解除事件注册。
3. 移除事件订阅者。

当猴调用unsubscribeByEventType()移除订阅者，如下所示：

```java
 private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
     // 1. 获取所有订阅者信息。
     List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
     if (subscriptions != null)
         // 2. 遍历订阅者
         int size = subscriptions.size();
         for (int i = 0; i < size; i++) {
             Subscription subscription = subscriptions.get(i);
             // 3. 移除该订阅对象。
             if (subscription.subscriber == subscriber) {
                 subscription.active = false;
                 subscriptions.remove(i);
                 i--;
                 size--;
             }
         }
     }
}
```

#### 四、总结

> `EventBus`中使用到了现在一些比较主流的技术，注解、反射、ThreadLocal，组成了订阅发布的一组框架，从中能看到作者对Event的理解，以及在架构时对效率的考虑。从以eventClazz为Key，映射出所有订阅者的思想，和现实中以一对多的思想类似，代码的思想不是一点点得来的，停下笔思考一下，如何才会更好？

>  **文中大多是从网上得来，只算自己的一种归纳整理吧。**

[GITHUB直连]:<https://github.com/greenrobot/EventBus>