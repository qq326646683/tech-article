#### 1.背景介绍
开启悬浮窗后,小窗悬浮在app内及桌面上,并保持悬浮窗页面所有状态
> 预览

![image](http://file.jinxianyun.com/floatwindowdemo.gif)
[video](http://file.jinxianyun.com/floatwindowdemo.mp4)
> 路由介绍

![image](http://file.jinxianyun.com/floatwindowroute.jpg)
[image](http://file.jinxianyun.com/floatwindowroute.jpg)

> 功能概览(⚠️:【】标记处有坑,后面有解释和解决办法)
1. splash->首页->详情页->悬浮窗页->回到桌面->点击桌面App图标->【悬浮窗页】
2. splash->首页->详情页->悬浮窗页->开启悬浮窗->点击悬浮窗->悬浮窗页
2. splash->首页->详情页->悬浮窗页->开启悬浮窗->回到桌面->点击悬浮窗->悬浮窗页->【返回详情页】

#### 2.功能实现
- 在AndroidManifest.xml将悬浮窗页面android:launchMode="singleInstance"
- 添加权限

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

- 检查悬浮窗权限
- 点击开启,将悬浮窗页置于后台,同时添加小窗
```kotlin
moveTaskToBack(true)

var windowManager = context.getSystemService(Context.WINDOW_SERVICE) as WindowManager
windowManager.addView(...)
```
- 点击小窗将浮窗页置于前台

```kotlin
var intent = Intent(it, FloatWindowActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
it.startActivity(intent)
```
- 进入悬浮窗页面关闭小窗，保留悬浮窗页面

```kotlin
override fun onRestart() {
    super.onRestart()
    closeFloatWindow(this, exitFloatActivity = false)
}
```


- 点击关闭小窗, 同时将后台的悬浮窗页面finish

```kotlin
closeFloatWindow(this, exitFloatActivity = true)
```

#### 3.踩坑级解决方案
1. 功能预览中第一条，点击桌面App图标会显示详情页面，期望显示悬浮窗页
> 解释:

由于设置了悬浮窗页为singleInstance,所以默认打开的是路由栈A的最上面路由，而不是路由栈B的悬浮窗页面。
> 解决:

在悬浮窗页面监听app回到前台，如果不在小窗中，将该路由栈拉回前台

```kotlin
if (FloatWindowHelper.instance.dragFloatWrapper == null) {
    var intent = Intent(this, FloatWindowActivity::class.java)
        .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        .addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION)
    startActivity(intent)
}
```

2.功能预览中第三条，悬浮窗页面点击返回会回到桌面，期望回到详情页
> 解释:

上一步点击悬浮窗，路由跳转到悬浮窗页面，此时启动的只有路由栈B，所以返回就直接回到桌面了


>解决:

如果在桌面点击悬浮窗时,先启动app(默认路由栈A),这里有个小细节,会启动我们清单文件里设置了LAUNCHER的页面,即Splash页面,这里只需要finish掉不需要再自动跳到主页

```
// 先启动app,默认standard路由栈
if (isAppInBg) {
    context.let {
        val intent = it.packageManager.getLaunchIntentForPackage(it.packageName)
        it.startActivity(intent)
        // 告诉splash不需要跳转到主页,然后再恢复需要跳转
        needJumpToMain = false
        timer.schedule(object : TimerTask() {
            override fun run() {[link](https://note.youdao.com/)
                needJumpToMain = true
            }
        }, 1200)

    }
    isAppInBg = false
}
```
---
源码已上传至[github](https://github.com/qq326646683/FloatWindowDemo)

