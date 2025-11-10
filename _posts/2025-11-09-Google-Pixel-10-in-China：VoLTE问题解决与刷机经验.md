---
layout: post
toc: true
categories: tech life
title: "Google Pixel 10 in China：VoLTE问题解决与刷机经验"
---
<!-- ## Google Pixel 10 in China: VoLTE问题解决和刷机经验 -->

昨天从上海开会回来时把伞忘在酒店了，前台找到室友说我的电话打不通。 我怀疑可能和Pixel 10在中国大陆的某些限制有关，求助ChatGPT解决这个问题。

### 检查VoLTE是否开通

拨号输入 `*#*#4636#*#*` → **Phone information**，发现未开通：

```
IMS registration status: Unegistered
Voice over LTE: Unavailable
```

后来用`adb shell getprop ro.boot.hardware.sku` 返回`GK2MP`，ChatGPT据此判断我的 Pixel 10 固件是美版。美版固件默认屏蔽中国运营商的 IMS 配置，因此会导致 **IMS 未注册、VoLTE 不可用**，且无法通过ADB强制激活：

```Shell
frankel:/ setprop persist.dbg.volte_avail_ovr 1
Failed to set property 'persist.dbg.volte_avail_ovr' to '1'. See dmesg for error reason. 1|frankel:/ $ dmesg dmesg: klogctl: Permission denied
```

ChatGPT说，这是因为Android 15以后的系统安全策略不允许非root用户修改VoLTE/VoWiFi属性。（也许更早版本的Pixel可以这样用ADB激活VoLTE？）

### 刷机的必要性

解决VoLTE不能开启的问题不一定需要刷机，更不需要root。可以先尝试 [IMS+Shizuku](#IMS+Shizuku) 的方案（仍需开启USB调试和下载ADB工具，参考[这些步骤](#准备工作)），不行再考虑刷机。

我的Pixel 10是美版硬件（GK2MP）+Global版系统（BD3A.251005.003.W4），Android 16。我自己是在尝试上述方案之前就刷了新加坡/东南亚版系统（BD3A.251005.003.B7），但我不确定Global版系统是否可以直接用上述方案解决VoLTE问题。据说美版或日版系统有运营商限制，上述方案可能不work，这时就需要刷机。

（虽然刷机可能是条不必要的弯路，但我也是因为在刷机过程中学会了基于ADB的USB调试，尝试 [IMS+Shizuku](#IMS+Shizuku) 时没有遇到这方面的问题。）

> ChatGPT建议我刷机的理由：
>
> - IMS 注册仍依赖系统内置CarrierConfig；
> - 对美版硬件 GK2MP 来说，中国联通 profile 默认不在内置CarrierConfig的IMS白名单里 → IMS 无法注册 → VoLTE 不可用。
> - Global W4 固件虽然是最新稳定版，但美版设备只会加载美版CarrierConfig，而不是 CN/HK profile。
> - 港版 / 新加坡版系统固件改变了 **CarrierConfig**（系统固件里的 XML 配置，位于 `/system/etc/carrier_config/` 或 `/vendor/etc/carrier_config/`），这是系统配置层面的“白名单授权”。
> 
> 查看硬件版本：
> ```Shell
> adb shell getprop ro.boot.hardware.sku
> ```

### 刷机步骤

#### 准备工作

1. 打开Pixel 10的**开发者模式**和**USB调试**

   - 设置 →关于手机，连续点版本号7次，直到提示 “您已成为开发者”
   - 返回 “设置 → 系统 → 开发者选项”，打开**USB调试** 和 **OEM unlocking**
   - 用原装数据线将手机连至电脑

2. 下载[**ADB工具**](https://developer.android.com/studio/releases/platform-tools)，解压，进入platform-tools文件夹，在资源管理器地址栏输入 `cmd` 回车进入终端，检查这两个命令是否正确返回版本号:

   ```Shell
   adb version
   fastboot --version
   ```

3. 备份数据（刷机会清除所有数据）

4. 下载官方镜像包：[Google for Developers](https://developers.google.com/android/images)，并解压

   - 我们选择**BD3A.250721.001.B7**，支持CN运营商。请保证网络稳定并校验 SHA-256（页面有校验码）。

   - ChatGPT给出的版本号说明：

     | 版本代号                    | 发布渠道 / 地区         | 说明                                |
     | --------------------------- | ----------------------- | ----------------------------------- |
     | **BD1A.250702.001**         | Global（全球通用）      | 无运营商锁，适合港版 / 新加坡版设备 |
     | **BD1A.250702.001.A3**      | Verizon（美运营商）     | 🚫 含运营商限制，不可用              |
     | **BD3A.250721.001**         | Global (GStore 2025 07) | ✅ 全球版新代号（推荐）              |
     | **BD3A.250721.001.A1**      | North America GStore    | ⚠️ 美版限定，仍屏蔽 CN IMS           |
     | **BD3A.250721.001.B7**      | 新加坡 / 东南亚版       | ✅ 支持 CN 联通 VoLTE                |
     | **BD3A.250721.001.E1**      | 欧洲 / 英国             | ⚠️ LTE 频段兼容但 IMS 不稳定         |
     | **BD3A.251005.003.J5 / J6** | 日本运营商              | 🚫 屏蔽非日卡 IMS                    |
     | **BD3A.251005.003.W3 / W4** | Global late build       | ✅ 最新全球稳定版                    |

#### 解锁Bootloader（会清除数据）

1. 重启进入bootloader:

   ```shell
   adb reboot bootloader
   ```

2. 进入 fastboot 后检查设备是否被识别：

   ```shell
   fastboot devices
   ```

   应该看到设备序列号。若看不到，可能是驱动未安装，Windows需要下载 **Google USB Driver**，在设备管理器中找到Pixel 10 进行安装。 

3. 进入解压后的镜像包目录，运行一键刷机脚本

   ```shell
   .\flash-all.bat
   ```

#### 刷机后第一次开机与检查

ChatGPT: 进入系统后 **先不要马上插入 SIM 卡或连接 Wi-Fi**（遵循之前店家的忠告：离线全部跳过设置向导，进入主界面后再插卡/联网）。

然而我发现连接Wi-Fi和插入SIM卡的步骤不能跳过（没有skip选项）。ChatGPT说这是因为设备是 **美版 SKU（GK2MP）**，禁止离线激活系统。

解决方案是让电脑用clash开vpn给手机共享热点。注意：

- clash需要打开“**Allow LAN**”
- 手机不要直接连热点WiFi，需要**手动设置代理**，填写主机名（IP，可用ipconfig查看）和端口号（一般是7890）

设置完毕后，恢复数据，大功告成。

输入`*#*#4636#*#*`进入Phone Information一看：

```
IMS registration status: Unegistered
Voice over LTE: Unavailable
```

诶诶诶... ？

### IMS+Shizuku

参考教程：[https://www.youtube.com/watch?v=32bkZnbOyoA](https://www.youtube.com/watch?v=32bkZnbOyoA)

最终在BD3A.251005.003.B7 系统固件中成功解决问题，其他固件就不清楚了。

最后Phone Information应该显示：
```
IMS registration status: Registered
Voice over LTE: Available
```