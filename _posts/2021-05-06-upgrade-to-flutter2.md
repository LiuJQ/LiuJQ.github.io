---
layout: post
title: Flutter 2.0.6 升级踩坑记录
tags: [Flutter 2, 升级flutter2，踩坑]
---

> 本次升级环境为 flutter 1.22.x 升级到 flutter 2.0.6，不涉及 dart sdk 和空安全升级

### `intl` 依赖版本与 Flutter SDK 内部依赖冲突

  ```shell
  Because balalaper depends on flutter_localizations any from sdk which depends on intl 0.17.0, intl 0.17.0 is required.
  So, because balalaper depends on intl ^0.16.1, version solving failed.
  pub get failed (1; So, because balalaper depends on intl ^0.16.1, version solving failed.)
  ```

  **解决方法：**覆盖依赖版本

  ```yaml
  dependency_overrides:
    intl: 0.17.0
  ```

  新增上面的配置后，运行`flutter pub get`会出现警告信息，但不影响依赖分析和下载

  ```shell
  Warning: You are using these overridden dependencies:
  ! intl 0.17.0
  ```

### 第三方依赖库api变更编译失败

- `Scaffold` 参数废弃：`resizeToAvoidBottomPadding`
    <br/>
  **解决方法：** 使用新的参数替换 `resizeToAvoidBottomInset`

  

- `extended_nested_scroll_view` 编译失败

  ```shell
  The non-abstract class '_NestedScrollPosition' is missing implementation
  s for these members:
   - ScrollPosition.pointerScroll
  Try to either
   - provide an implementation,
   - inherit an implementation from a superclass or mixin,
   - mark the class as abstract, or
   - provide a 'noSuchMethod' implementation.
  ```

- `pull_to_refresh`编译失败，`BuildContext` API变更，以下几个方法已废弃

  ```dart
  ancestorWidgetOfExactType
  ancestorStateOfType
  inheritFromWidgetOfExactType
  ```

- `fluro-1.7.4`编译失败，`Navigator.of` API变更，去掉了`nullOk`参数

  ```shell
  Error: No named parameter with the name 'nullOk'.
              Navigator.of(context, rootNavigator: rootNavigator, nullOk: nullOk);
  ```

  

  **解决方法：**升级依赖版本

  |               升级前                |               升级后                |
  | :---------------------------------: | :---------------------------------: |
  | extended_nested_scroll_view: ^1.0.1 | extended_nested_scroll_view: ^3.0.0 |
  |       pull_to_refresh: ^1.6.3       |       pull_to_refresh: ^1.6.5       |
  |            fluro: ^1.7.4            |            fluro: ^2.0.3            |



### `CupertinoPageRoute` 构造方法编译失败，`opaque`由成员变量变成了`getter`属性，不能直接`assert`

  ```shell
  lib/components/scaffold/side_close_route.dart:55:16: Error: Getter not found: 'opaque'.
          assert(opaque),
  lib/components/scaffold/side_close_route.dart:85:16: Error: Getter not found: 'opaque'.
          assert(opaque),
  ```

  **解决方法：**修改构造方法，将`assert`放入构造方法体内部

  ```dart
  CupertinoPageRoute({
    required this.builder,
    this.title,
    RouteSettings? settings,
    this.maintainState = true,
    bool fullscreenDialog = false,
  }) : assert(builder != null),
       assert(maintainState != null),
       assert(fullscreenDialog != null),
       super(settings: settings, fullscreenDialog: fullscreenDialog) {
    assert(opaque);
  }
  ```



### iOS 升级 flutter 2 后编译失败，`Flutter/Flutter.h not found`
  **问题原因**：flutter 2 之前 iOS 依赖 flutter 的方式可能是直接符号链接的形式引用相关的framework，flutter 2 升级后不作任何修改的话，找不到对应的framework

  **解决方法：** 更新podfile，动态增加对flutter的依赖，在`post_install` section 下新增 `flutter_additional_ios_build_settings(target)`



### 弹窗蒙层消失
  **问题原因：** `baseDialog`定义时`barrierColor`没有赋默认值，flutter2之后`showDialog` api变更，导致蒙层失效
  
  ```dart
  ///flutter 1.x showDialog
  barrierDismissible: barrierDismissible,
  barrierLabel: MaterialLocalizations.of(context).modalBarrierDismissLabel,
  barrierColor: barrierColor ?? Colors.black54,
  transitionDuration: const Duration(milliseconds: 150),
  transitionBuilder: _buildMaterialDialogTransitions,
  useRootNavigator: useRootNavigator,
  routeSettings: routeSettings,
  
  ///flutter 2 showDialog
  Future<T?> showDialog<T>({
    required BuildContext context,
    required WidgetBuilder builder,
    bool barrierDismissible = true,
    Color? barrierColor = Colors.black54,
    String? barrierLabel,
    bool useSafeArea = true,
    bool useRootNavigator = true,
    RouteSettings? routeSettings,
  })
  ```
  
  **解决方法：** `baseDialog`定义时`barrierColor`默认赋值`Color barrierColor = const Color(0x80000000)`

