---
title: Looper中的死循环
date: 2019-04-25 07:35:09
categories: Android Application
tags:
---

Handler 应该是每个 Android 开发同学都非常熟悉的组件了，和大部分的 GUI 程序类似，Android 上的 UI 绘制只允许在单个线程上进行。这也是 Google 提供了 Handler 的原因之一，为了方便在其他线程使用 Handler 来通知 UI 线程更新。关于 Handler 的使用和源码解析不是本文的主题，网上已经有很多不错的文章了。本文的主题是探讨下 Looper 类中的 `loop()` 方法。

我们知道，`loop()` 实质上是在一个无限的 `for` 循环中，从 `MessageQueue` 中不断地取出消息，再交由对应的 `Handler` 进行分发。这里实际上就涉及到了三个问题：

1. 为什么要是死循环？
2. 既然是死循环，那为什么不会阻塞应用？
3. 这种死循环，会不会非常消耗资源？

在写这篇文章之前，我也从网上看到很多答案，大致上是说，Handler 基于 Epoll 模型实现，只有在有新消息进来时，才会重新唤醒等待的线程，这里指 UI 线程。当时并没有去深究这些答案，觉得好像说的挺对的。实际上，这些答案确实是对的，但并没有回答完整，只回答了上面的第三个问题。

当我在 Java 程序上，仿照 Handler 实现了一个简易版本的 Handler，这是我实现的 `loop()`：

```java
public static void loop() {                                              
    if (sThreadLocal.get() == null) {                                    
        throw new IllegalStateException("must call prepare() before");   
    }                                                                    
    final Looper curLooper = sThreadLocal.get();                         
    while (true) {                                                       
        Message message = curLooper.messageQueue.next();                 
        if (message == null) {                                           
            // error                                                     
            System.out.println("message is null");                       
            return;                                                      
        }                                                                
        message.target.dispatchMessage(message);                         
    }                                                                    
}                                                                        
```

大家不用去深究每个方法的具体实现，从命名理解即可，和 Android 版的 Handler 一样，在一个死循环中，不断从 MessageQueue 中取出消息。但当我使用以下代码执行时，却发现完全按预期那样打印，

```java
public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(10);
        Looper.prepare();
        Looper.loop();
        Handler handler = new Handler() {
            @Override
            protected void handleMessage(Message msg) {
                System.out.println(Thread.currentThread().getName() + ": handle " + msg.getData());
            }
        };
        service.execute(() -> handler.sendMessage(new Message("Hello World")));
}
```

`loop()` 方法阻塞了当前线程，无法执行后面的代码。也就是说，Android Handler 应该也是这样的效果的。有疑惑的同学可以尝试以下代码，可以发现 "After" 不会被打印的。

```kotlin
val thread = Thread(Runnable {      
    println("Before")               
    Looper.prepare()                
    Looper.loop()                   
    println("After")                
})                                  
thread.start()                      
```

为了一探究竟，我们看下执行主线程的 `Looper.loop()` 方法执行，它位于 `ActivityThread.main()` 方法中，和普通的 Java 程序一样，Android 的入口也是 `main()` 方法，执行该方法的线程就是我们说的主线程，即 UI 线程。

```java
                                                                                                                                  
Looper.prepareMainLooper();                                                          
                                                                                                                                                                 
ActivityThread thread = new ActivityThread();                                        
thread.attach(false, startSeq);                                                      
                                                                                     
if (sMainThreadHandler == null) {                                                    
    sMainThreadHandler = thread.getHandler();                                        
}                                                                                    
                                                                                                                                                                                                                                                        
                                    
Looper.loop();                                                                       
                                                                                     
throw new RuntimeException("Main thread loop unexpectedly exited");                  
```

这段代码位于 `main()` 方法的最后，注意最后一句 `throw new RuntimeException("Main thread loop unexpectedly exited");` 也就是说，`Looper.loop()` 之后一般情况下并不会执行到那里。

接下来，就可以来回答第一个问题了，为什么是死循环？我们知道应用在执行完任务后就会退出，但 Android 应用开始运行后，即使没任务也应该保持执行，所以使用死循环来使得应用不结束。

第二个问题，为什么不会阻塞应用。其实是会的，更确切地说，是阻塞了主线程。但 Android 的 UI 事件都是通过 Handler 来调度的，比如启动 Activity，Activity 的生命周期调用等等，这一块有兴趣的同学，可以去看下 Activity 的启动流程，具体可以看看 `ActivityThread.H` 这个类。

### 总结

总结下上面我们讨论的，`Looper.loop()` 方法开始执行后，会阻塞了当前执行的线程，主线程也是这样，这样做的原因之一是，保持当前应用执行不退出。 