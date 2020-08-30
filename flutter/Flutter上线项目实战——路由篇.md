## 1. 应用场景
>开发中经常遇到

- 路由跳转时拿不到context怎么办，eg: token失效/异地登录跳转登录页面。
- 获取不到当前路由名称怎么办，eg: 点击push推送跳转指定路由，如果已经在当前页面就replace，如果不在就push。
- 注册监听路由跳转,做一些想做的事情，eg：不同路由，显示不同状态栏颜色。
- 监听当前页面获取、失去焦点
- 等等...


## 2. 解决方案
>解决思路：

1. MaterialApp 的routes属性赋值路由数组，navigatorObservers属性赋值路由监听对象NavigatorManager。
2. 在NavigatorManager里实现NavigatorObserver的didPush/didReplace/didPop/didRemove，并记录到路由栈
List<Route> _mRoutes中。
3. 将实时记录的路由跳转，用stream发一个广播，哪里需要哪里注册。
4. 用mixin实现当前页面获取、失去焦点，监听当前路由变化，触发onFocus,onBlur。


## 3. 具体实现
>main.dart
  
``` dart
MaterialApp(
    navigatorObservers: [NavigatorManager.getInstance()],
    routes: NavigatorManager.configRoutes,
    ...
)
```
>navigator_manager.dart
  
``` dart
class NavigatorManager extends NavigatorObserver {
  /* 配置routes */
  static Map<String, WidgetBuilder> configRoutes = {
  PackageInfoPage.sName: (context) =>
    SplashPage.sName: (context) => SplashPage(),
    LoginPage.sName: (context) => SplashPage()),
    MainPage.sName: (context) => SplashPage(),
    //...
  }
  // 当前路由栈
  static List<Route> _mRoutes;
  List<Route> get routes => _mRoutes;
  // 当前路由
  Route get currentRoute => _mRoutes[_mRoutes.length - 1];
  // stream相关
  static StreamController _streamController;
  StreamController get streamController=> _streamController;
  // 用来路由跳转
  static NavigatorState navigator;
  
  /* 单例给出NavigatorManager */
  static NavigatorManager navigatorManager;
  static NavigatorManager getInstance() {
    if (navigatorManager == null) {
      navigatorManager = new NavigatorManager();
      _streamController = StreamController.broadcast();
    }
    return navigatorManager;
  }
  
  // replace 页面
  pushReplacementNamed(String routeName, [WidgetBuilder builder]) {
    return navigator.pushReplacement(
      CupertinoPageRoute(
        builder: builder ?? configRoutes[routeName],
        settings: RouteSettings(name: routeName),
      ),
    );
  }
  
  // push 页面
  pushNamed(String routeName, [WidgetBuilder builder]) {
    return navigator.push(
      CupertinoPageRoute(
        builder: builder ?? configRoutes[routeName],
        settings: RouteSettings(name: routeName),
      ),
    );
  }
  
  // pop 页面
  pop<T extends Object>([T result]) {
    navigator.pop(result);
  }
  
  // push一个页面， 移除该页面下面所有页面
  pushNamedAndRemoveUntil(String newRouteName) {
    return navigator.pushNamedAndRemoveUntil(newRouteName, (Route<dynamic> route) => false);
  }
  
  // 当调用Navigator.push时回调
  @override
  void didPush(Route route, Route previousRoute) {
    super.didPush(route, previousRoute);
    if (_mRoutes == null) {
      _mRoutes = new List<Route>();
    }
    // 这里过滤调push的是dialog的情况
    if (route is CupertinoPageRoute || route is MaterialPageRoute) {
      _mRoutes.add(route);
      routeObserver();
    }
  }
  
  // 当调用Navigator.replace时回调
  @override
  void didReplace({Route newRoute, Route oldRoute}) {
    super.didReplace();
    if (newRoute is CupertinoPageRoute || newRoute is MaterialPageRoute) {
      _mRoutes.remove(oldRoute);
      _mRoutes.add(newRoute);
      routeObserver();
    }
  }
  
  // 当调用Navigator.pop时回调
  @override
  void didPop(Route route, Route previousRoute) {
    super.didPop(route, previousRoute);
    if (route is CupertinoPageRoute || route is MaterialPageRoute) {
      _mRoutes.remove(route);
      routeObserver();
    }
  }
  
  @override
  void didRemove(Route removedRoute, Route oldRoute) {
    super.didRemove(removedRoute, oldRoute);
    if (removedRoute is CupertinoPageRoute || removedRoute is MaterialPageRoute) {
      _mRoutes.remove(removedRoute);
      routeObserver();
    }
  }
  
  void routeObserver() {
    LogUtil.i(sName, '&&路由栈&&');
    LogUtil.i(sName, _mRoutes);
    LogUtil.i(sName, '&&当前路由&&');
    LogUtil.i(sName, _mRoutes[_mRoutes.length - 1]);
    // 当前页面的navigator，用来路由跳转
    navigator = _mRoutes[_mRoutes.length - 1].navigator;
    streamController.sink.add(_mRoutes);
  }
}
```
>navigation_mixin.dart
  
``` dart
mixin NavigationMixin<T extends StatefulWidget> on State<T> {
  StreamSubscription<RouteInfo> streamSubscription;
  Route lastRoute;

  @override
  void initState() {
    super.initState();

    streamSubscription = NavigationUtil.getInstance().streamController.stream.listen((RouteInfo routeInfo) {
      if (routeInfo.currentRoute.settings.name == routName) {
        onFocus();
      }
      /// 第一次监听到路由变化
      if (lastRoute == null) {
        onBlur();
      }
      /// 上一个是该页面，新的路由不是该页面
      if (lastRoute?.settings?.name == routName && routeInfo.currentRoute.settings.name != routName) {
        onBlur();
      }
      lastRoute = routeInfo.currentRoute;

    });
  }

  @override
  void dispose() {
    super.dispose();
    streamSubscription?.cancel();
    streamSubscription = null;
  }

  @protected
  String get routName;

  @protected
  void onBlur() {

  }

  @protected
  void onFocus() {

  }
}
```
  
## 4. 如何使用
>token失效跳转
  
``` dart
case 401:
    ToastUtil.showRed('登录失效,请重新登陆');
    UserDao.clearAll();
    NavigatorManager.getInstance().pushNamedAndRemoveUntil(LoginPage.sName);
    break;
```

>点击push推送跳转
  
``` dart
static jumpPage(String pageName, [WidgetBuilder builder]) {
    String currentRouteName = NavigatorManager.getInstance().currentRoute.settings.name;
    // 如果是未登录，不跳转
    if (NavigatorManager.getInstance().routes[0].settings.name != MainPage.sName) {
      return;
    }

    // 如果已经是当前页面就replace
    if (currentRouteName == pageName) {
      NavigatorManager.getInstance().pushReplacementNamed(pageName, builder);
    } else {
      NavigatorManager.getInstance().pushNamed(pageName, builder);
    }
}
```
>监听路由改变状态栏颜色
  
``` dart
class StatusBarUtil {
      static List<String> lightRouteNameList = [
        TaskhallPage.sName,
        //...
      ];
      static List darkRoutNameList = [
        SplashPage.sName,
        LoginPage.sName,
        MainPage.sName,
        //...
      ];
      
      static init() {
        NavigatorManager.getInstance().streamController.stream.listen((state) {
            setupStatusBar(state[state.length - 1]);
        })
      }
    
      setupStatusBar(Route currentRoute) {
        if (lightRouteNameList.contains(currentRoute.settings.name)) {
          setLight();
        } else if (darkRoutNameList.contains(currentRoute.settings.name)) {
          setDart();
        }
      }
}
```
>当前页面获取、失去焦点
  
``` dart
class _ChatPageState extends State<ChatPage> with NavigationMixin<ChatPage> {
  ...
  @override
  String get routName => ChatPage.sName;
  
  @override
  void onBlur() {
  	super.onBlur();
  	// do something
  }
  
  @override
  void onFocus() {
  	super.onFocus();
  	// do something
  }
}
  
```


---
#####  完结，撒花🎉





