# 例子 1、
#### 现有一个滑动聊天页面出现/隐藏jump按钮的需求:

![image](http://file.jinxianyun.com/flutter_optimize1.gif)
[🔗](http://file.jinxianyun.com/flutter_optimize1.gif)


> 1. 优化前：ListView外层setState实现:

```dart
class _ChatListViewWidgetState extends State<ChatListViewWidget> {
  ScrollController scrollController;
  double _chatListOffset = 0;
  
    @override
  void initState() {
    super.initState();

    scrollController = new ScrollController()
      ..addListener(() {
        setState(() {
          _chatListOffset = scrollController.offset;
        });
      });
  }
    
  @override
  Widget build(BuildContext context) {
    return Stack(
      alignment: Alignment.bottomCenter,
      children: <Widget>[
        ListView.builder(
          controller: _scrollController,
          ...
        ),
        _chatListOffset > 150 ? Image.asset('jump.png') : SizedBox()
      ],
    );
  }
}

```

> 2. 优化后：在JumpButton内部setState实现

``` dart
class JumpButton extends StatefulWidget {
  ScrollController scrollController;

  JumpButton({this.scrollController});

  @override
  _JumpButtonState createState() => _JumpButtonState();
}

class _JumpButtonState extends State<JumpButton> {
  double _chatListOffset = 0;

  @override
  void initState() {
    super.initState();
    widget.scrollController.addListener(() {
      setState(() {
        _chatListOffset = widget.scrollController.offset;
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    return _chatListOffset > 150 ? Image.asset('jump.png') : SizedBox();
  }
}

```
> 3. 开启 DevTools 的 Repaint RainBow差别:

![image](http://file.jinxianyun.com/flutter_optimize2.gif)
[🔗](http://file.jinxianyun.com/flutter_optimize2.gif)

###### 左边是优化前: 发现在滑动过程中，ListView在不断的绘制，右边为优化后，在按钮出现的时候绘制一次

> 4. 思考与总结:
###### setState会导致当前页面根节点开始遍历，所以我们要在需要重新渲染的Widget内部进行setState更新自己，可以有效防止多余的树节点遍历。参考flutter官方的性能测试视频也有相应的讲解：https://www.bilibili.com/video/BV1F4411D7rP 的13’开始



# 例子 2、
#### 现有一个在聊天页面上层悬浮一个跑马灯的需求:

> 1. 功能实现:

``` dart
// 聊天页面
Stack(
      alignment: Alignment.bottomCenter,
      children: <Widget>[
        ListView.builder(...),
        /// 跑马灯
        MMarqueeWidget(),
      ],
)
```

###### 参考徐医生的[dojo](https://github.com/xuyisheng/flutter_dojo/blob/a0020faa635022ba8a2c47b41bf0668a27e2d1f3/lib/category/pattern/texteffect/marquee.dart)
``` dart
// 跑马灯组件实现，
import 'package:flutter/material.dart';

class MMarqueeWidget extends StatefulWidget {
  final Widget child;

  MMarqueeWidget({Key key, this.child}) : super(key: key);

  @override
  _MMarqueeWidgetState createState() => _MMarqueeWidgetState();
}

class _MMarqueeWidgetState extends State<MMarqueeWidget> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<Offset> animation;

  @override
  void initState() {
    super.initState();
    controller = AnimationController(vsync: this, duration: Duration(seconds: 10));
    animation = Tween<Offset>(begin: Offset(1, 0), end: Offset(-1, 0)).animate(controller);
    controller.repeat();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: animation,
      builder: (context, _) {
        return ClipRect(
          child: FractionalTranslation(
            translation: animation.value,
            child: SingleChildScrollView(
              scrollDirection: Axis.horizontal,
              child: widget.child,
            ),
          ),
        );
      },
    );
  }

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```

> 2. 开启 DevTools 的 Repaint RainBow

![image](http://file.jinxianyun.com/flutter_optimize3.gif)
[🔗](http://file.jinxianyun.com/flutter_optimize3.gif)

###### 发现跑马灯动画会导致其他部分也重绘

> 3.  RepaintBoundary优化

```dart
  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: AnimatedBuilder(
        animation: animation,
        ...
      ),
    );
  }
```

###### flutter提供了创建一个单独的图层，不会影响其他图层，参考https://www.bilibili.com/video/BV1F4411D7rP的24’20’’或者唯鹿的这篇https://www.jianshu.com/p/99d8a42f6704



---
❤️未完待续...