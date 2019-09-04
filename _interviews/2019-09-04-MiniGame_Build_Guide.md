---
layout: post
title: MiniGame Build 参考文档
author: Jackin
tags: [MiniGame-Build]
---

# 欢乐1010
## 发版代码
> Repository: http://gitlab.liebaopay.com:9090/small-game/game_1010.git  
Branch: origin/master

## 编译平台
> Engine:   Cocos Creator v1.9.3  
Render    Mode: Auto (WebGL First)  
Platform: Wechat Game  
AppId: wxdb85ab8cbba21ec0

## 资源 CDN
> server: ftp:tp:up.cdn.kisops.com  
account: piano-weixin-game.cmcm.com  
password: mqQ)j~EBtI+Vyq  
directory: happy1010

## 修改 game.json
> step 1: 修改 build_subcontext.py 脚本中的 buildSubContext() 方法，里面有最新的AppId列表  
step 2: 更新 version.json 版本号  
step 3: 启动 Cocos Creator 编译，等待构建完成  
step 4: 执行 build.py 脚本，选择 3（腾讯版本）  
step 5: 微信开发者工具打开`game_1010/build/wechatgame`目录，如运行无问题，直接上传新版本即可  
step 6: 记得提交本次修改，一共包括三个文件 `version.json` `build_subcontext.py` `game_info.json`


# 开心六角消除
## 注意事项
> **1、game-2048项目，Cocos Creator构建完成后不能直接上传，需要经过压缩混淆；**  
**2、需要混淆的原因，据说是该项目复用了大量原有游戏的代码框架，腾讯审核为侵权**  
**3、混淆平台对接人：官金檀**  
> 平台地址：http://129.204.186.15/loginr.php  
账号： gpduser 密码： gpduseradmindp


## 发版代码
> Repository: http://gitlab.liebaopay.com:9090/small-game/game-2048.git  
Branch: origin/final_1010

## 编译平台
> Engine:   Cocos Creator v1.9.3  
Render    Mode: Auto (WebGL First)  
Platform: Wechat Game  
AppId: wxdf0cbb9fbd12e76f

## 资源 CDN
> server: ftp:tp:up.cdn.kisops.com  
account: piano-weixin-game.cmcm.com  
password: mqQ)j~EBtI+Vyq  
directory: happy2048

## 修改 game.json
> step 1: 修改 build_subcontext.py 脚本中的 buildSubContext() 方法，里面有最新的AppId列表  
step 2: 更新 version.json 版本号  
step 3: 启动 Cocos Creator 编译，等待构建完成  
step 4: 执行 build.py 脚本，选择 3（腾讯版本）  
step 5: 将`game-2048/build/wechatgame`目录打包zip压缩，上传到混淆平台，混淆完成后下载压缩包，（先清空）覆盖到`game-2048/build/wechatgame`目录即可，任何压缩包不要保留在该目录  
step 6: 微信开发者工具打开`game-2048/build/wechatgame`目录，如运行无问题，直接上传新版本即可  
step 7: 记得提交本次修改，一共包括三个文件 `version.json` `build_subcontext.py` `game_info.json`

# 滚动的天空-微信
## 发版代码
> Repository: http://gitlab.liebaopay.com:9090/small-game/rollingsky.git  
Branch: origin/wx_1.1.11

## 编译平台
> Engine:   Laya Air v1.8.3  
Platform: Wechat Game  
AppId: wx60001e99e6f99e70  
发布平台：微信小游戏  
是否重新编译脚本：勾选  
是否压缩混淆JS： 勾选（如果是要调试断点不发外网，不勾选）

## 资源 CDN
> server: up.cdn.kisops.com  
account: minigame-rs.duba.com  
password: I+Vyq_3JSEBtyq8*H@MkS  
directory: rs2/wx/res20190902_01（最新资源版本）

## 构建流程
**详细的构建流程请参考：/rollingsky/readme.txt**
>1、进入到`/tools/wx/`目录，执行`preprocess.py`脚本，输出目录`/client/build_preprocess/macro_process`  
>2、LayaAir打开上述输出目录；  
>3、LayaAir切换到编辑模式，执行导出；  
>4、LayaAir切换到代码模式，点击发布，弹出构建弹窗：
```raw
发布平台：微信小游戏
源根目录：/client/build_preprocess/macro_process/bin
发布目录：/client/release/wxgame
“是否重新编译脚本”：勾选
“是否压缩混淆JS”：勾选
其他选项均不勾选
```
>5、点击发布，等待构建完成，如果报错，说明失败，要找原因；  
```raw
如提示ISuperMan.js recommender.js找不到，可以忽略不管
```
>6、进入到`/tools/wx/`目录，执行`buildPackage.bat`脚本，拷贝`code.js`文件  
>7、如有资源变动，需要新打资源包；进入到`/tools/wx/`目录，执行`buildCdnRes.bat`脚本，等待；

# 欢乐小农场
## 发版代码
> Repository: http://gitlab.liebaopay.com:9090/master-caffe/FarmVegeteal.git  
Branch: origin/farmvegeteal_1.0.0.25_rb

## 编译平台
> Engine:   Cocos Creator v2.0.5  
```raw
该版本引擎渲染有bug，需要修改引擎代码，请找陈春晓协助修改
```
Platform: Wechat Game  
AppId: wxd6f57d0f0707d17a

## 资源 CDN
> server: ftp:tp:up.cdn.kisops.com  
account: piano-weixin-game.cmcm.com  
password: mqQ)j~EBtI+Vyq  
directory: farmvegeteal/dev/1.0.1.0（最新资源版本）

## 构建流程
> step 1: 更新 version.json 版本号  
step 2: 启动 Cocos Creator 编译，等待构建完成  
step 3: 执行 build.py 脚本，选择 1（WX腾讯正式版本），如需打资源包 `python build.py upgrade:res`  
step 4: 微信开发者工具打开`FarmVegeteal/build/wechatgame`目录，如运行无问题，直接上传新版本即可  
```raw
欢乐小农场编译脚本有问题，checkout代码后首次编译出来运行会报找不到xxx.json文件错误，复制完整的文件名，添加到 build_config.json 文件的 packageZip/exceptFile 数组中，再次重新编译
```
step 5: 记得提交本次修改，一共包括两个个文件 `version.json` `configuration.js`
