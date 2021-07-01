## 1. 应用场景
>开发中经常遇到

- 路由跳转时拿不到context怎么办，eg: token失效/异地登录跳转登录页面。
- 获取不到当前路由名称怎么办，eg: 点击push推送跳转指定路由，如果已经在当前页面就replace，如果不在就push。
- 注册监听路由跳转,做一些想做的事情，eg：不同路由，显示不同状态栏颜色。
- 监听当前页面获取、失去焦点
- 等等...


## 2. 解决方案
>解决思路：

1. MaterialApp 的routes属性赋值路由数组，navigatorObservers属性赋值路由监听对象NavigationUtil。
2. 在NavigationUtil里实现NavigatorObserver的didPush/didReplace/didPop/didRemove，并记录到路由栈
List<Route> _mRoutes中。
3. 将实时记录的路由跳转，用stream发一个广播，哪里需要哪里注册。
4. 用mixin实现当前页面获取、失去焦点，监听当前路由变化，触发onFocus,onBlur。


## 3. 具体实现
>支持空安全版本 [戳我](https://github.com/qq326646683/flutter_collection_demo/tree/main/lib/util/navigation)

>main.dart
  
``` dart
MaterialApp(
    navigatorObservers: [NavigationUtil.getInstance()],
    routes: NavigationUtil.configRoutes,
    ...
)
```
>navigation_util.dart
  
``` dart
class RouteInfo {
  Route currentRoute;
  List<Route> routes;

  RouteInfo(this.currentRoute, this.routes);

  @override
  String toString() {
    return 'RouteInfo{currentRoute: $currentRoute, routes: $routes}';
  }
}

class NavigationUtil extends NavigatorObserver {
  static NavigationUtil _instance;

  static Map<String, WidgetBuilder> configRoutes = {
    SplashPage.sName: (_) => SplashPage(),
    ScanQrPage.sName: (_) => ScanQrPage(),
  };

  ///路由信息
  RouteInfo _routeInfo;
  RouteInfo get routeInfo => _routeInfo;
  ///stream相关
  static StreamController _streamController;
  StreamController<RouteInfo> get streamController=> _streamController;
  ///用来路由跳转
  static NavigatorState navigatorState;


  static NavigationUtil getInstance() {
    if (_instance == null) {
      _instance = new NavigationUtil();
      _streamController = StreamController<RouteInfo>.broadcast();
    }
    return _instance;
  }

  ///push页面
  Future<T> pushNamed<T>(String routeName, {WidgetBuilder builder, bool fullscreenDialog}) {
    return navigatorState.push<T>(
      MaterialPageRoute(
        builder: builder ?? configRoutes[routeName],
        settings: RouteSettings(name: routeName),
        fullscreenDialog: fullscreenDialog ?? false,
      ),
    );
  }

  ///replace页面
  Future<T> pushReplacementNamed<T, R>(String routeName, {WidgetBuilder builder, bool fullscreenDialog}) {
    return navigatorState.pushReplacement<T, R>(
      MaterialPageRoute(
        builder: builder ?? configRoutes[routeName],
        settings: RouteSettings(name: routeName),
        fullscreenDialog: fullscreenDialog ?? false,
      ),
    );
  }

  /// pop 页面
  pop<T>([T result]) {
    navigatorState.pop<T>(result);
  }

  pushNamedAndRemoveUntil(String newRouteName) {
    return navigatorState.pushNamedAndRemoveUntil(newRouteName, (Route<dynamic> route) => false);
  }

  @override
  void didPush(Route route, Route previousRoute) {
    super.didPush(route, previousRoute);
    if (_routeInfo == null) {
      _routeInfo = new RouteInfo(null, new List<Route>());
    }
    ///这里过滤调push的是dialog的情况
    if (route is CupertinoPageRoute || route is MaterialPageRoute) {
      _routeInfo.routes.add(route);
      routeObserver();
    }
  }

  @override
  void didReplace({Route newRoute, Route oldRoute}) {
    super.didReplace();
    if (newRoute is CupertinoPageRoute || newRoute is MaterialPageRoute) {
      _routeInfo.routes.remove(oldRoute);
      _routeInfo.routes.add(newRoute);
      routeObserver();
    }
  }

  @override
  void didPop(Route route, Route previousRoute) {
    super.didPop(route, previousRoute);
    if (route is CupertinoPageRoute || route is MaterialPageRoute) {
      _routeInfo.routes.remove(route);
      routeObserver();
    }
  }

  @override
  void didRemove(Route removedRoute, Route oldRoute) {
    super.didRemove(removedRoute, oldRoute);
    if (removedRoute is CupertinoPageRoute || removedRoute is MaterialPageRoute) {
      _routeInfo.routes.remove(removedRoute);
      routeObserver();
    }
  }



  void routeObserver() {
    _routeInfo.currentRoute = _routeInfo.routes.last;
    navigatorState = _routeInfo.currentRoute.navigator;
    _streamController.sink.add(_routeInfo);
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
    NavigationUtil.getInstance().pushNamedAndRemoveUntil(LoginPage.sName);
    break;
```

>点击push推送跳转
  
``` dart
static jumpPage(String pageName, [WidgetBuilder builder]) {
    String currentRouteName = NavigationUtil.getInstance().currentRoute.settings.name;
    // 如果是未登录，不跳转
    if (NavigationUtil.getInstance().routes[0].settings.name != MainPage.sName) {
      return;
    }

    // 如果已经是当前页面就replace
    if (currentRouteName == pageName) {
      NavigationUtil.getInstance().pushReplacementNamed(pageName, builder);
    } else {
      NavigationUtil.getInstance().pushNamed(pageName, builder);
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
        NavigationUtil.getInstance().streamController.stream.listen((state) {
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





