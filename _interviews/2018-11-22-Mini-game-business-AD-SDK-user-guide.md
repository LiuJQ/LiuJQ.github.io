---
layout: post
title: MGB SDK 参考文档
author: Jackin
tags: [MGB-SDK]
---

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

	- [修订记录](#修订记录)
	- [适用说明](#适用说明)
	- [概述](#概述)
		- [MGB SDK简介](#mgb-sdk简介)
		- [MGB SDK能力](#mgb-sdk能力)
	- [接入指南](#接入指南)
		- [1. 导入MGB SDK](#1-导入mgb-sdk)
		- [2. PosId 和 NativePosId](#2-posid-和-nativeposid)
		- [3. MGB SDK初始化](#3-mgb-sdk初始化)
	- [使用指南](#使用指南)
		- [1. 拉取广告](#1-拉取广告)
		- [2. 获取广告](#2-获取广告)
		- [3. 展示广告](#3-展示广告)
	- [API参考](#api参考)
	- [Q&A](#qa)

<!-- /TOC -->

## 修订记录

Date | Author | Version | Remark
:--: | :--: | :--: | :--:
2018/11/22 | Jackin | 1.0.0 | 初始版本

## 适用说明
&emsp;&emsp;本文档由猎豹移动（珠海）商业产品部开发组维护，主要提供轻游戏商业广告SDK（以下简称MGB SDK）用户指引，适用于从事小程序、小游戏或其他前端开发人群。

## 概述
### MGB SDK简介
&emsp;&emsp;MGB SDK包含两个文件，`mgb_sdk.d.ts` 和 `mgb_sdk.js`。前者是SDK接口声明文件，后者是SDK实现代码文件。
### MGB SDK能力
&emsp;&emsp;MGB SDK提供轻游戏商业场景广告获取和展示接口，开发者接入后即可轻松获取到相关游戏平台提供的多种形式商业广告。
* **多开发引擎**

&emsp;&emsp;现阶段，MGB SDK内部提供不同开发引擎的API适配，已经支持Cocos Creator和Laya两个开发引擎，开发者不需要做相关开发引擎的判断，专注于商业场景广告的获取、展示、销毁逻辑即可。

* **多游戏平台**

&emsp;&emsp;现阶段，MGB SDK已提供包括微信小游戏、QQ玩一玩、Facebook Instant Game在内的多个轻游戏平台广告获取和展示能力。后续根据业务的发展，会陆续拓展其他轻游戏平台广告能力。
* **多样式广告**

&emsp;&emsp;现阶段，MGB SDK已提供包括Banner、激励视频、插屏广告在内的多种形式广告能力。后续根据业务的发展，会陆续拓展其他形式广告能力。

## 接入指南
### 1. 导入MGB SDK
* Cocos Creator

&emsp;&emsp;将MGB SDK接口声明文件和代码实现文件一起导入到源码工程`assets/libs/`目录，可能需要进一步设置，在属性检查器里面选中`mgb_sdk.js`文件，勾选‘导入为插件’，然后就可以在工程中引用MGB SDK的接口了。

* LayaAir

&emsp;&emsp;Laya工程导入的方式稍有差异，MGB SDK接口声明文件导入到`/libs/`目录，代码实现文件则需要导入`/bin/`目录下指定文件夹，比如`/bin/mgb/`，然后须在`/bin/index.html`文件中以JavaScript脚本形式引入，方可在源码工程中引用MGB SDK接口。

### 2. PosId 和 NativePosId

&emsp;&emsp;MGB SDK内部为每个广告位设计了一个广告位ID（定义为**PosId**）的概念，用于广告场景标记和Infoc埋点数据上报，PosId由产品负责人向猎豹BI数据平台申请下发；各游戏平台的广告获取必须要提供相应游戏平台分配的广告位ID（定义为**NativePosId**），每个游戏平台又有差别，下面列表力图向开发者清晰地展示各游戏平台广告位的差别：

Platform | AD Type | PosId(Infoc埋点) | Native PosId | Extra
:--: | :--: | :--: | :--:| :--:
微信 | Banner | BI平台分配 | 微信后台申请，adunit-xxx | 每个广告位分配一个adunit
微信 | 激励视频 | BI平台分配 | 微信后台申请，adunit-xxx | 所有广告位共用一个adunit
QQ | Banner | BI平台分配 | QQ玩一玩不需要 | /
QQ | 激励视频 | BI平台分配 | QQ玩一玩不需要 | /
Facebook | 激励视频 | BI平台分配 | Facebook Audience Network申请 | 所有广告位共用一个placement id
Facebook | 插屏广告 | BI平台分配 | Facebook Audience Network申请 | 所有广告位共用一个placement id

### 3. MGB SDK初始化
* 广告位配置

&emsp;&emsp;广告位的必要元素包括PosId、NativePosId和广告类型，具体的配置实现参考如下：
```js
/**
 * 配置banner广告
 * @param posId BI平台分配的广告位ID，用于埋点统计
 * @param adunitId 游戏平台分配的广告位ID，NativePosId
 * @param listPosIdCfg 广告位配置列表
 */
private static prepareBanner(posId, adunitId, listPosIdCfg) {
    let posIdCfg4ReviveScene = new MGB.MGBAdPosIdCfg(posId);
    // 设置广告类型为Banner
    posIdCfg4ReviveScene.setNum(MGB.MGBAdPosIdCfg.KEY_ADTYPE, MGB.MGBAdSelector4Auto.ADTYPE_BANNER);
    posIdCfg4ReviveScene.setStr(MGB.MGBAdPosIdCfg.KEY_ADPROVIDER_NATIVE_POSID, adunitId);
    // 设置默认的banner广告位置调整适配器，可选
    posIdCfg4ReviveScene.setObject(MGB.MGBAdPosIdCfg.KEY_AP4WXBANNER_POS_ADAPTER, new MGB.DefaultWXBAdPosAdapter());
    listPosIdCfg.push(posIdCfg4ReviveScene);
}

/**
 * 配置视频广告
 * @param posId BI平台分配的广告位ID，用于埋点统计
 * @param adunitId 游戏平台分配的广告位ID，NativePosId
 * @param listPosIdCfg 广告位配置列表
 */
private static prepareVideo(posId, adunitId, listPosIdCfg) {
    let posIdCfg4ReviveScene = new MGB.MGBAdPosIdCfg(posId);
    // 设置广告类型为Video
    posIdCfg4ReviveScene.setNum(MGB.MGBAdPosIdCfg.KEY_ADTYPE, MGB.MGBAdSelector4Auto.ADTYPE_REWARD_VIDEO);
    posIdCfg4ReviveScene.setStr(MGB.MGBAdPosIdCfg.KEY_ADPROVIDER_NATIVE_POSID, adunitId);
    // QQ视频广告要求设置视频类型，可选值包括0-游戏页面挂件广告|1-游戏结算页广告|2-游戏任务广告|3-游戏复活广告
    posIdCfg4ReviveScene.setNum(MGB.MGBAdPosIdCfg.KEY_AP4QQVIDEO_TYPE, 1);
    listPosIdCfg.push(posIdCfg4ReviveScene);
}
```

* 应用信息配置

&emsp;&emsp;MGB SDK需要简单的应用信息，包括应用唯一ID、应用名称、应用版本等，用于Infoc埋点统计上报。具体的配置实现参考如下：
```js
var infocEnvCfg = {
    game_id: 1000,
    game_name: "砖块消消消",
    gameVer: "1.0.0.0",
    channelKey: 1,  //可选，游戏平台渠道
    sub_channel: "" //可选，游戏用户来源渠道
}
var adEnvCfg = new MGB.MGBAdEnvCfg(infocEnvCfg);
```

* MGB SDK初始化

&emsp;&emsp;上述广告位配置和应用信息配置完成后，需要把广告位配置列表和应用配置信息一起传入MGB SDK内部进行初始化操作，具体实现如下：
```js
var adEnvCfg = MGBAdModule4BNB.prepareAdEnvCfg(); //获取应用信息配置
var aryPosIdCfg = MGBAdModule4BNB.preparePosIdCfgList(); //获取广告位配置列表
MGB.MGBAdModule.getInstance().init(adEnvCfg, aryPosIdCfg);
```

## 使用指南
&emsp;&emsp;开发者发起一次完整的广告展示请求，须包含三个必要的步骤:
1. 调用MGB SDK `loadAd`接口从广告平台拉取广告；
2. 调用MGB SDK `fetchAd`接口从MGB SDK缓存获取广告；
3. 调用获取到的广告实例进行展示；

### 1. 拉取广告
&emsp;&emsp;在广告场景触发时，调用MGB SDK拉取广告接口：
```js
MGB.MGBAdModule.getInstance().loadAd(posId, new MGB.ParamAdLoad(), {
    onSuccess: () => {
        console.log('ad loaded, posId: ' + posId);
        // 执行获取广告操作
    },
    onFailed: () => {
        console.log('ad load failed, posId: ' + posId);
    }
});
```
&emsp;&emsp;为了提高用户操作体验，我们建议在广告场景触发前进行视频和插屏广告的预加载操作。
```js
/**
 * 预加载广告
 * @param posId 广告位ID
 */
public static preloadAd(posId) {
    let cacheSize = MGB.MGBAdModule.getInstance().getCacheSize(posId);
    if (cacheSize > 0) {
        // 避免频繁加载广告
        console.info("ignore preload request, cache size: " + cacheSize);
        return;
    }
    MGB.MGBAdModule.getInstance().loadAd(posId, new MGB.ParamAdLoad(), {
        onSuccess: () => { },
        onFailed: () => { }
    });
}
```

### 2. 获取广告
&emsp;&emsp;广告拉取成功后，MGB SDK会将其保存至内部内存缓存中，方便开发者可以快速获取到广告数据。
```js
let resFetch = MGB.MGBAdModule.getInstance().fetchAd(posId, new MGB.ParamAdFetch(), null);
if (resFetch.mAdErrCode != MGB.MGBAdErrCode.ADERR_SUCCESS) {
    console.error("res(" + resFetch.mAdErrCode + ") is not MGBAdErrCode.ADERR_SUCCESS");
    return;
}
let ad: MGB.IAd = resFetch.getAd();
// 执行展示广告动作
```

### 3. 展示广告
&emsp;&emsp;获取到广告数据实例后，就可以执行广告的展示动作了：
```js
let ad: MGB.IAd = resFetch.getAd();
if (ad == null) {
    console.error("ad == null");
    return;
}
ad.show({
    onShowSuccess: (ad, arg0, arg1) => {
        // 广告展示成功
    },
    onShowFail: (ad, arg0, arg1) => {
        // 广告展示失败
    },
    onShowFinish: (ad, arg0, arg1) => {
        // 视频、插屏广告完整播放
    },
    onShowUnFinish: (ad, arg0, arg1) => {
        // 视频广告未完整播放
    }
});
```

## API参考
**拉取广告**
```js
MGB.MGBAdModule.loadAd(posId: string, param: ParamAdLoad, callback: IAdLoadCallback): number;
```
参数说明

参数 | 类型 | 必填 | 说明
:-: | :-: | :-: | :-: |
posId | string | 是 | 广告位id
param | ParamAdLoad | 是 | 预留拉取参数
callback | IAdLoadCallback | 是 | 拉取结果回调

返回值(number) 拉取广告结果码

**获取广告**
```js
MGB.MGBAdModule.fetchAd(posId: string, param: ParamAdFetch, callback: IAdFetchCallback): ResultFetchAd;
```
参数说明

参数 | 类型 | 必填 | 说明
:-: | :-: | :-: | :-: |
posId | string | 是 | 广告位id
param | ParamAdFetch | 是 | 预留获取参数
callback | IAdFetchCallback | 是 | 获取结果回调，可传null

返回值(ResultFetchAd) 获取广告返回数据，包好结果码和广告实例（可能为空）

## Q&A
Q: 什么是PosId？从哪里获取到？<br/>
A: PosId是MGB SDK定义的广告位ID概念；一般由产品负责人向猎豹BI平台申请。

Q: 为什么Facebook Instant Game接入MGB SDK后拉取广告总是返回'ad no fill' ？<br/>
A: 可以让产品负责人在Facebook广告系统后台查询一下该账户是否有开发者权限。
