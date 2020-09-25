---
layout: post
title: iOS In-App Purchase -- 探索
subtitle: iOS In-App Purchase explore
feature-img: "assets/img/iOS/iap_crop.png"
tags: [iOS, IAP, Swift]
---

> 提及iOS IAP，各位开发者可能心里都有一万只草泥马，实在太多坑。
>
> *本系列文章实践环境: **iOS 13 / Swift 5 / Xcode 11.3.1***

## 简介
IAP 全称：In-App Purchase，是指苹果 App Store 的应用内购买，是苹果为 App 内购买虚拟商品或服务提供的一套交易系统。


## 适用范围
在App内需要付费使用的虚拟商品/服务，比如游戏道具、电子书、音乐、视频、订阅会员、App的一些高级功能需要使用IAP，如果是用户在iOS应用内购买的是实体商品则是不适用于IAP的，比如用户在淘宝买的购买的鞋子等实体商品，又或者用户完成购买后不在App内使用的虚拟商品（如话费充值）或服务（滴滴打车、Uber、Lyft）则不适用于IAP。

如果需要在应用内界定并使用IAP的话，苹果会收取30%的交易额作为平台抽成，如果开发者或者公司又需要缴纳一定的税的话实际到手的的收入会少于70%，对于内容创作者或者是分销商等利润比较少的公司则是非常大的门槛和打击。

这30%的抽成就是大家常说的苹果税。因为苹果的内购在国内具体使用时会存在使用时比较慢，操作复杂等问题，所以在实际用苹果内购支付的话在支付转化率上，会产生一些比如转化率折损的情况。所以在实际体验时比如订餐时都会弹出选项让用户去选择微信或者支付宝。

## 类型
IAP提供的Product类型

类型 | 示例 | 描述
:-- | :--: | :--
**Consumable** / 消耗型 | 游戏币<br/>生命值<br/>经验值 | 这些可以购买多次，并且可以用完。这些非常适合额外的生活，游戏中的货币，临时强化等。
**Non-Consumable** / 非消耗型 | 升级到专业版本<br/>去除广告<br/>新角色<br/>新场景 | 购买过一次并且希望永久拥有的东西，例如额外的关卡和可解锁的内容。
**Non-Renewing Subscription** / 不可续订订阅 | VIP包月<br/>运动季票<br/>报纸订阅<br/>腾讯NBA单场球赛 | 固定时间段内可用的内容。
**Auto-Renewing Subscription** / 自动续订订阅 | VIP连续包月 | 重复订阅，例如每月自动付费订阅VIP

## 创建Product
IAP购买的产品需要在App Store Connect平台创建，需要提供以下几个必要信息：
- **Reference Name:** 在iTunes Connect中标识IAP Product的昵称。该名称不会出现在应用程序中的任何位置。
- **Product ID:** 这是标识IAP Product的唯一字符串。通常最好以`bundle id`作为前缀，然后为该可购买商品添加一个唯一的名称。例如 `cn.jackin.vip.monthly`
- **Cleared for Sale:** 启用或禁用IAP Product的销售。记得要启用它！
- **Price Tier** 该IAP Product的价格

## 创建 Sandbox 用户
在深入研究一些代码之前，还需要执行一步。在应用程序的开发版本中测试应用程序内购买时，Apple提供了一个测试环境，使您可以“购买”您的IAP产品，而无需进行财务交易。 这些特殊的测试购买只能通过App Store Connect中的特殊“Sandbox Tester”用户帐户进行。仅注册后的 Sandbox 测试账号可以参与IAP Product测试，并且支付时不会产生任何实际费用。

在App Store Connect中，单击窗口左上角的App Store Connect返回主菜单。选择`Users and Roles`，然后单击`Sandbox Testers`选项卡。单击`Tester`标题旁边的 + 号即可开始创建 Sandbox 测试账号。

![创建沙盒测试账号](/assets/img/iOS/iap_sandbox_user.png)

## 项目配置
- Provision 描述文件更新
App Store Connect 平台找到对应的Provision描述文件配置，勾选In App Purchase，然后重新下载新.mobileprovision文件，导入Xcode
- Xcode 工程配置
选中项目Target，切换到 `Signing & Capabilities` 选项卡，点击左上角 `+ Capability`，搜索 In App Purchase 并添加后即可

## 注意事项
接入IAP必须要保证开发者账号填写的税务和收款信息，即itunesconnect.apple.com里面的Agreements, Tax, and Banking里面的收款信息。否则编码过程中向App Store请求商品数据的时候将不会收到任何相关的Product信息。

![Apple账号税务信息](/assets/img/iOS/iap_tax_transfer.png)
