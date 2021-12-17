### 一、概要
续上一篇搭建篇[《Android实战——Cocos游戏容器搭建篇》](https://github.com/qq326646683/tech-article/blob/master/android/Android%E5%AE%9E%E6%88%98%E2%80%94%E2%80%94Cocos%E6%B8%B8%E6%88%8F%E5%AE%B9%E5%99%A8%E6%90%AD%E5%BB%BA%E7%AF%87.md)，本篇带来cocos和Android通信篇的实现和使用, 围绕着多进程通信和cocos-android互调来实现

### 二、通信模型
![image](http://file.jinxianyun.com/cocos_android_msg.png)
如果不需要主进程的数据，可以直接1->4

### 三、如何实现通信

#### 1.1 cocos调android

cocos/mainUI.ts:
```typescript
cocosCallNative(action: String, argument: String, callbackId: String) {
    jsb.reflection.callStaticMethod(
        'com.cocos.bridge.CocosCallNative',
        'invoke',
        '(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V',
        action,
        argument,
        callbackId
    )
}
```
#### 1.2 android接收cocos

android/CocosCallNative.java:
```java
public class CocosCallNative {
    public static void invoke(String action, String argument, String callbackId) {
        CocosBridgeHelper.log("cocos进程获得游戏端数据", "argument： " + argument + ",action: " + action + ",callbackId: " + callbackId);
        // 下发给cocos进程的监听者(不需要去主进程取数据时使用)
        CocosBridgeHelper.getInstance().dispatchCocosListener(action, argument, callbackId);
        try {
            // 2.0 cocos进程发消息给主进程
            CocosActivity.mIAIDLCocos2Main.cocos2Main(action, argument, callbackId);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```
#### 2.1 cocos进程发消息给主进程
##### 2.1.1 编写aidl
android/IAIDLCocos2Main.aidl
```java
interface IAIDLCocos2Main {
    void cocos2Main(String action, String argument, String callbackId);
    // 将cocos进程的IAIDLCallBack传到主进程，用来主进程发消息给Cocos进程
    void setAIDLCallBack(IAIDLCallBack iAIDLCallBack);
}
```
##### 2.1.2 创建服务Cocos2MainService
android/Cocos2MainService.java
```java
public class Cocos2MainService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    IAIDLCocos2Main.Stub mBinder = new IAIDLCocos2Main.Stub() {

        @Override
        public void cocos2Main(String action, String argument, String callbackId) {
            CocosBridgeHelper.log("主进程收到Cocos进程消息", "action:" + action + "argument:" + argument + "callbackId:" + callbackId);
            // 主进程收到消息，下发给主进程的监听者
            CocosBridgeHelper.getInstance().dispatchMainListener(action, argument, callbackId);
        }

        @Override
        public void setAIDLCallBack(IAIDLCallBack iAIDLCallBack) {
            // 将cocos进程的IAIDLCallBack传到主进程，用来主进程发消息给Cocos进程
            CocosBridgeHelper.getInstance().setIAIDLCallBack(iAIDLCallBack);
        }
    };
}
```

##### 2.1.3 在cocos进程启动服务相关
android/CocosActivity.java
```java
public static IAIDLCocos2Main mIAIDLCocos2Main;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    Intent intent = new Intent(this, Cocos2MainService.class);
    bindService(intent, this, Context.BIND_AUTO_CREATE);
}

// 死亡代理，保证服务通信通道正常
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        if (mIAIDLCocos2Main == null) return;

        mIAIDLCocos2Main.asBinder().unlinkToDeath(mDeathRecipient, 0);
        mIAIDLCocos2Main = null;
        // 重新绑定远程服务
        Intent intent = new Intent(CocosActivity.this, Cocos2MainService.class);
        bindService(intent, CocosActivity.this, Context.BIND_AUTO_CREATE);
    }
};

@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    mIAIDLCocos2Main = IAIDLCocos2Main.Stub.asInterface(service);
    try {
        // 注册死亡代理
        service.linkToDeath(mDeathRecipient, 0);
        //将cocos进程的IAIDLCallBack传到主进程，用来主进程发消息给Cocos进程
        mIAIDLCocos2Main.setAIDLCallBack(iAidlCallBack);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}

@Override
public void onServiceDisconnected(ComponentName name) {
    mIAIDLCocos2Main = null;
}

```
#### 2.1 Android主进程接收到消息
android/Cocos2MainService.java
```java
@Override
public void cocos2Main(String action, String argument, String callbackId) {
    CocosBridgeHelper.log("主进程收到Cocos进程消息", "action:" + action + "argument:" + argument + "callbackId:" + callbackId);
    CocosBridgeHelper.getInstance().dispatchMainListener(action, argument, callbackId);
}
```

#### 3.1主进程回消息给cocos进程
android/IAIDLCallBack.aidl
```java
interface IAIDLCallBack {
    void main2Cocos(String action, String argument, String callbackId);
}
```

android/CocosBridgeHelper.java
```java
public void main2Cocos(String action, String argument, String callbackId) {
    try {
        mIAIDLCallBack.main2Cocos(action, argument, callbackId);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
```

#### 3.2 cocos进程接受主进程的消息
android/CocosActivity.java
```java
private IAIDLCallBack iAidlCallBack = new IAIDLCallBack.Stub() {
    @Override
    public void main2Cocos(String action, String argument, String callbackId) {
        CocosBridgeHelper.log("Cocos进程收到主进程消息", "action: " + action + ", argument: " + argument + ", callbackId: " + callbackId);
        CocosBridgeHelper.getInstance().nativeCallCocos(action, argument, callbackId);
    }
};
```

#### 4.1 Android调cocos
android/CocosBridgeHelper.java
```java
public void nativeCallCocos(String action, String argument, String callbackId) {
    CocosHelper.runOnGameThread(() -> CocosJavascriptJavaBridge.evalString(String.format("cc.nativeCallCocos('%s', '%s', '%s');", action, argument, callbackId)));
}
```

#### 4.2 cocos接收Android发来的消息
cocos/mainUI.ts
```typescript
cc.nativeCallCocos = function(action: String, argument: String, callbackId: String) {
    if (action == ACTION_SHOW_STARDIALOG) {
        uiManager.instance.showDialog('common/tips', [argument]);
    } else if (action == ACTION_APP_VERSION) {
        uiManager.instance.showDialog('common/tips', [argument]);
    }

};
```

### 四、如何使用

#### 1. 先来看效果
![image](http://file.jinxianyun.com/cocos_msg.gif)

#### 2. 实现弹出Android的Dialog,选择后把结果传给cocos显示(不需要主进程的数据，可以直接1->4)
android/CocosGameActivity.kt
```java
private val showArray = arrayOf("刘德华", "周华健")
private val cocosListenerInCocos: CocosDataListener = CocosDataListener { action, argument, callbackId ->
    CocosBridgeHelper.log("接收InCocos", action)
    if (action == "action_showStarDialog") {
        runOnUiThread {
            AlertDialog.Builder(this)
                .setTitle("选择")
                .setItems(showArray) { _, index ->
                    CocosBridgeHelper.getInstance().nativeCallCocos(action, showArray[index], callbackId)
                }
                .create()
                .show()
        }
    }

}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    CocosBridgeHelper.getInstance().addCocosListener(cocosListenerInCocos)
}
override fun onDestroy() {
    super.onDestroy()
    CocosBridgeHelper.getInstance().removeCocosListener(cocosListenerInCocos)
}
```
#### 2. 实现从主进程取数据给cocos显示
android/MainActivity.kt
```java
private val cocosListenerInMain: CocosDataListener = CocosDataListener { action, argument, callbackId ->
    CocosBridgeHelper.log("接收InMain", action)
    if (action == "action_appVersion") {
        CocosBridgeHelper.getInstance().main2Cocos(action, packageManager.getPackageInfo(packageName, 0).versionName, callbackId)
    }
}
override fun onCreate(savedInstanceState: Bundle?) {
    CocosBridgeHelper.getInstance().addMainListener(cocosListenerInMain)
}

override fun onDestroy() {
    super.onDestroy()
    CocosBridgeHelper.getInstance().removeMainListener(cocosListenerInMain)
}
```

### 五、[本篇完整code](https://github.com/qq326646683/cocos-creator-android/commit/6e5151d1c62dd2bb4c9b2e49f6a4755fe5edd4e0)

---
完结，撒花🎉
