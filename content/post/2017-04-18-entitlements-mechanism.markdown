---
layout:     post
title:      "entitlements授权机制"
subtitle:   "解析 iOS entitlements作用"
date:       2017-04-18 00:00:00
author:     "sumbrilliance"
thumbnailImagePosition: left
thumbnailImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/about-bg.jpg"
coverImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/about-bg.jpg"
header-mask: 0.3
categories:
    - iOS
tags:
    - iOS
    - 签名
    - entitlements
---
在对ipa重签名的时候，你可能会疑惑，为什么可以用别的证书重签之后，ipa就可以安装，又为什么在重签的时候需要加入 embedded.mobileprovision 文件，重签失败出现的闪退又是为什么。那么也许你需要深入了解 entitlements 的验证机制。

<!--more-->

原来的素材图放在七牛云可能因为很久没有登上去处理上面的图片，无法查看无法下载，也是坑。中间又经历过重装系统，原图也丢失了，有兴趣的朋友可以通过下文流程图出处找

## 一、entitlements 验证流程

### 1.1 entitlements流程图

![流程图](http://7xqjl4.com1.z0.glb.clouddn.com/Entitlements-0.png)
如图所示，entitlements.plist和pp文件都是entitlements这一授权机制的体现，两者彼此分工又互相配合。前者主管app在沙盒中的权限，而pp文件既管理沙盒权限，也管理允许运行的设备号，验证签名的证书是否有效等等。
![启动验证流程](http://7xqjl4.com1.z0.glb.clouddn.com/Entitlements-1.png)

### 1.2 app的entitlements会在程序安装和启动时进行验证
分为两步：

**1.2.1 There are no unexpected or misconfigured entitlements**
检查格式对不对，以及有没有正确配置。

**1.2.2 That entitlements match across the app’s signature and the app’s embedded provisioning profile.**
拿entitlements去检查app内部的signature文件和pp文件中的权限声明是否匹配。
如果验证不通过，**无法安装/启动**。

## 二、查看entitlements的方法

### 2.1 直接查看
![常看entitlements文件](http://7xqjl4.com1.z0.glb.clouddn.com/Entitlements-2.png)
但是这个entitlements文件所列的权限不全，有些不会列出来。比如application-identifier,keychain-access-groups这些。

### 2.2 打包时查看。
![Archive时查看entitlements](http://7xqjl4.com1.z0.glb.clouddn.com/Entitlements-4.png)
在这里可以看到我们将要打包的app的所有权限，我们可以检查是否有遗漏，比如说推送的权限，后台播放的权限。

### 2.3 codesign 命令行工具查看
> codesign -d --entitlements :- YourApp.app

也可简写为

> codesign -d --ent :- /path/to/the.app

### 2.4 ldid 命令行工具查看
> ldid -e YourApp.app/YourApp

### 2.5 surcurity 命令行工具查看
>security cms -D -i /path/XXXXX.mobileprovision

### 2.6 进到xxx.app里面，用vim打开里面的embedded.mobileprovision，文件，往下翻到entitlements目录
![mobileprovision中的entitlments示意](http://7xqjl4.com1.z0.glb.clouddn.com/Entitlements-3.png)

## 三、生成entitlements.plist文件
这里介绍一个技巧快速生成entitlements.plist文件，使用codesign工具：
> codesign -d --entitlements :entitlements.plist /path/from/XXX.app

会在当前文件夹生成和XXX.app权限一样的entitlements.plist文件

## 四、一些entitlements说明

- beta-reports-active
Beta reports active is a key entitlement with no value. It exists on all App Store distribution provisioning profiles and determines that the app can be distributed through TestFlight.

- team-identifier
Your Team ID, which is your team's unique 10-digit alpha/numeric value. This value is often used as the default App ID prefix. Certain features are only allowed across apps whose team-identifier value match, for example, Handoff, keychain and UIPasteboard sharing.

- get-task-allow
The boolean value of get-task-allow determines whether Xcode's debugger can attach to the app.

- application-identifier
The string value of application-identifier is of the format: <prefix>.<bundle_id> and it corresponds to your app's App ID. Often times the prefix is equal to the Team ID though it isn't always the case. In the provisioning profile, this value includes an asterisk if it is associated to a wildcard App ID. In either case, the application-identifier on an app's signature is always fully qualified to include the app's full bundle ID.

- keychain-access-groups
This entitlement defines an array of strings corresponding to keychains you intend the app to have access to. Each string value in the array are of the format: <prefix>.<bundle_id>. By default, Xcode creates this entitlement for you a value equal to the application-identifier entitlement. All prefixes in this array of strings must match. For more information, see the SecItemAdd section of the Keychain Services Reference.

- push-notification
The presence of this entitlement indicates that the app is enabled to receive push notifications. The value of this entitlement is either "production" or "development", which refer to the live and sandbox push notification servers, respectively. For more information on troubleshooting push notifications.

## 一些注意的点

**1.app上架会经过App Store的再次签名，所以embedded.mobileprovision文件只存在于你开发的时候那个.app中，从App Store下载下来的ipa里面的XXX.app中是没有这个文件的。**

**2.越狱开发的时候，使用一些私有api，有些api是需要特殊的entitlements属性的，必须设置才能生效。** 比如

```
<key>com.apple.locationd.authorizeapplications</key>
<true/>

```
是用来快速开关定位所依赖的entitlements，如果没有用此属性签名，定位开关的私有api将不会起效。
而通常这些特殊entitlements是不公开的，所以用这些entitlements签名的app，将安装不上非越狱机(经本人测试确实不行，如果有其他黑魔法，可以让非越狱机使用这些依赖特殊entitlements的私有api，请告知笔者，谢谢！)

[流程图出处](https://developer.apple.com/library/ios/technotes/tn2415/_index.html#//apple_ref/doc/uid/DTS40016427-CH1-OSVALIDATION)
[pp 验证流程图出处](https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/AppDistributionGuide/TestingYouriOSApp/TestingYouriOSApp.html#//apple_ref/doc/uid/TP40012582-CH8-SW1)
[代码签名探析](http://objccn.io/issue-17-2/)
