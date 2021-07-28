---
layout: post
title: iOS In-App Purchase -- 实践
subtitle: iOS In-App Purchase coding
feature-img: "assets/img/iOS/iap_crop.png"
tags: [iOS, IAP, Swift]
author: Jackin
---

> 提及iOS IAP，各位开发者可能心里都有一万只草泥马，实在太多坑。
>
> *本系列文章实践环境: **iOS 13 / Swift 5 / Xcode 11.3.1***

<p align="center">
  <img src="/assets/img/iOS/iap_framework.png" />
</p>

> The StoreKit framework connects to the App Store on your app’s behalf to prompt for and securely process payments. The framework then notifies your app, which delivers the purchased products. To validate purchases, you can verify receipts on your server with the App Store or on the device. For auto-renewable subscriptions, the App Store can also notify your server of key subscription events.

从代码的角度出发，当我们讨论IAP时，其实我们在讨论的是（Apple提供的）可与App Store安全链接的内购API——StoreKit。下面介绍几个接入IAP时必不可少的StoreKit API。


## [SKProduct](https://developer.apple.com/documentation/storekit/skproduct)
`SKProduct`对象是由`SKProductsResponse`对象返回数据的一部分，用来提供了在iTunesConnect中注册过的商品信息。
- `localizedDescription` 本地化描述
- `localizedTitle` 商品名称
- `price` 本地化价格
- `priceLocale` 用来格式化价格的本地化环境
- `productIdentifier` AppStore里的商品ID

## [SKProductsRequest](https://developer.apple.com/documentation/storekit/skproductsrequest)
`SKProductsRequest`对象用于从AppStore检索商品的本地化信息列表，程序可以用该请求获得商品的价格及其他本地化信息。

- **使用方法：** 使用商品ID(product identifier string)初始化一个`SKProductsRequest`对象，添加代理，然后调用对象的`start`方法，当请求返回数据，代理会通过代理方法获取`SKProductsResponse`对象的数据。
- **注意:** 要保持对请求对象的强引用，否则在请求完成之前对象就可能被释放。

## [SKPaymentTransaction](https://developer.apple.com/documentation/storekit/skpaymenttransaction)
`SKPaymentTransaction`类定义的是在`PaymentQueue`中驻留的交易对象，当一个付费事务(payment)被加入到队列(payment queue)中时，就会创建一个`SKPaymentTransaction`的实例。当AppStore处理完付费流程后，会将交易(transaction)传递到客户端。完成付费流程的交易对象会提供收据和标识，您可以在软件中永久保存已经完成的付费记录。
- `error` 描述执行过程中发生的错误信息
- `payment` 交易对应的付费事务对象
- `transactionState` 交易的当前状态
- `transactionIdentifier` 一个成功付费交易对应交易的唯一标识符
- `transactionDate` 付费事务(payment)添加到AppStore付费队列(payment queue)的时间

## [SKPayment](https://developer.apple.com/documentation/storekit/skpayment)
`SKPayment`类定义了在APP中为额外功能提供付费服务时向苹果AppStore请求的过程。一个payment封装了一个用于标识用户付费的商品和product的数量。
- `productIdentifier` 用于在APP内付费的唯一标识符，在iTunes Connect注册商品的时候写入的
- `quantity` 用户想要购买的数量
- `requestData` 保留以后使用的属性
- `applicationUsername` 一个用户帐户系统的不透明标识
- `simulatesAskToBuyInSandbox` 标识是否为沙箱测试的布尔值

## [SKPaymentQueue](https://developer.apple.com/documentation/storekit/skpaymentqueue)
`SKPaymentQueue`类代表了AppStore处理付费事务(payment)的队列。首先将至少一个观察者对象附加到队列中，然后添加一个用户想要付费的payment对象，每次添加payment对象，队列都会创建一个交易对象(transaction)去负责处理付费事务的流程和排序，付款完成后，队列会更新事务对象，然后调用预先设置的观察者对象更新事务的方法。观察者在该方法中处理事务，然后会被在队列中删除。

- 获取当前用户设置，是否允许付费
```swift
open class func canMakePayments() -> Bool
```

- 获取支付队列数据
```swift
open class func default() -> Self
```

- 添加和删除观察者
```swift
open func add(_ observer: SKPaymentTransactionObserver)
open func remove(_ observer: SKPaymentTransactionObserver)
```
- 处理交易数据
```swift
//获取队列中所有交易数据对象数组
open var transactions: [SKPaymentTransaction] { get }
//添加付费事务对象
open func add(_ payment: SKPayment)
//结束交易
//Remove a finished (i.e. failed or completed) transaction from the queue.  Attempting to finish a purchasing transaction will throw an exception.
open func finishTransaction(_ transaction: SKPaymentTransaction)
```

- 恢复购买
```swift
open func restoreCompletedTransactions()
```
***
下面我们开始进入IAP流程的开发

***

> **Tip**
>
Consider creating your observer as a shared instance of the class for global reference in any other class. A shared instance also guarantees the lifetime of the object, ensuring callbacks via the SKPaymentTransactionObserver protocol are always handled by the same instance.

IAP模块涉及全局付款交易、交易回调观察、交易数据缓存管理等，因此建议将此模块独立成一个单实例，与StoreKit API通信的部分写成一个辅助类。因此我们会有`PayApi`全局单例和`IAPHandler`辅助类。

## 设置付款队列的 Transaction Observer
在App中处理内购交易，必须创建一个观察者并将其添加到付款队列中。观察者对象响应新交易并将待处理交易队列与App Store同步，并且支付队列提示用户授权支付。App应在应用启动时添加交易观察者，以确保应用将尽快收到付款队列通知。
```swift
class IAPHandler: NSObject {
    ...
    override init() {
        super.init()
        // Attach an observer to the payment queue.
        SKPaymentQueue.default().add(self)
    }

    deinit {
        // Remove the observer.
        SKPaymentQueue.default().remove(self)
    }
    ...
}

//MARK:- Implement Payment Transaction Observer
//Observe transaction updates.
extension IAPHandler: SKPaymentTransactionObserver {
    func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
      //Handle transaction states here.
      ...
    }
}
```
在我们的单例`PayApi`中创建`IAPHandler`的实例，即可使Observer持续存在于整个App生命周期中。
```swift
class PayApi {
    private static let instance = {
        return CFPayApi()
    }()

    private init() {}

    class func shared() -> CFPayApi {
        return instance
    }

    ...

    fileprivate let iapHandler: IAPHandler = IAPHandler()

    ...
}
```
**在应用启动时初始化并添加交易数据观察者是非常重要的**，这样可以确保它在应用程序所有启动期间持续存在，接收所有付款队列通知并继续在应用程序外部可能处理的交易，例如：
- 推广应用内购买
- 后台订阅续订
- 购买中断
```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
      ...
      PayApi.shared()
      ...
    }
}
```

## 提供、完成和恢复应用内购买
### 展示可用本地化定价出售的产品
在展示可售产品之前，可以先判断当前用户是否被授权可以在该设备上进行交易
```swift
var isAuthorizedForPayments: Bool {
    return SKPaymentQueue.canMakePayments()
}
```
App确认授权后，就会向App Store发送产品请求，以从App Store获取本地化的产品信息。查询App Store可确保该App仅向用户显示可购买的产品。
```swift
fileprivate func fetchProducts(matchingIdentifiers identifiers: [String]) {
    // Create a set for the product identifiers.
    let productIdentifiers = Set(identifiers)

    // Initialize the product request with the above identifiers.
    productRequest = SKProductsRequest(productIdentifiers: productIdentifiers)
    productRequest.delegate = self

    // Send the request to the App Store.
    productRequest.start()
}
```
我们将在`IAPHandler`中实现产品请求回调代理协议，App Store使用`SKProductsResponse`对象响应产品请求。其产品属性包含有关可在App Store中实际购买的所有产品的信息。
```swift
extension IAPHandler: SKProductsRequestDelegate {
    func productsRequest (_ request:SKProductsRequest, didReceive response:SKProductsResponse) {
        if (response.products.count > 0) {
            response.products.forEach { product in
                LogUtil.d(tag: TAG, msg: "Valid product: \(product)")
                self.products[product.productIdentifier] = product
            }
        }
        if let complition = self.fetchProductComplition {
            complition(response.products)
        }
    }
}
```
### 发起产品购买
一旦我们通过product_id查询到合法的本地化可购买产品信息之后，即可通过`SKPaymentQueue`发起一笔购买交易。
```swift
func purchase(product: SKProduct, complition: @escaping ((IAPHandlerAlertType, SKProduct?, SKPaymentTransaction?)->Void)) -> Void {
    self.purchaseProductComplition = complition
    self.productToPurchase = product
    guard self.productToPurchase != nil else {
        complition(IAPHandlerAlertType.invalidProduct, nil, nil)
        return
    }

    if self.isAuthorizedForPayments {
        let payment = SKPayment(product: product)
        SKPaymentQueue.default().add(payment)
        LogUtil.d(tag: TAG, msg: "PRODUCT TO PURCHASE: \(product.productIdentifier)")
    } else {
        complition(IAPHandlerAlertType.disabled, nil, nil)
        LogUtil.w(tag: TAG, msg: "In App Purchase disabled by User")
    }
}
```

### 处理付款交易状态
当付款队列中有待处理的交易时，StoreKit会通过调用App的`paymentQueue（_：updatedTransactions :)`方法来通知应用的交易观察者。App应确保其观察者的`paymentQueue（_：updatedTransactions :)`可以随时响应任何一种状态。
```swift
func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
    for transaction: SKPaymentTransaction in transactions {
        switch transaction.transactionState {
        case .purchased:
            // The purchase was successful.
            completeTransaction(transaction)
        case .failed:
            // The transaction failed.
            failedTransaction(transaction)
        case .restored:
            // There're restored products.
            restoreTransaction(transaction)
        case .purchasing:
        case .deferred:
            // Do not block the UI. Allow the user to continue using the app.
        @unknown default:
        }
    }
}
```
当transaction failed时，检查其error属性以确定发生了什么。仅显示SKError#code与paymentCancelled不同的错误。
```swift
// Do not send any notifications when the user cancels the purchase.
if (transaction.error as? SKError)?.code != .paymentCancelled {
    DispatchQueue.main.async {
        // UI线程展示错误信息
        self.delegate?.storeObserverDidReceiveMessage(message)
    }
}
```

### 恢复已完成的购买
当用户购买非消耗品，自动更新订阅或非更新订阅时，他们希望它们可以在所有设备上无限期可用。App需提供一个UI交互，允许用户恢复其过去的购买记录。
```swift
func restorePurchase(_ callback: ((Bool, SKPaymentTransaction?) ->Void)?) -> Void {
    self.restorePayComplition = callback
    SKPaymentQueue.default().add(self)
    SKPaymentQueue.default().restoreCompletedTransactions()
}
```

### 提供内容并完成交易
&emsp;&emsp;在收到状态为`.purchased`或`.restored`的交易后，应用必须交付内容或解锁购买的功能。这些状态表明App Store已从用户处收到产品付款。

&emsp;&emsp;未完成的交易留在付款队列中。每当从后台启动或从后台恢复运行时，StoreKit都会调用该应用程序的永久观察者的`paymentQueue（_：updatedTransactions :)`，直到该应用程序完成这些交易为止。因此，App Store可能会反复提示用户对他们的购买进行身份验证或阻止他们从该应用程序再次购买产品。

&emsp;&emsp;对状态为`.failed`，`.purchased`或`.restored`的transaction调用`finishTransaction（_ :)`以将其从队列中删除。已完成的交易无法恢复，**因此应用程序必须提供购买的内容、下载产品的所有Apple托管内容或完成产品购买过程，然后才能调用`finishTransaction（_ :)`完成交易。**

```swift
// Finish the successful transaction.
SKPaymentQueue.default().finishTransaction(transaction)
```
