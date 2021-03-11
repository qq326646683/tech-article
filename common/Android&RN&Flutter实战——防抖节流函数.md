> ## 1.背景介绍

### 防抖
函数防抖，这里的抖动就是执行的意思，而一般的抖动都是持续的，多次的。假设函数持续多次执行，我们希望让它冷静下来再执行。也就是当持续触发事件的时候，函数是完全不执行的，等最后一次触发结束的一段时间之后，再去执行。


### 节流
节流的意思是让函数有节制地执行，而不是毫无节制的触发一次就执行一次。什么叫有节制呢？就是在一段时间内，只执行一次。
<br/><br/>
> ## 2.经典举例

1. 防抖函数：搜索页面,用户连续输入,等停下来再去触发搜索接口
2. 节流函数：防止按钮连点

<br/>

> ## 3.Android实现

1. 代码实现：

```kotlin
object FunctionUtil {
    private const val DEFAULT_DURATION_TIME = 300L
    var timer: Timer? = null


    /**
     * 防抖函数
     */
    fun debounce(duration: Long = DEFAULT_DURATION_TIME, doThing: () -> Unit) {
        timer?.cancel()
        timer = Timer().apply {
            schedule(timerTask {
                doThing.invoke()
                timer = null
            }, duration)
        }
    }

    /**
     * 节流函数
     */
    var lastTimeMill = 0L
    fun throttle(duration: Long = DEFAULT_DURATION_TIME, continueCall: (() -> Unit)? = null, doThing: () -> Unit) {
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastTimeMill > duration) {
            doThing.invoke()
            lastTimeMill = System.currentTimeMillis()
        } else {
            continueCall?.invoke()
        }
    }
}
```
2. 使用：
```kotlin
btn_sure.setOnClickListener {
    FunctionUtil.throttle {
        Log.i("nell-click", "hahah")
    }
}

btn_sure.setOnClickListener {
    FunctionUtil.throttle(500L) {
        Log.i("nell-click", "hahah")
    }
}

FunctionUtil.debounce {
	searchApi(text)
}
```

> ## 4.RN实现

1. 代码实现：
```javascript
/**
 * 防抖函数
 */
function debounce(func, delay) {
  let timeout
  return function() {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      func.apply(this, arguments)
    }, delay)
  }
}

/**
 * 节流函数
 */
function throttle(func, delay) {
    let run = true
    return function () {
      if (!run) {
        return
      }
      run = false // 持续触发的话，run一直是false，就会停在上边的判断那里
      setTimeout(() => {
        func.apply(this, arguments)
        run = true // 定时器到时间之后，会把开关打开，我们的函数就会被执行
      }, delay)
    }
}

```

2. 使用：

```javascript
throttle(function (e) {
  console.log("nell-click")
}, 1000)


debounce(function (e) {
    searchApi(text)
}, 300)
```


> ## 5.Flutter实现

1. 代码实现：
```dart
class CommonUtil {
  static const deFaultDurationTime = 300;
  static Timer timer;

  // 防抖函数
  static debounce(Function doSomething, {durationTime = deFaultDurationTime}) {
    timer?.cancel();
    timer = new Timer(Duration(milliseconds: durationTime), () {
      doSomething?.call();
      timer = null;
    });
  }

  // 节流函数
  static const String deFaultThrottleId = 'DeFaultThrottleId';
  static Map<String, int> startTimeMap = {deFaultThrottleId: 0};
  static throttle(Function doSomething, {String throttleId = deFaultThrottleId, durationTime = deFaultDurationTime, Function continueClick}) {
    int currentTime = DateTime.now().millisecondsSinceEpoch;
    if (currentTime - (startTimeMap[throttleId] ?? 0) > durationTime) {
      doSomething?.call();
      startTimeMap[throttleId] = DateTime.now().millisecondsSinceEpoch;
    } else {
      continueClick?.call();
    }
  }
  
}
```
2. 使用：

```dart
GestureDetector(
      onTap: () => CommonUtil.throttle(onTap, durationTime: durationTime)
)


CommonUtil.debounce(searchApi)
```


---
完结，撒花🎉








