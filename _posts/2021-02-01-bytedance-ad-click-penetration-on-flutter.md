---
layout: post
title: Flutter原生广告优化 -- 点击穿透
subtitle: Flutter UiKitView 接入头条原生模版信息流广告，出现点击穿透问题，如何解决？
tags: [Flutter, 穿山甲, 头条, iOS信息流广告, 点击穿透]
---

> 本文默认以iOS平台开发示例，以下不再赘述

![flutter+BUADSDK](/assets/img/flutter/flutter+BUADSDK.png)

## Flutter 如何接入头条原生广告

头条SDK提供的原生模版广告是基于iOS平台的UIView（对应Android平台View），Flutter页面渲染通常是Widget树，因此如果要在Flutter中接入原生广告，就必须采用 Platform Views 的方式实现。（这里提供对于Flutter接入原生View不太熟悉的读者传送门：[Hosting native Android and iOS views](https://flutter.dev/docs/development/platform-integration/platform-views)）

~~~swift
<details>
  <summary>提供`FlutterPlatformView`接口实现类</summary>
	<p>
	```swift
public class FeedAdPlatformView: NSObject, FlutterPlatformView {
    
    private var feedAd: IExpressFeedAd?
    
    init(feedAd: IExpressFeedAd?) {
        self.feedAd = feedAd;
        super.init();
    }
    
    public func view() -> UIView {
        return feedAd?.getView() ?? UIView();
    }
}
	```
	</p>
</details>
~~~

* 提供`FlutterPlatformView`接口实现类

  ```swift
  public class FeedAdPlatformView: NSObject, FlutterPlatformView {
      
      private var feedAd: IExpressFeedAd?
      
      init(feedAd: IExpressFeedAd?) {
          self.feedAd = feedAd;
          super.init();
      }
      
      public func view() -> UIView {
          return feedAd?.getView() ?? UIView();
      }
  }
  ```

* 实现`FlutterPlatformViewFactory`的工厂接口实现类

  ```swift
  class FeedAdPlatformViewFactory: NSObject, FlutterPlatformViewFactory {
      
      private lazy var delegate: AdEntityDelegate? = nil;
          
      init(withDelegate delegate: AdEntityDelegate) {
          super.init()
          self.delegate = delegate;
      }
      
      required override init() {
          super.init()
      }
      
      func create(withFrame frame: CGRect, viewIdentifier viewId: Int64, arguments args: Any?) -> FlutterPlatformView {
          //MARK: - 初始化各种参数
        	let express = delegate?.fetchExpressFeedAd(withPosId: posId, withViewId: viewId)
        	return FeedAdPlatformView(nativeEntity: express)
      }
      
      public func createArgsCodec() -> FlutterMessageCodec & NSObjectProtocol {
        	//必须实现这个接口，并且提供与Flutter层相对应的实现算法类，否则通信参数无法传输
          return FlutterStandardMessageCodec.sharedInstance()
      }
  }
  ```

  

## 什么是点击穿透

## 点击穿透时 Flutter 做了什么

## 为什么会点击穿透

## 如何解决点击穿透

## 抽象通用解决方案

[Github 相关issue](https://github.com/gstory0404/flutter_unionad/issues/4)