---
layout: post
title: iOS In-App Purchase -- 实践
subtitle: iOS In-App Purchase coding
feature-img: "assets/img/iOS/iap_crop.png"
tags: [iOS, IAP, Swift]
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
        SKPaymentQueue.default().add(self)
    }

    deinit {
        SKPaymentQueue.default().remove(self)
    }
    ...
}

//MARK:- Implement Payment Transaction Observer
extension IAPHandler: SKPaymentTransactionObserver {
    func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
      ...
    }
}
```
在我们的单例`PayApi`中创建`IAPHandler`的实例，即可使Observer保持存在于整个App生命周期中。
