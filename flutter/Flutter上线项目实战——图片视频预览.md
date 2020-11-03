# interactiveviewer_gallery
[![pub package](https://img.shields.io/pub/v/interactiveviewer_gallery.svg)](https://pub.dartlang.org/packages/interactiveviewer_gallery)

A flutter library to show picture and video preview gallery
support
1. two-finger gesture zoom
2. double-click to zoom
3. switch left and right
4. gesture back: scale, transfer, opacity of background
5. video auto paused when miss focus

## Preview
[video for youtube](https://youtu.be/S-93Et_nYQs)

[video for qiniu](http://file.jinxianyun.com/interactiveviewer_gallery_0_1_0.mp4)

[apk download](http://file.jinxianyun.com/interactiveviewer_gallery_0_1_0.apk)

## Setup

because the library is base on InteractiveViewer so require flutter verion above or equal 1.20.0
```dart
interactiveviewer_gallery: ${last_version}
```

## How to use

1. Wrap Hero in your image gridview item:
```dart
Hero(
    tag: source.url,
    child: ${gridview item}
)
 ```

2. gridview item's GestureDetector add jumping to interactiveviewer_gallery:
```dart
// DemoSourceEntity is your data model
// itemBuilder is gallery page item
// heroTagBuilder accordding to gridview item's hero tag
void _openGallery(DemoSourceEntity source) {
    Navigator.of(context).push(
      HeroDialogRoute<void>(
        builder: (BuildContext context) => InteractiveviewerGallery<DemoSourceEntity>(
          sources: sourceList,
          initIndex: sourceList.indexOf(source),
          itemBuilder: itemBuilder,
          heroTagBuilder: (int index) => sourceList[index].url,
        ),
      ),
    );
}
```

3. edit itemBuilder: you can reference the [example/lib/main.dart](https://github.com/qq326646683/interactiveviewer_gallery/blob/main/example/lib/main.dart) then customize

```dart
Widget itemBuilder(BuildContext context, int index, bool isFocus) {
    DemoSourceEntity sourceEntity = sourceList[index];
    if (sourceEntity.type == 'video') {
      return DemoVideoItem(
        sourceEntity.url,
        sourceEntity.previewUrl,
        isFocus: isFocus,
      );
    } else {
      return DemoImageItem(sourceEntity.url);
    }
}
```

## Other
Comments and pr are welcome
