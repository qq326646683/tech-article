### 一、前言
现在市面上很多app有游戏中心功能，最早的有微信小游戏和QQ小游戏，再后来像bilibili、喜马拉雅、爱奇艺、比心等等应用中也加入了游戏中心模块。本篇文章将介绍如何上手搭建cocos creater游戏容器，先来看看效果：

![image](http://file.jinxianyun.com/cocos_android.gif)

### 二、准备工作
1. [下载](https://www.cocos.com/creator)并安装最新版本CocosDashboard
2. 在Dashborad下载最新版本编辑器

![image](http://file.jinxianyun.com/cocos_creator_install.jpg)

3. 在Android Studio安装NDK，我这里安装的是21.1.6352462，目前为止比较稳定

![image](http://file.jinxianyun.com/ndk_install.png)

4. 在CocosDashboard新建HelloWorld项目并打开运行，我这里用的3.1.1版本

![image](http://file.jinxianyun.com/cocos_hellerun.png)

5. 打开CocosCreator菜单栏偏好设置，在外部程序栏中设置Android NDK和Android SDK路径

![image](http://file.jinxianyun.com/cococreator_setup.jpg)

### 三、构建cocos游戏.so文件
1. 在CocosCreator菜单栏选择项目-构建发布，选择发布平台：安卓，点击构建，等大概几分钟

![image](http://file.jinxianyun.com/cocos_build.png)

2. 成功后，用Android Studio打开文件夹里生成的proj项目，并运行该项目到手机上，这里游戏资源加载的是proj同级目录assets，后续，我们会将assets压缩包zip存放在我们服务器，达到用户下载解压后加载启动游戏的目的。

3. 为了后续游戏容器能加载本地filePath下的游戏资源，需要修改JniCocosActivity.cpp里的Java_com_cocos_lib_CocosActivity_onCreateNative方法

![image](http://file.jinxianyun.com/cocos_modify.png)


4. ./gradlew assembleRelease打release包, 将instantapp-release.apk后缀改成zip，解压后获取lib下arm64-v8a/armeabi-v7a下的libcocos.so（构建版本设置那里可以勾选不同架构）


### 四、制作自己的游戏容器
1. 创建module，包名为com.cocos.lib（为了和.so文件里保持一致，不然无法调用c方法）

2. module的清单文件加
```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

3. 将/Applications/CocosCreator/Creator/3.1.1/CocosCreator.app/Contents/Resources/resources/3d/engine-native/cocos/platform/android/java/libs拷贝到module/libs下

4. module下build.gradle添加
```
implementation fileTree(include: ['*.jar'], dir: 'libs')
```

5. 将.so文件放在module/src/main/jniLibs/下
6. 将/Applications/CocosCreator/Creator/3.1.1/CocosCreator.app/Contents/Resources/resources/3d/engine-native/cocos/platform/android/java/src/com/cocos/lib下的java文件复制到module/src/main/java/com.cocos.lib下
7. 修改文件CocosActivity.java，因为游戏页面官方推荐用多进程来做，所以这里退出游戏，即将游戏进程kill
```java
// 加一个filePath参数
private native void onCreateNative(Activity activity, AssetManager assetManager, String obbPath, int sdkVersion, String filePath);

// 外部传入游戏资源路径
protected String filePath() {
    return "";
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    onCreateNative(this, getAssets(), getAbsolutePath(getObbDir()), Build.VERSION.SDK_INT, filePath());
}

@Override
public void onBackPressed() {
    super.onBackPressed();
    System.exit(0);
}
```
### 五、总结
自此，我们游戏容器制作完毕，我也将该篇的游戏容器module传到了jitpack，可以直接使用:
```
allprojects {
        repositories {
            ...
            maven { url 'https://jitpack.io' }
        }
}
```
```
dependencies {
       implementation 'com.github.qq326646683:cocos-creator-android:1.0.0'
}
```

### 六、如何使用
1. 文件读写、网络权限
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.DOWNLOAD_WITHOUT_NOTIFICATION" />
```
2. 下载游戏zip并解压
3. 继承CocosActivity，并将解压后的路径赋值给filePath
```kotlin
class CocosGameActivity: CocosActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }

    override fun filePath() = intent.getStringExtra("path")
}
```
4. 清单文件
```
 <application>
        <meta-data
            android:name="android.app.lib_name"
            android:value="cocos" />
        <activity android:name=".CocosGameActivity" android:process=":cocos"/>
```

5. 本篇的module和事例app代码放在[gitlab](https://github.com/qq326646683/cocos-creator-android)

### 七、后续计划
cocos游戏和android通信，因为牵扯到多进程，通信变的麻烦，后续计划将这部分内容封装在module library，方便使用者调用


---
完结，撒花🎉
