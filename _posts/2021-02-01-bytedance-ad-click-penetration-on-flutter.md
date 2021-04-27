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

**实际案例展示**

文字总是晦涩难懂，下面是`PlatformView`实际案例的截图，可以更清晰明了地看到`View`的层级关系

|                     iOS View Tree                     |                 实时渲染层级                 |
| :---------------------------------------------------: | :------------------------------------------: |
| ![](/assets/img/flutter/express-overlay-viewtree.png) | ![](/assets/img/flutter/express-overlay.png) |



## 什么是点击穿透

在了解了头条原生广告是如何以`PlatformView`形式展示之后，我们继续来看看，点击穿透是如何发生的。当`PlatformView`滑动到页面某个位置时，可能会出现Flutter 页面中的`Widget`和`PlatformView`重叠的情况。此时点击屏幕时用户期望的是由最上方展示的UI层来响应点击事件，然而在实际开发中，笔者发现头条广告所在的`PlatformView`被Widget遮挡的时候，用户点击`Widget`，广告的点击事件却被响应了。如图所示：

![](/assets/img/flutter/express-penetrate.gif)

上图的场景是，把头条广告滑动到搜索栏下方，然后点击搜索栏`Widget`，响应到搜索事件（可以看到搜索页面已经被调起），然而广告点击事件也响应了（跳转到广告指定的App Store链接）。

## 点击穿透时 Flutter 做了什么

不知道读者发现了没，如上动图展示的场景，跟普通`PlatformView`没有被遮挡的场景不太一样，那么`PlatformView`被遮挡时，Flutter 又是如何渲染的呢？

由上面的iOS View Tree图我们知道，当 Flutter Application 启动后，会展示一个 `FlutterViewController`，而`FlutterViewController`的 root view 是一个 `FlutterView`，所有的`Widget` Tree对应的layer都是渲染在`FlutterView`上的。

原生广告也是一个原生`UIView`，按照我们看到的View层级，理论上如果添加了原生广告，原生广告`UIView`和`FlutterView`重叠的部分`Widget` layer应该是不可见的，然而现在它们确实可见的，而且还可以响应触摸事件，是不是很神奇？

**FlutterOverlayView**

我们再来看看，原生广告被遮挡时，iOS View Tree层级是啥样的

|                 iOS View Tree                  |                  实时渲染层级                  |
| :--------------------------------------------: | :--------------------------------------------: |
| ![](/assets/img/flutter/express-overlay-1.png) | ![](/assets/img/flutter/express-overlay-2.png) |

从图中可以看到，原生广告和`Widget`重叠之后，`PlatformView`的正上方被覆盖了一层大小一样的`FlutterOverlayView`，那么可以猜测被`PlatformView`遮挡的那部分`Widget` layer应该是被再次地绘制到了`FlutterOverlayView`上。一起来看看源码：

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

要理解为什么会点击穿透，首先要理解iOS的事件传递和UI Responder Chain相关的知识。这里作一个简介：

### 传递链

- 事件传递的两个核心方法

  ```objc
  // recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system
  - (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;
  // default returns YES if point is in bounds
  - (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
  ```

  `hitTest`方法返回的是一个`UIView`，是用来寻找最终哪一个视图来响应这个事件。`pointInside`方法是用来判断某一个点击的位置是否在视图范围内，如果在就返回YES。

- `UIView`不响应事件的三种情况

  ```objc
  alpha < 0.01
  userInteractionEnabled = NO
  hidden ＝ YES
  ```

- 事件传递流程

  > 1. 我们点击屏幕产生触摸事件，系统将这个事件加入到一个由UIApplication管理的事件队列中，UIApplication会从消息队列里取事件分发下去，首先传给UIWindow
  > 2. 在UIWindow中就会调用hitTest:withEvent:方法去返回一个最终响应的视图
  > 3. 在hitTest:withEvent:方法中就会去调用pointInside: withEvent:去判断当前点击的point是否在UIWindow范围内，如果是的话，就会去遍历它的子视图来查找最终响应的子视图
  > 4. 遍历的方式是使用倒序的方式来遍历子视图，也就是说最后添加的子视图会最先遍历，在每一个视图中都会去调用它的hitTest:withEvent:方法，可以理解为是一个递归调用
  > 5. 最终会返回一个响应视图，如果返回视图有值，那么这个视图就作为最终响应视图，结束整个事件传递；如果没有值，那么就会将UIWindow作为响应者

### 响应链

- 响应链流程

  > 1. 如果view的控制器存在，就传递给控制器处理；如果控制器不存在，则传递给它的父视图
  > 2. 在视图层次结构的最顶层，如果也不能处理收到的事件，则将事件传递给UIWindow对象进行处理
  > 3. 如果UIWindow对象也不处理，则将事件传递给UIApplication对象
  > 4. 如果UIApplication也不能处理该事件，则将该事件丢弃

- 响应链图示

  ![](/assets/img/iOS/ui_responder_chain.png)

### 小结

理解了 iOS 事件传递链和响应链，再对照上面我们截图的点击穿透iOS view hierachy，不难发现，当我们点击覆盖在广告`View`上方的`Widget`时，最优先响应该事件的`View`是用来渲染被原生广告遮挡的`Widget` layer 的`FlutterOverlayView`，那么为什么广告响应了点击？笔者查看了`FlutterOverlayView`的实现源码，发现它在初始化时被禁用了用户交互响应：

```objc
- (instancetype)init {
  self = [super initWithFrame:CGRectZero];

  if (self) {
    self.layer.opaque = NO;
    self.userInteractionEnabled = NO;//笔者注：被禁用了用户交互，不能作为UI Responder Chain的其中一环了
    self.autoresizingMask = (UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight);
  }

  return self;
}
```

所以，点击事件就会在广告这个`View`上被响应！

## 如何解决点击穿透

问题找到了，导致问题出现的原因也清晰了，那么解决方案自然就呼之欲出了。

### 备选方案

上面分析了`FlutterOverlayView`被禁用了用户交互，不能作为事件响应链的其中一环，但它却又是最优先响应这个点击事件的`View`，因此最直接的方案就是修改`FlutterOverlayView`的源码，将它放开用户交互，事件就不会再穿透给原生广告`View`了。但这相当于是修改了 Flutter 引擎源码，不仅要重新编译引擎源码，而且会不会对 Flutter 的绘制或者事件响应造成影响，无法预料，该方案只能作为备选。

### 分析解决方案

那么，`FlutterOverlayView`我们无法修改，是不是可以在UI Responder Chain中插入一环，来拦截掉这种特殊情况下的点击事件传递呢？当然是可以的！上面我们介绍了iOS事件传递链的流程，仅需要给原生广告`View`添加一个父容器（其实上面的截图中已经出现过并用红色框聚焦了，`FLInterceptPenetrateView`），重写它的`hitTest`方法，通过调用`pointInside`方法来判断，当次点击事件是属于点击穿透的场景，则消费掉此次点击事件，不再传递即可。

根据我们对广告和`FlutterOverlayView`的重叠关系，当出现以下三种情况时，点击位置在`FlutterOverlayView`内部，则会出现点击穿透的问题

![](/assets/img/flutter/penetrate_cases.png)

<div align="center">
  <label width="50%" textAlignment="left" style="color: #B3B3B3; font-size: 0.8rem">FlutterOverlayView 与 原生广告View 重叠平面图</label>
</div>

<br/>上面三种case转换成算法也很简单

```swift
if feed!.frame.contains(view.frame) || view.frame.contains(feed!.frame) {
    // feed view contains overlay, or overlay contains feed view
    print("zzz \(containerType) contains overlay, or overlay contains \(containerType)")
    return true
}
if feed!.frame.intersects(view.frame) || view.frame.intersects(feed!.frame) {
    // feed view is intersected with overaly
    print("zzz \(containerType) is intersected with overaly")
    return true
}
```

### 点击穿透最终拦截方案

当我们利用上述算法判断出`FlutterOverlayView`出现在广告上方时，存在相交或者包含关系时，就可以认定如果点击位置在`FlutterOverlayView`内部，就会出现点击穿透，此时`FLIntercptPenetrateView`的`hitTest`方法，返回nil即可实现点击事件拦截，从而屏蔽调原生广告对本次点击事件的响应。

```swift
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    if !self.isUserInteractionEnabled || self.isHidden || self.alpha < 0.01 {
        // interaction disable
        // hidden
        // nearly invisble
        return nil
    }
    
    //找到点击落点所在的Overlay
    let targetOverlay = findTargetOverlayView(point)
    //判断Overlay符合与广告相交、重合，则返回nil实现事件拦截
    if let target = targetOverlay, isOverlayOnTopOfFeedView(container: self, overlay: target) {
      	return nil
    } else {
      	return super.hitTest(point, with: event)
    }
}
```



## 参考链接

[Github 相关issue](https://github.com/gstory0404/flutter_unionad/issues/4)

[Flutter PlatformView 渲染层级介绍](http://www.luoyibu.cn/posts/63773/)

[手把手教你定位FlutterPlatformView內存洩漏](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/759897/)

[Using Responders and the Responder Chain to Handle Events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events)