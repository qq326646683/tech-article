> ### 一、以下是消耗类和非消耗类的正常流程（订阅类的不太清楚）
- 1.进入充值页面，向app server获取productIdList及展示信息。
- 2.用productIdList调iapsdk获取productDetailList（用来发起支付的参数）。
- 3.用户选择一个productDetail，然后调iap sdk发起支付。
- 4.监听到apple支付成功，将purshaseId、receipt发给app server。
- 5.app server 向apple server发起校验请求，比对in_app数组里对应的purshaseId的校验结果，返回给第4步app的请求。
- 6.app端收到成功结果后finish掉该productDetail。

> ### 二、以下是应对异常情况
- a.上面第4步断网或者app闪退。
- b.上面第6步因为网络原因没有finish该productDetail。
###### 针对第a种异常：
- i.下次进入app用iap sdk获取未处理(未finish)栈里的productDetail(这个flutter iap plugin没有提供方法，我自己fork后加了[该方法](https://github.com/qq326646683/plugins))（注意这里是单数，只有一个未处理），然后接着走正常流程的4、5、6。
- ii.再次购买时，先执行i的步骤，确保处理完毕了才能发起第二笔支付，否则获取未处理productDetail为复数时会导致receipt紊乱导致校验失败造成卡单（未处理栈里一直在）。
###### 针对第b种异常：
- a的异常处理会让app server重复校验，所以这里需要app server做一下记录，校验过的结果存在数据库里，再发起该purshaseId校验直接返回结果，避免重复增加余额。

> ### 三、编码参考

环境：
```
flutter版本: v1.9.1+hotfix.4
插件依赖：in_app_purchase:
            git:
              url: https://github.com/qq326646683/plugins.git
              ref: 13df320b6112a3a4abfbec47bba53b2f95402637
              path: packages/in_app_purchase
```
balance_page.dart:
``` dart
  @override
  void initState() {
    InappPurchaseService.getInstance().initListener(context);
    super.initState();
    /// 步骤1
    ResponseResult<List<AppleProduct>> response = await OrderService.getInstance().getAppleProduct();
    if (response.isSuccess) {
        /// 步骤2
        _initStoreInfo(response.data);
    }
  }

  @override
  void dispose() {
    InappPurchaseService.getInstance().removeListener();
    super.dispose();
  }
  
  _initStoreInfo(List<AppleProduct> appProductList) async {
    productDetailList = await InappPurchaseService.getInstance().initStoreInfo(context, appProductList);
  }
  
  build() {
      ...
      SMClickButton(
            /// 步骤3
            onTap: () => InappPurchaseService.getInstance().toCharge(productDetailList, selectIndex, context),
            child: Container(
              width: _Style.btnContainerW,
              height: _Style.bottomContainer,
              color: SMColors.btnColorfe373c,
              alignment: Alignment.center,
              child: Text('确认充值', style: SMTxtStyle.colorfffdp16,),
            ),
          ),
  }


```

inapp_purchase_service.dart：

```dart

initListener(BuildContext context) {
    final Stream purchaseUpdates = InAppPurchaseConnection.instance.purchaseUpdatedStream;
    _subscription = purchaseUpdates.listen((purchases) {
      _listenToPurchaseUpdated(context, purchases);
    }, onDone: () => _subscription.cancel(), onError: (error) => LogUtil.i(InappPurchaseService.sName, "error:" + error));

}
void _listenToPurchaseUpdated(BuildContext context, List<PurchaseDetails> purchaseDetailsList) {
    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {
      if (purchaseDetails.status == PurchaseStatus.pending) {
        LogUtil.i(InappPurchaseService.sName, 'PurchaseStatus.pending');
        LoadingUtil.show(context);
      } else {
        LoadingUtil.hide();

        if (purchaseDetails.status == PurchaseStatus.error) {
          ToastUtil.showRed("交易取消或失败");
        } else if (purchaseDetails.status == PurchaseStatus.purchased) {
          ToastUtil.showGreen("交易成功,正在校验");
          LogUtil.i(InappPurchaseService.sName, "purchaseDetails.purchaseID:" + purchaseDetails.purchaseID);
          LogUtil.i(InappPurchaseService.sName, "purchaseDetails.serverVerificationData:" + purchaseDetails.verificationData.serverVerificationData);
          /// 步骤4
          _verifyPurchase(purchaseDetails, needLoadingAndToast: true, context: context);
        }
      }
    });
}

// return bool needLock
Future<bool> _verifyPurchase(PurchaseDetails purchaseDetails, {bool needLoadingAndToast = false, BuildContext context}) async {
    Map param = {
      "transactionId" : purchaseDetails.purchaseID,
      "receipt": purchaseDetails.verificationData.serverVerificationData,
    };
    if (needLoadingAndToast) LoadingUtil.show(context);
    ResponseResult<dynamic> response = await ZBDao.charge(param);
    if (needLoadingAndToast) LoadingUtil.hide();
    if (response.isSuccess) {
      if (response.data == true) {
        /// 步骤6
        await InAppPurchaseConnection.instance.completePurchase(purchaseDetails);
        if (needLoadingAndToast) ToastUtil.showGreen('充值成功');
        OrderService.getInstance().getBalance();
        return false;
      } else {
        if (needLoadingAndToast) ToastUtil.showRed('充值失败');
        return true;
      }
    } else {
      LogUtil.i(InappPurchaseService.sName, '处理失败');
      return true;
    }
}


Future<List<ProductDetails>> initStoreInfo(BuildContext context, List<AppleProduct> appleProductList) async {
    final bool isAvailable = await _connection.isAvailable();
    if (!isAvailable) {
      return null;
    }

    List<String> productIdList = [];

    for(AppleProduct appleProduct in appleProductList) {
      productIdList.add(appleProduct.productId);
    }

    LoadingUtil.show(context);
    ProductDetailsResponse productDetailResponse = await _connection.queryProductDetails(productIdList.toSet());
    LoadingUtil.hide();

    if (productDetailResponse.error != null) {
      return null;
    }

    if (productDetailResponse.productDetails.isEmpty) {
      return null;
    }

    return productDetailResponse.productDetails;
}


void toCharge(List<ProductDetails> productDetailList, int selectIndex, BuildContext context) async {
    if (productDetailList == null) {
      ToastUtil.showRed("productDetailList为空");
      return;
    }
    LoadingUtil.show(context);
    /// a异常ii步骤
    bool needLock = await checkUndealPurshase();
    LoadingUtil.hide();
    if (needLock) {
      ToastUtil.showRed("有订单未处理");
      return;
    }

    final PurchaseParam purchaseParam = PurchaseParam(productDetails:productDetailList[selectIndex]);

    InAppPurchaseConnection.instance.buyConsumable(purchaseParam: purchaseParam);
}

// return bool needLock
Future<bool> checkUndealPurshase() async {
    /// a.异常i步骤，这里在进入app后，用户获取登录状态后调用
    LogUtil.i(InappPurchaseService.sName, '获取未处理list');
    try {
      List<PurchaseDetails> purchaseDetailsList = await _connection.getUndealPurchases();
      if (purchaseDetailsList.isEmpty) return false;
      LogUtil.i(InappPurchaseService.sName, '处理数组最后一个');
      PurchaseDetails purchaseDetails = purchaseDetailsList[purchaseDetailsList.length - 1];
      return _verifyPurchase(purchaseDetails);
    } catch(e) {
      ToastUtil.showRed('同步苹果支付信息失败');
      return true;
    }
}

```

---
完结，撒花🎉
