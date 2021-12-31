### 1. 作用
> 先来思考一个问题，android线程间内存是共享的，为什么我们还需要Handler传递消息?

1. 为了UI渲染不卡顿，需要将UI渲染和耗时任务放在不同线程中执行，互不干扰保证UI渲染的流畅性。
2. 为了做到第1点，Android默认将UI渲染限制在UI线程中执行
```java
// ViewRootImpl.java:

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```
可以看到在执行requestLayout时检查了是否在UI线程

> 如果没有Handler我们该怎么做?
3. 限于1和2的背景，应用到我们实际场景中去，假如我们在子线程做完耗时任务后，将数据直接赋值给UI线程中，这个时候UI线程并不知道该值发生了变化，于是我们写一个Listener通知主线程，像这样:
```kotlin
var test = 1
private var listener = object: TestCallback {
    override fun call(t: Int) {
        // 当前线程: 子线程
        Log.d("当前线程:",  Thread.currentThread().toString())
    }
}
override fun onCreate(savedInstanceState: Bundle?) {
    findViewById<Button>(R.id.btn2).setOnClickListener {
        Thread(Runnable {
            test++
            listener.call(test)
        }).start()
    }
}

```
这个时候发现在子线程中调该listener的回调方法仍然是运行在子线程，并没有实现将数据传递给UI线程且UI线程能监听到的效果。<br/>
4. 此时我们就会想到在UI线程起一个轮询，时刻关注着test值的变化，于是今天的主角Handler登场了。

### 2. 原理
先来介绍一下Handler消息处理机制的流程，请看图解：
![image](http://file.jinxianyun.com/handler%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)

图中大致了解到子线程调了handler的sendMessage方法，将Message添加到MessageQueue，loop循环取出Message后，将消息分发(dispatchMessage)给Callback回调(handleMessage)，即将子线程中的Message传到了主线程的handleMessage回调中。
<br/>

### 3. 源码分析
#### 3.1 Looper/Handler/MessageQueue类剖析
> Looper.java
```
静态成员：
    ThreadLocal<Looper> sThreadLocal //线程存储管理类，用来存储Looper
    Looper sMainLooper //主线程的Looper
普通成员：
    MessageQueue mQueue // 消息链表
    Thread mThread //当前线程
静态方法：
    prepare() //创建Looper并存在sThreadLocal,即：将线程和Looper绑定
    prepareMainLooper() //ActivityThread创建主线程的Looper并存储到sThreadLocal和sMainLooper
    loop() //从sThreadLocal获取当前Looper，获取Looper里的mQueue,并循环获取message，然后调Message对应的handler的dispatchMessage方法
    myLooper() //在sThreadLocal中获取当前线程的Looper()
```
> Handler.java
```
构造：
    Handler(Looper looper, Callback callback)
普通成员:
    Looper mLooper //对应的Looper
    MessageQueue mQueue //Looper里的mQueue
    Callback mCallback //handleMessage回调
普通方法：
    dispatchMessage(Message msg) //分发给回调handleMessage
    Message obtainMessage() //创建Message
    post(Runnable r) //创建message并sendMessage
    removeCallbacks(Runnable r) //在mQueue移除Message
    sendMessage(Message msg) // 将msg绑定当前handler后执行queue的enqueueMessage
    removeMessages(int what) //执行mQueue.removeMessages
    hasMessages(int what) // 执行mQueue.hasMessages
```
> MessageQueue.java
```
普通成员：
    long mPtr //共享native层的消息机制的指针
    Message mMessages //具有next指针的链表
    next() //获取下一个message
    enqueueMessage //将消息加入到链表
    removeMessages // 移除消息
```

#### 3.2 关键流程代码

> 3.2.1 子线程发送消息：handler.sendMessage(Message msg);
```java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    ...
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    ...
    return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this; //将当前handler赋值给Message的target
    ...
    return queue.enqueueMessage(msg, uptimeMillis); 
}
```
> 3.2.2 将消息加入消息队列：queue.enqueueMessage(msg, uptimeMillis);
```java
boolean enqueueMessage(Message msg, long when) {
    Message p = mMessages;
    boolean needWake;
    if (p == null || when == 0 || when < p.when) {
        msg.next = p;
        mMessages = msg;
        needWake = mBlocked;
    } else {
        Message prev;
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
        }
        msg.next = p;
        prev.next = msg;
    }
}
```
图解一下可以发现是单向链表MessageQueue进行插入操作：
![image](http://file.jinxianyun.com/messagequeue_insert.jpg?imageMogr2/thumbnail/!60p)

> 3.2.3 遍历消息链表并dispatchMessage：Looper.loop()
```java
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            return;
        }
        msg.target.dispatchMessage(msg);
    }
}
```

> 3.2.4 在MessageQueue获取Message：MessageQueue.next()
```java
Message next() {
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);
        Message prevMsg = null;
        Message msg = mMessages;
        if (msg != null && msg.target == null) {
            do {
                prevMsg = msg;
                msg = msg.next;
            } while (msg != null);
        }
        if (msg != null) {
            if (prevMsg != null) {
                prevMsg.next = msg.next;
            } else {
                mMessages = msg.next;
            }
            msg.next = null;
            return msg;
        } 
    }
}
```
图解一下MessageQueue取消息操作：
![image](http://file.jinxianyun.com/queue_next.jpg?imageMogr2/thumbnail/!60p)

> 3.2.4 在Looper取到消息后调Meesage.handler.dispatchMessage()
```java
public void dispatchMessage(@NonNull Message msg) {
    //handler.post(Runnable)的情况
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //触发回调方法
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

### 4.应用
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    //假装自己是主线程
    final MainThread mainThread = new MainThread();
    mainThread.start();

    findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            //开一个子线程发消息给主线程
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Message msg = Message.obtain();
                    msg.what = 666;
                    mainThread.mHandler.sendMessage(msg);
                }
            }).start();
        }
    });
}

class MainThread extends Thread {
    public Handler mHandler;
    @Override
    public void run() {
        Looper.prepare();
        mHandler = new Handler(Looper.myLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                // print: nell-handleMessage: what:666
                Log.d("nell-handleMessage", "what:" + msg.what);
            }
        };
        Looper.loop();
    }
}
```
### 5. 难点
> 细心的同学可能会问为什么在UI线程中loop()无限循环，却不堵塞UI线程呢?

Looper的loop无限循环取message不堵塞ui，是因为messagequeue的next方法，调用了nativePollOnce(ptr,nextPollTimeoutMillis),该方法是native方法，借助linux的管道和epoll机制，epoll机制是高效的IO多路复用机制，最终nativePollOnce阻塞在epoll_wait方法，该方法会释放当前线程的CPU资源并进入休眠状态，等到下一个消息到达再唤醒线程。所以loop是死循环，当线程空闲时进入休眠状态，不会消耗大量CPU资源。

### 6. 总结
Handler 发送的消息由 MessageQueue 存储管理，并由 Loopler 负责回调消息到 handleMessage()。

---
完结，撒花🎉