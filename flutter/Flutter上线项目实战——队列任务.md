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

> TaskQueueUtil.dart

```dart
class TaskQueueUtil {
  static TaskQueueUtil _instance;

  static TaskQueueUtil getInstance() {
    if (_instance == null) {
      _instance = TaskQueueUtil._();
    }
    return _instance;
  }

  TaskQueueUtil._() {
    /// 初始化代码
  }

  List<TaskInfo> taskList = [];
  bool isTaskRunning = false;

  Future<TaskInfo> addTask(TaskInfo taskInfo) {
    /// 创建future
    Completer<TaskInfo> taskCompleter = Completer<TaskInfo>();

    /// 创建当前任务stream
    StreamController<TaskInfo> streamController = new StreamController();
    taskInfo.controller = streamController;

    /// 添加到任务队列
    taskList.add(taskInfo);

    /// 当前任务的stream添加监听
    streamController.stream.listen((TaskInfo completeTaskInfo) {
      if (completeTaskInfo.id == taskInfo.id) {
        taskCompleter.complete(completeTaskInfo);
        streamController.close();
      }
    });

    /// 触发任务
    doTask();

    return taskCompleter.future;
  }

  doTask() async {
    if (isTaskRunning) return;
    if (taskList.isEmpty) return;

    /// 取任务
    TaskInfo taskInfo = taskList[0];
    isTaskRunning = true;

    /// 模拟执行任务
    await Future.delayed(Duration(seconds: 2));
    taskInfo.content += 'ed';
    taskInfo.controller.sink.add(taskInfo);


    /// 出队列
    taskList.removeAt(0);
    isTaskRunning = false;

    /// 递归执行任务
    doTask();
  }

}


class TaskInfo {
  int id; // 任务唯一标识
  String content;
  StreamController<TaskInfo> controller;

  TaskInfo(this.id, this.content, {this.controller});

  @override
  String toString() {
    return 'TaskInfo{id: $id, content: $content, controller: $controller}';
  }
}
```

## 四、使用

```dart
main() {
    TaskInfo runTask = new TaskInfo(1, 'run');
    TaskInfo playTask = new TaskInfo(2, 'play');
    TaskInfo swimTask = new TaskInfo(3, 'swim');
    
    task(runTask);
    task(playTask);
    task(swimTask);
}



task1() async {
  TaskInfo taskInfo = await TaskQueueUtil.getInstance().addTask(runTask);
  debugPrint('task${taskInfo.id}-result:$taskInfo');
}


// 执行结果(每两秒打印)：
// flutter: task1-result:TaskInfo{id: 1, content: runed, controller: Instance of '_AsyncStreamController<TaskInfo>'}
// flutter: task2-result:TaskInfo{id: 2, content: played, controller: Instance of '_AsyncStreamController<TaskInfo>'}
// flutter: task3-result:TaskInfo{id: 3, content: swimed, controller: Instance of '_AsyncStreamController<TaskInfo>'}
```


---
完结，撒花🎉

 
