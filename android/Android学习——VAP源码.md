### 一、背景介绍
##### 1. VAP（Video Animation Player）是直播中台使用的一个视频动画特效SDK，可以通过制作Alpha通道分离的视频素材，再在客户端上通过OpenGL ES重新实现Alpha通道和RGB通道的混合，从而实现在端上播放带透明通道的视频。
> 已经接入的app

![](http://file.jinxianyun.com/vap_apps.png)

> 同原理实现也用在其他app

     抖音、西瓜视频、今日头条、爱奇艺、比心等


##### 2. 方案对比
目前较常见的动画实现方案有帧动画、gif/webp、lottie/SVGA，对于复杂动画特效的实现做个简单对比

方案 | 文件大小 | 解码方式 | 特效支持 | 应用表现   
---|---|---|---|---
Lottie/SVGA | 无法导出 | 软解 | 部分复杂特效不支持 | 绘制耗时,内存抖动
gif/webp | 4.6M | 软解 | 只支持8位色彩 | 文件资源消耗大
apng | 10.6M | 软解 | 全支持 | 资源消耗大
vap | 1.5M | 硬解码 | 全支持 | 性能好,复杂动画支持好

##### 3. 运行效果
![image](http://file.jinxianyun.com/flutter_vap.gif)


##### 4. 本文主要内容

主要介绍VAP如何实现，从MediaCodec视频解码到OpenGL ES 渲染RGB及Alpha通道，最后输出到TextureView上。



### 二、VAP实现架构
##### 1. 需要的技术
- OpenGL ES: 创建纹理，及绘制工作
- TextureView: Android UI组件，持有SurfaceTexture，监听SurfaceTexture的size变化，做显示区域更新
- SurfaceTexture: 持有OpenGL ES创建的纹理，监听解码后的帧数据做帧更新，并通知OpenGL ES绘制
- MediaCodec: 解码，把解码后的数据绑定到SurfaceTexture

##### 2. 实现流程图
![image](http://file.jinxianyun.com/vap_main.jpg)
[image](http://file.jinxianyun.com/vap_main.jpg)


##### 3. 整体工作流程
为了方便查看各个类的职责及顺序执行的流程，这里用泳道图:
![image](http://file.jinxianyun.com/vap_codes.jpg)
[image](http://file.jinxianyun.com/vap_codes.jpg)


### 三、其他
- [本文源码](https://github.com/Tencent/vap)
- [字节对应实现](https://github.com/bytedance/AlphaPlayer)
- [flutter版vap](https://github.com/qq326646683/flutter_vap)


---
完结，撒花🎉