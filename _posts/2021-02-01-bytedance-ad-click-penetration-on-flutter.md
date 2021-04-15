---
layout: post
title: Flutter原生广告优化 -- 点击穿透
subtitle: Flutter UiKitView 接入头条原生模版信息流广告，出现点击穿透问题，如何解决？
tags: [Flutter, 穿山甲, 头条, iOS信息流广告, 点击穿透]
mermaid: true
---

> 本文默认以iOS平台开发示例，以下不再赘述

![flutter+BUADSDK](/assets/img/flutter/flutter+BUADSDK.png)

## Flutter 如何接入头条原生广告

头条SDK提供的原生模版广告是基于iOS平台的`UIView`（对应Android平台`View`），Flutter 页面渲染通常是`Widget`树，因此如果要在 Flutter 中接入原生广告，就必须采用 Platform Views 的方式实现。（这里提供对于 Flutter 接入原生`View`不太熟悉的读者传送门：[Hosting native Android and iOS views](https://flutter.dev/docs/development/platform-integration/platform-views)）

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

* 提供原生广告`View`

  ```swift
  class BUSdkNativeAd: NSObject, BaseNativeEntity {
      func getView() -> UIView? {
        	//默认提供一个空的广告View容器，等待广告渲染完成后添加上来
          return container
      }
      
      func loadAd() {
        	//省略：初始化参数
        	//开始加载原生广告
          self.nativeExpressAdManager = BUNativeExpressAdManager.init(slot: buSlot, adSize: size)
          self.nativeExpressAdManager.delegate = self
          self.nativeExpressAdManager.loadAdData(withCount: 1)
      }
      
      func dispose() {
          UiUtils.removeAllView(container: container)
      }
  }
  
  extension BUSdkNativeAd : BUNativeExpressAdViewDelegate {
      public func nativeExpressAdSuccess(toLoad nativeExpressAd: BUNativeExpressAdManager, views: [BUNativeExpressAdView]) {
        	//渲染广告View
          if(views.count > 0){
              let view = views[0]
              view.rootViewController = DevicesUtils.getVC()
              view.render()
          }
      }
      
      public func nativeExpressAdViewRenderSuccess(_ nativeExpressAdView: BUNativeExpressAdView) {
        	//渲染成功，把广告View挂到container上
          UiUtils.removeAllView(container: container)
          container.addSubview(nativeExpressAdView)
      }
      
      //MARK: - 省略其他头条广告回调接口
    	...
  }
  ```

  

## 原生广告如何渲染

头条广告原生`View`以`PlatformView`方式嵌入到 Flutter `Widget`树中展示时，Flutter 是怎么展示的呢？我们通过查看 Flutter 源码和实际使用案例的原生`View`层级两方面来看看。

**Flutter 源码简析**

首先我们定位到 Flutter 源码关于`PlatformView`的实现类，看看dart层创建一个`UiKitView`时，Flutter 是如何创建对应的原生`View`的。

从`FlutterViewController`创建入口开始，我们可以跟踪到`FlutterPlatformViewsController`实例是`FlutterEngine`的成员，用于管理所有`PlatformView`的添加移除，位置大小，层级顺序。 dart层的一个`UiKitView`都会对应到 native 层的一个`PlatformView`，两者通过持有相同的 viewid 进行关联，每次创建一个新`PlatformView`时，viewid++。

```objc
NSObject<FlutterPlatformView>* embedded_view = [factory createWithFrame:CGRectZero
                                                           viewIdentifier:viewId
                                                                arguments:params];
UIView* platform_view = [embedded_view view];
// Set a unique view identifier, so the platform view can be identified in unit tests.
platform_view.accessibilityIdentifier = [NSString stringWithFormat:@"platform_view[%ld]", viewId];
views_[viewId] = fml::scoped_nsobject<NSObject<FlutterPlatformView>>([embedded_view retain]);

FlutterTouchInterceptingView* touch_interceptor = [[[FlutterTouchInterceptingView alloc]
                                                    initWithEmbeddedView:platform_view
                                                    platformViewsController:GetWeakPtr()
                                                    gestureRecognizersBlockingPolicy:gesture_recognizers_blocking_policies[viewType]]
                                                   autorelease];

touch_interceptors_[viewId] =
  fml::scoped_nsobject<FlutterTouchInterceptingView>([touch_interceptor retain]);

ChildClippingView* clipping_view =
  [[[ChildClippingView alloc] initWithFrame:CGRectZero] autorelease];
[clipping_view addSubview:touch_interceptor];
root_views_[viewId] = fml::scoped_nsobject<UIView>([clipping_view retain]);
```

通过源码我们得知，Flutter 创建 `PlatformView`时，同时会创建`FlutterTouchInterceptingView`和`ChildClippingView`，它们之间的层级关系是：

<div class="language-mermaid" align="center">
    graph LR       
    ChildClippingView-->|contains|FlutterTouchInterceptingView-->|contains|PlatformView
</div>

<div align="center">
  <label width="50%" textAlignment="left" style="color: #B3B3B3; font-size: 0.8rem">PlatformView层级关系</label>
</div>

<br/>`FlutterTouchInterceptingView`的作用是拦截或传递原生View的手势事件，引述官方文档的描述是：

> A UIView that is used as the parent for embedded UIViews.
>
> This view has 2 roles:
> 1. Delay or prevent touch events from arriving the embedded view.
> 2. Dispatching all events that are hittested to the embedded view to the FlutterView.

文字总是晦涩难懂，下面是`PlatformView`实际案例的截图，可以更清晰明了地看到View的层级关系

|                     iOS View Tree                     |                 实时渲染层级                 |
| :---------------------------------------------------: | :------------------------------------------: |
| ![](/assets/img/flutter/express-overlay-viewtree.png) | ![](/assets/img/flutter/express-overlay.png) |



## 什么是点击穿透

在了解了头条原生广告是如何以`PlatformView`形式展示之后，我们继续来看看，点击穿透是如何发生的。当`PlatformView`滑动到页面某个位置时，可能会出现Flutter 页面中的Widget和`PlatformView`重叠的情况。此时点击屏幕时用户期望的是由最上方展示的UI层来响应点击事件，然而在实际开发中，笔者发现头条广告所在的`PlatformView`被Widget遮挡的时候，用户点击Widget，广告的点击事件却被响应了。如图所示：

![](/assets/img/flutter/express-penetrate.gif)

上图的场景是，把头条广告滑动到搜索栏下方，然后点击搜索栏Widget，响应到搜索事件（可以看到搜索页面已经被调起），然而广告点击事件也响应了（跳转到广告指定的App Store链接）。

## 点击穿透时 Flutter 做了什么

不知道读者发现了没，如上动图展示的场景，跟普通`PlatformView`没有被遮挡的场景不太一样，那么`PlatformView`被遮挡时，Flutter 又是如何渲染的呢？

由上面的iOS View Tree图我们知道，当 Flutter Application 启动后，会展示一个 `FlutterViewController`，而`FlutterViewController`的 root view 是一个 `FlutterView`，所有的Widget Tree对应的layer都是渲染在`FlutterView`上的。

原生广告也是一个原生`UIView`，按照我们看到的View层级，理论上如果添加了原生广告，原生广告`UIView`和`FlutterView`重叠的部分Widget layer应该是不可见的，然而现在它们确实可见的，而且还可以响应触摸事件，是不是很神奇？

**FlutterOverlayView**

我们再来看看，原生广告被遮挡时，iOS View Tree层级是啥样的

|                 iOS View Tree                  |                  实时渲染层级                  |
| :--------------------------------------------: | :--------------------------------------------: |
| ![](/assets/img/flutter/express-overlay-1.png) | ![](/assets/img/flutter/express-overlay-2.png) |

从图中可以看到，原生广告和Widget重叠之后，`PlatformView`的正上方被覆盖了一层大小一样的`FlutterOverlayView`，那么可以猜测被`PlatformView`遮挡的那部分Wiget layer应该是被再次地绘制到了`FlutterOverlayView`上。一起来看看源码：

```c++
void FlutterPlatformViewsController::BringLayersIntoView(LayersMap layer_map) {
  FML_DCHECK(flutter_view_);
  UIView* flutter_view = flutter_view_.get();
  auto zIndex = 0;
  // Clear the `active_composition_order_`, which will be populated down below.
  active_composition_order_.clear();
  for (size_t i = 0; i < composition_order_.size(); i++) {
    int64_t platform_view_id = composition_order_[i];
    std::vector<std::shared_ptr<FlutterPlatformViewLayer>> layers = layer_map[platform_view_id];
    UIView* platform_view_root = root_views_[platform_view_id].get();

    if (platform_view_root.superview != flutter_view) {
      [flutter_view addSubview:platform_view_root];
    } else {
      platform_view_root.layer.zPosition = zIndex++;
    }
    for (const std::shared_ptr<FlutterPlatformViewLayer>& layer : layers) {
      if ([layer->overlay_view_wrapper superview] != flutter_view) {
        [flutter_view addSubview:layer->overlay_view_wrapper];
      } else {
        layer->overlay_view_wrapper.get().layer.zPosition = zIndex++;
      }
    }
    active_composition_order_.push_back(platform_view_id);
  }
}
```



## 为什么会点击穿透

## 如何解决点击穿透

## 抽象通用解决方案



## 参考链接

[Github 相关issue](https://github.com/gstory0404/flutter_unionad/issues/4)

[Flutter PlatformView 渲染层级介绍](http://www.luoyibu.cn/posts/63773/)

[手把手教你定位FlutterPlatformView內存洩漏](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/759897/)