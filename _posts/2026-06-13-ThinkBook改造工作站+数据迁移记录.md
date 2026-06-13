---
layout: post
toc: true
categories: tech
title: "ThinkBook改造工作站+数据搬迁记录"
excerpt: "SSD升级+WSL配置+scp传输"
---

最近课题组毕业交接，我需要清空工作站（Dell Precision Tower 7920），但毕竟不是所有组都买得起这种配置，深感由奢入俭难：我需要一个方便写代码、编译、测试、做简单计算的linux系统，快速处理一些没必要上超算的任务。


最方便的办法是在自己的笔记本电脑（ ThinkBook 14 G5+ IRH ）用WSL装一个Ubuntu子系统，但电脑配置堪忧。和AI讨论了一番，它认为这款笔记本的CPU比想象中更能打，瓶颈在硬盘（SSD 512GB，已接近占满）和内存（16GB）。用CPU-Z看到内存是板载焊死的（LPDDR5），无法升级。但硬盘（SSD）依然值得升级，换了电脑可以继续用。

综上，我决定先把笔记本改造成勉强能用的小型工作站（升级SSD+安装WSL子系统），然后直接scp传数据。

数据总共有几个T，但筛选之后只需要搬一小部分（不到500GB左右）：

- 有本地修改的软件: abacus, libri等，注意多个版本，可以git clone再补齐文件，别忘了`.claude`；

- 脚本和试验代码（code lab, scripts ）、notes、latex文章项目：github私人仓库管理；

- 真正需要备份的计算数据（以防万一需要复现文章）
- 近期debug/测试算例（大文件提前批量删除）
- 古早debug/测试直接丢掉；软件几乎全部重新装

### SSD升级

（选购、验货、克隆系统、分区、拆机换盘）

首先确定我的SSD的BusType是NVMe（以免买到不支持的接口类型）：

```shell
PS C:\Users\Fortneu> Get-PhysicalDisk | Select FriendlyName, MediaType, BusType, Size
FriendlyName MediaType BusType Size 
------------ --------- ------- ---- 
Micron MTFDKBA512TFH SSD NVMe 512110190592
```

经过几轮AI的推荐和对比，最终决定购买 WD SN7100 2TB SSD，外加一个NVMe协议的USB-SSD转接盒用于克隆系统。收到SSD后根据 [这篇文档](https://xbnz.yuque.com/sy5cby/vz63l4/vgxiu8y7n9e6vb4c?singleDoc#) 的流程验货；扫二维码得到序列号，去[官网](https://support-en.sandisk.com/app/warrantystatusweb )查询质保。

- 小插曲：我第一次收到的盘，序列号在官网查不到，联系卖家发现错寄成渠道版（OEM，用于组装成整机出货的），于是经历了一次换货。

克隆Windows系统用了DiskGenius的系统迁移功能，原有C、D盘扩充至400GB左右，留出1TB给WSL（后续分E盘）。

拆机注意：

- 拆机前最好查一下螺丝槽的规格，买对应的正规品牌精密螺丝刀。例如我的ThinkBook是十字PH00（后盖）+六角T5（SSD固定）。一开始图便宜省事买了万能工具，结果哪个都不能完美匹配，折腾了半天。好的工具就是不一样。
- 关机——长按10s电源键释放残余电荷——拆后盖——拔掉电源插头（电池到主板的彩色导线连接处）——更换SSD——记得贴散热片、插回插头。

换完之后windows系统和原来的体验完全一样。

### Linux-WSL 数据传输

安装WSL只要一个命令（我指定了安装位置），重启后就可以进Ubuntu了。

```
wsl --install -d Ubuntu-24.04 --location E:\WSL\Ubuntu
```

AI建议添加`C:\Users\username\.wslconfig`文件，限制WSL吃内存，防止windows卡死。（AI建议给windows留一半，我这里取的比较激进）

```shell
[wsl2]
memory=14GB
processors=16
swap=4GB

[experimental]
autoMemoryReclaim=gradual
```

swap指内存不够用时接受多少硬盘空间用于腾挪（性能变慢换不卡）；autoMemoryReclaim 指当Windows内存不够时自动从WSL回收。

然后开始迁移工作站的数据。两个linux系统同时连校园网WIFI，按理说可以直接用 `scp`直接传输，但第一次尝试没反应。这是因为WSL 2 使用的是虚拟化的 NAT（网络地址转换）网络，局域网其他设备无法直接访问，它们只能看到 Windows 主机，看不到主机“内部”这台 WSL 的 IP 地址。解决方法是在**Windows管理员终端设置端口转发**，也就是在Windows主机上开一个入口（端口），将远程机的访问请求转发给 WSL 的 SSH 端口（`22`）：

```shell
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=172.23.217.14
```

 配置Windows 防火墙，允许进入 `2222` 端口的流量

```shell
New-NetFirewallRule -DisplayName "WSL-SSH-Forward" -Direction Inbound -LocalPort 2222 -Protocol TCP -Action Allow
```

现在可以在远程机（工作站）运行scp了。`-P`指定端口`2222`，目标用户名是WSL的，主机名（IP）是Windows的：

```shell
scp -P 2222 -r /local-path username@WindowsIP:/remote-path
```

- WindowsIP是和远程机连同一WIFI（校园网WIFI）的IP地址，不是WSL虚拟网卡地址。在远程机上能ping通才能scp（后者是ping不通的）。

- 若WSL IP变化，需要用以下命令删除旧规则重新配置：

  ```shell
  netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0
  ```

scp容易中断，不能断点续传，终端ssh断了就只能重来，用nohup又不方便监控进度。这里我用了screen：`screen`进入会话——运行scp命令——ctrl+A松开后按D关闭（detach）screen会话，需要时随时可以：

```shell
screen -ls	 #查看session number
screen -d -r <number> 	#监控进度
#然后ctrl+A松开后按D关闭（detach）
```

screen会话终端和普通终端一模一样，`echo $STY`  如果返回带编号的非空值就说明在screen会话内部。如果在内部运行`screen -d -r <number>`会返回有趣的东西（禁止套娃）：

```shell
(base) fortneu@Eisbrecher:~$ echo $STY
9828.pts-2.Eisbrecher
(base) fortneu@Eisbrecher:~$ screen -d -r 9828
Attaching from inside of screen?
```

确认scp结束后，在内部`exit`或 ctrl+D 终止（terminate）screen会话。

### 后续：升级内存（换机）

试了两天发现shell终端非常卡（编译configure阶段、甚至tab命令补全都会卡一分钟以上），和AI讨论后觉得16GB内存瓶颈完全不可接受，于是痛下决心，赶在618尾声买了台非板载内存的新电脑（ThinkPad T14p, 64GB），打算以后放在工位当工作站。旧的ThinkBook 随身携带，远程ssh连接（连Windows或WSL都可以）。经过一番折腾，这条路已经可以走通，总结一下踩过的坑：

- 将带有旧Windows系统的 SSD 0 插到新电脑（SSD 1，已激活Windows）的另一个空插槽：默认启动了 SSD 0，开机F12手动选择 SSD 1 启动，确认C盘在 SSD 1 后格式化了 SSD 0，重启失败（选哪个都不行）。后来发现BIOS关掉 Secure boot 可以选择 SSD 1启动新系统（需要BitLocker恢复密钥就输入），这是因为跳过了“启动链完整性验证”。进入系统后给EFI分区分配盘符S，然后 `bcdboot C:\Windows /s S: /f UEFI` 重建启动链（ 把“Windows启动文件 + BCD启动配置”从系统盘重新复制/生成到EFI分区，并注册为可启动项），此后开启 Secure boot 也能正常启动且不触发BitLocker，圆满解决。
- .wslconfig 设置 `networkingMode=mirrored` 引入很多问题（Windows opensshd 和 WSL ssh因为共享IP导致端口冲突、重置后WSL的网络整个炸掉... ），后来索性改回原始的 NAT 网络+2222端口转发，以上问题全部解决，旧电脑 ssh 连接新电脑 WSL 成功 （WSL用户名@WindowsIP -p 2222）。
- 旧电脑 ssh 连接 新电脑Windows （Windows用户名@WindowsIP）怎么输密码都不对，原因是新电脑Windows用的是Microsoft账户，正确的登录方式应该是：**ssh Microsoft账户邮箱@WindowsIP**（是的，有两个@符号）然后输入邮箱密码。
