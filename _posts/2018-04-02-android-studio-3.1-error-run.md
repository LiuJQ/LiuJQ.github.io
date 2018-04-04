---
layout: post
title: Android Studio -- The APK file does not exist on disk
tags: [Android Studio, Gradle]
---

### 问题探究
&emsp;&emsp;最近Android Studio推出了最新稳定版本3.1，收到升级提示后迫不及待更新了。然而，之前好好的项目运行环境，升级了3.1之后居然Run不起来了，感觉有坑。Gradle Sync是成功的，就是每次点Run就立即提示错误：
> Session ‘app’: Error Installing APK

&emsp;&emsp;底部Run面板提示如下：
```gradle
The APK file /xxx/xxx/app/build/outputs/apk/app-debug-unsign.apk does not exist on disk.
```
&emsp;&emsp;根据提示，那么Terminal手动执行一下命令来编译apk，再点击Run会是什么结果？
```gradle
./gradlew assembleDebug
```
&emsp;&emsp;答案是不报错apk也可以跑到机器上了。很明显，Gradle并没有执行代码编译操作就提示错误了。

### 解决方法
&emsp;&emsp;原因很明确，解决方法就不难了。
* 选择菜单Run
* 选择选项Edit Configurations
  * 展开左边Android App下拉列表
  * 选择app module
  * 看右下方的配置Before launch
  * 点击+，新增task，选择Gradle-aware Make，不填参数，直接点OK
  * 调整顺序，Gradle-aware Make上移到顶部
  * 点击Apply，应用配置变更

![Android Studio 3.1 Error Installing APK](/assets/img/screenshots/20180402.png)
