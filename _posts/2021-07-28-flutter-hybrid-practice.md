---
layout: post
title: Flutter混合开发实践之--元气壁纸iOS
tags: [Flutter, Hybrid, Flutter混合开发, 混合开发, FlutterBoost]
author: Jackin
---

## 背景

&emsp;&emsp;元气壁纸APP基于纯`Flutter`工程模式开发，在接入外部SDK进行商业化开发时，纯`Flutter`工程模式决定了接入第三方广告`View`时，仅能走`Flutter`提供的`PlatformView`方式。这条路我们一路走来跌跌撞撞，也算是付出了努力，然而结果却差强人意。

## Xcode工程准备

&emsp;&emsp;在做《Flutter混合开发模式探索》分享时我们曾提到，`Flutter`混合开发的工程模式是三端分离模式，`iOS`代码与`Flutter Module`代码是分离的，并且`Flutter Module`是可以单独编译导出`Android/iOS`双端依赖产物。
&emsp;&emsp;然而，元气壁纸工程已经是纯`Flutter`工程，如果进行`Module`化改造，代价非常大，并且还不好合理地进行模块化分离，这显然不是我们期望的。后来，我们创造性地就在当前已有的工程模式下进行混合开发模式改造。当然，这种方式能成功，也是得益于`Flutter`已有的构建体系，帮助我们省去很多中间过程。

## 混合架构

### `Flutter`模式

&emsp;&emsp;该模式下，App仅需要一个继承自`UIViewController`的`FlutterViewController`，不需要对`iOS`原生部分源代码作过多的修改，维持`Flutter`创建`Project`时的代码结构即可。`AppDelegate`继承自`FlutterAppDelegate`，与`Flutter`相关的所有逻辑都由父类实现了，包括`engine`初始化并启动、`FlutterViewController`的创建并绑定`engine`、`plugins`注册及初始化、App生命周期代理回调等。对源码感兴趣的同学可以去阅读，路径：`engine/shell/platform/darwin/ios/framework/Source/FlutterAppDelegate.mm`。

### `Hybrid`模式

&emsp;&emsp;该模式下，App的主页面框架必须是`native`实现的，在此基础上考虑将部分需要原生支持的功能修改为`native`实现，优先级较低或者不涉及混合的功能保持原有`Flutter`实现。结合元气壁纸的产品和商业化需求，主页面的混合结构为：

- 动态壁纸`tab`，壁纸列表需要插入原生信息流广告，`native`实现
- 静态壁纸`tab`，壁纸列表需要插入原生信息流广告，`native`实现
- 桌面美化`tab`，页面结构简单且功能单一，保持`Flutter`实现
- 人气作者`tab`，暂无商业化需求和其他混合需求，保持`Flutter`实现
- 个人中心`tab`，不适合插入商业化且无混合需求，保持`Flutter`实现

## 混合改造

&emsp;&emsp;经过前期调研，基于`Android`版混合改造经验，`iOS`混合改造仍有必要接入`FlutterBoost`来管理混合路由栈，`FlutterBoost`是基于全局单实例`Flutter engine`实现的，因此混合开发改造首先就需要对`AppDelegate`进行改造。

- `AppDelegate`

  不能再继承`FlutterAppDelegate`，伴随而来的是与`Flutter`相关的所有事情都要开发者自己处理。所幸`FlutterBoost`框架为我们处理了大部分工作，包括`Flutter engine`初始化并启动、`plugins`注册及初始化。但是App生命周期代理回调，就需要开发者自己处理了。

- 主页面结构

  基于元气壁纸设计，主页面采用`UITabBarController`来实现底部`tab`设计，然后为每个`tab`创建对应的`UIViewController`，`native`实现的模块继承`UIViewController`，`Flutter`实现的模块直接采用`FBFlutterViewContainer`（该类由`FlutterBoost`提供，继承自`FlutterViewController`，内部处理了`engine`的绑定/解绑、路由处理、`Flutter`生命周期感知等）

## 代码细节

&emsp;&emsp;`FlutterBoost` 3.0 以后进行了双端`api`一致性改造，这也是当时我们选择 3.0 版本的重要原因。

- `BoostDelegate` -- `FlutterBoost`初始化必要参数，提供路由栈`push`/`pop`的具体实现

  ```swift
  class BoostDelegate: NSObject, FlutterBoostDelegate {
    func pushNativeRoute(_ pageName: String!, arguments: [AnyHashable : Any]!) {
      //在这里提供原生页面跳转实现方案
      ...
    }
    func pushFlutterRoute(_ options: FlutterBoostRouteOptions!) {
      //在这里提供flutter路由push操作实现方案
      //options会提供携带的参数、路由name等
      ...
    }
    func popRoute(_ options: FlutterBoostRouteOptions!) {
      //在这里提供flutter路由pop操作实现方案
      //options会提供携带的参数、路由name等
      ...
    }
  }
  ```

  

- `BoostRouters` -- 原生调用`Flutter`功能的实现类，最终会调用到`BoostDelegate`中去

  ```swift
  class BoostRouters: NSObject {
    class func presentDialog(target: FDCDialogs) {
      //原生调用flutter，打开对应类型的flutter dialog
    }
    class func pushFlutterPage(pageName: String, opaque: Bool = true) {
      //原生调用flutter，打开pageName对应的flutter页面，opaque指定页面不透明模式
    }
    class func pushFlutterPage(pageName: String, opaque: Bool = true, args: [String: Any]?) {
      //原生调用flutter，打开pageName对应的flutter页面，opaque指定页面不透明模式
      //args，可用于传递到flutter层的参数
    }
    class func popFlutterPage(uniqueId: String) {
      //原生关闭一个路由栈内的页面，很少用到
      FlutterBoost.instance().close(uniqueId)
    }
  }
  ```

- `FlutterBoost`初始化

  ```swift
  let delegate = BoostDelegate()
  FlutterBoost.instance().setup(application, delegate: delegate) {[weak self] engine in
    
  }
  ```

