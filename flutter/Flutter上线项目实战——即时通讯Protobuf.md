## 一、应用背景：
[Protobuf](https://github.com/protocolbuffers/protobuf)是google 的一种数据交换的格式，它独立于语言，独立于平台。


优点：

- json优点就是较XML格式更加小巧，传输效率较xml提高了很多，可读性还不错。
- xml优点就是可读性强，解析方便。
- protobuf优点就是传输效率快（据说在数据量大的时候，传输效率比xml和json快10-20倍），序列化后体积相比Json和XML很小，支持跨平台多语言，消息格式升级和兼容性还不错，序列化反序列化速度很快。

缺点：

- json缺点就是传输效率也不是特别高（比xml快，但比protobuf要慢很多）。
- xml缺点就是效率不高，资源消耗过大。
- protobuf缺点就是使用不太方便。

在一个需要大量的数据传输的场景中，如果数据量很大，那么选择protobuf可以明显的减少数据量，减少网络IO，从而减少网络传输所消耗的时间。考虑到作为一个主打社交的产品，消息数据量会非常大，同时为了节约流量，所以采用protobuf是一个不错的选择。



## 二、使用

### 1.引入protobuf库

> pubspec.yaml

```dart
...

protobuf: 1.0.1

```

### 2.编写proto文件
> socket.message.proto

```dart
syntax = "proto3";
package socket;

// 发送聊天信息
message Message {
  string eventId = 1;
  string from = 2;
  string to = 3;
  string createAt = 4;
  string type = 5;
  string body = 6;
}

// 收到聊天消息
message AckMessage {
  string eventId = 1;
}
```
### 3.生成proto相关Model
> Terminal
```
protoc --dart_out=. socket.message.proto
```

### 4.编码、发消息
> a.准备protobuf对象

```dart
Message message = Message();
message.eventId = '####';
message.type = 'text';
message.body = 'hello world';
```

> b.ProtobufUtil编码

```dart
const MESSAGE_HEADER_LEN = 2;
/// 数据编码
static List<int> encode(int type, var content) {
    ByteData data = ByteData(MESSAGE_HEADER_LEN);
    data.setUint16(0, type, Endian.little);
    List<int> msg = data.buffer.asUint8List() + content.writeToBuffer().buffer.asUint8List();
    return msg;
}

```
> c.发消息

```dart
/// 发送
sendSocket(int type, var content) async {
    IOWebSocketChannel channel = await SocketService.getInstance().getChannel();
    if (channel == null) return;
    List<int> msg = ProtobufUtil.encode(type, content);
    channel.sink.add(msg);
}

sendSocket(11, message)
```



### 5.收消息、解码

> a.解码

```dart
  /// 数据解码
  static DecodedMsg decode(data) {
    Int8List int8Data = Int8List.fromList(data);
    Int8List contentTypeInt8Data = int8Data.sublist(0, MESSAGE_HEADER_LEN);
    Int8List contentInt8Data = int8Data.sublist(MESSAGE_HEADER_LEN, int8Data.length);
    int contentType = contentTypeInt8Data.elementAt(0);


    GeneratedMessage content;
    switch (contentType) {
      case 10:
        content = AckMessage.fromBuffer(contentInt8Data);
        break;
      case 11:
        content = Message.fromBuffer(contentInt8Data);
        break;
    }

    DecodedMsg decodedMsg;
    if (contentType != null && content != null) {
      decodedMsg = DecodedMsg(
        contentType: contentType,
        content: content,
      );
    }
    return decodedMsg;
  }
  

```

> b.收消息

```dart
  channel.stream.listen((data) {
    DecodedMsg msg = ProtobufUtil.decode(data);
  }

```

---
完结，撒花🎉