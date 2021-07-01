## 一、应用场景

- 队列压缩视频
- 队列解密视频
- 队列请求网络
- 等等

## 二、实现思路

1. 定义一个任务队列taskList [先进先出]
2. 提供添加任务方法
3. 取第一个任务执行
4. 执行完后，从taskList移除
5. 递归获取第一个任务并执行任务
6. 直到taskList为空停止队列任务

## 三、具体实现
>支持空安全版本 [戳我](https://github.com/qq326646683/flutter_vap/blob/main/lib/queue_util.dart)

> QueueUtil.dart

```dart
class QueueUtil {
  /// 用map key存储多个QueueUtil单例,目的是隔离多个类型队列任务互不干扰
  /// Use map key to store multiple QueueUtil singletons, the purpose is to isolate multiple types of queue tasks without interfering with each other
  static Map<String, QueueUtil> _instance = Map<String, QueueUtil>();

  static QueueUtil get(String key) {
    if (_instance[key] == null) {
      _instance[key] = QueueUtil._();
    }
    return _instance[key];
  }

  QueueUtil._() {
    /// 初始化代码
  }

  List<_TaskInfo> _taskList = [];
  bool _isTaskRunning = false;
  int _mId = 0;
  bool _isCancelQueue = false;

  Future<_TaskInfo> addTask(Function doSomething) {
    _isCancelQueue = false;
    _mId++;
    _TaskInfo taskInfo = _TaskInfo(_mId, doSomething);

    /// 创建future
    Completer<_TaskInfo> taskCompleter = Completer<_TaskInfo>();

    /// 创建当前任务stream
    StreamController<_TaskInfo> streamController = new StreamController();
    taskInfo.controller = streamController;

    /// 添加到任务队列
    _taskList.add(taskInfo);

    /// 当前任务的stream添加监听
    streamController.stream.listen((_TaskInfo completeTaskInfo) {
      if (completeTaskInfo.id == taskInfo.id) {
        taskCompleter.complete(completeTaskInfo);
        streamController.close();
      }
    });

    /// 触发任务
    _doTask();

    return taskCompleter.future;
  }

  void cancelTask() {
    _taskList = [];
    _isCancelQueue = true;
    _mId = 0;
    _isTaskRunning = false;
  }

  _doTask() async {
    if (_isTaskRunning) return;
    if (_taskList.isEmpty) return;

    /// 取任务
    _TaskInfo taskInfo = _taskList[0];
    _isTaskRunning = true;

    /// 模拟执行任务
    await taskInfo.doSomething?.call();

    taskInfo.controller.sink.add(taskInfo);

    if (_isCancelQueue) return;

    /// 出队列
    _taskList.removeAt(0);
    _isTaskRunning = false;

    /// 递归执行任务
    _doTask();
  }
}

class _TaskInfo {
  int id; // 任务唯一标识
  Function doSomething;
  StreamController<_TaskInfo> controller;

  _TaskInfo(this.id, this.doSomething, {this.controller});
}
```

## 四、使用

```dart
main() {
      /// 将任务添加到队列
      print("加入队列-net, taskNo: 1");
      QueueUtil.get("net").addTask(() {
        return _doFuture("net", 1);
      });
      print("加入队列-net, taskNo: 2");
      QueueUtil.get("net").addTask(() {
        return _doFuture("net", 2);
      });
      print("加入队列-local, taskNo: 1");
      QueueUtil.get("local").addTask(() {
        return _doFuture("local", 1);
      });
      
      
      
      /// 取消队列任务
      /// QueueUtil.get("net").cancelTask();
}



  Future _doFuture(String queueName, int taskNo) {
    return Future.delayed(Duration(seconds: 2), () {
      print("任务完成  queueName: $queueName, taskNo: $taskNo");
    });
  }


// 执行结果：
I/flutter (26436): 加入队列-net, taskNo: 1
I/flutter (26436): 加入队列-net, taskNo: 2
I/flutter (26436): 加入队列-local, taskNo: 1
------------两秒后--------
I/flutter (26436): 任务完成  queueName: net, taskNo: 1
I/flutter (26436): 任务完成  queueName: local, taskNo: 1
------------两秒后--------
I/flutter (26436): 任务完成  queueName: net, taskNo: 2
```


---
完结，撒花🎉

 
