# iStoreOS · Orange Pi R1 Plus LTS 专修分支

> 基于 **[istoreos/istoreos](https://github.com/istoreos/istoreos)** 官方源码
> 分支 **`istoreos-25.12`**（commit `0465e2cf`）的 fork。
>
> **唯一目的：让 Orange Pi R1 Plus LTS 在主线 Linux 内核下，WAN 口能正常工作。**
>
> 除本文档「本分支相对上游的全部改动」一节列出的内容外，与官方源码**完全一致**。

---

## 一、这是什么项目

Orange Pi R1 Plus LTS 这块板子有一个**持续了三年多、一直没被公开解释清楚**的问题：

> 刷第三方固件 / 自己编译的固件后，**WAN 口（原生千兆口）不通** ——
> 链路能协商到 1 Gbps、能收到包，但**发出去的包对端收不到**，
> 所以 DHCP 拿不到 IP、ping 不通、等于没用。

同样的板子在**香橙派官方 OpenWrt 固件**下却一切正常。这导致社区长期流传的解决办法只有三种：

1. 移植香橙派厂商版 PHY 驱动（不会打补丁的人做不了）；
2. 退回 5.10 内核（厂商补丁能直接打上）；
3. 下载别人编译好的固件。

**没有人公开过真正的根因，也没有给出可上传的最小补丁。**

这个 fork 就是来终结这件事的：**根因已定位到寄存器级，修复只有一个 13 行的内核补丁**，
并且已经过真机端到端验证。

---

## 二、根因：PHY 的参考时钟被配成了 25 MHz

R1 Plus LTS 的 WAN 口用的是 **YT8531C** PHY（MDIO 地址 0），
设备树里是 `clock_in_out = "input"`，意思是：

> **RK3328 GMAC 的参考时钟不由 SoC 提供，而是由 PHY 的 CLKOUT 引脚提供，必须 125 MHz。**

而实测这块板子上该寄存器是错的：

| PHY 扩展寄存器 `0xa012` | 实际值 | 含义 |
|---|---|---|
| 故障态 | `0x00c8` | 选了 **25 MHz** 参考源（`CLK_SRC_REF_25M`）❌ |
| 正常态 | `0x00d0` | `SYNCE_ENABLE` + `CLK_FRE_SEL_125M` + `CLK_SRC_PLL_125M` ✅ |

25 MHz 的后果就是极其典型的**「能收不能发」**：PHY 和 MAC 之间 TX 时钟是死的，
RX 完全正常 —— 所以链路一切正常、能收到对端的包，但本机发出去的帧出不去。

### 为什么内核「看起来设对了」却还是错

1. DTS 里确实写了 `motorcomm,clk-out-frequency-hz = <125000000>`；
2. 上游 `motorcomm.c` 的 `yt8531_probe()` 确实会把它配成 125 MHz；
3. **但 `yt8531_probe()` 只在 probe 那一瞬间执行一次**；
4. 而 `phy_init_hw()` 每次都会做「**PHY 软复位 → 调用 `config_init()`**」，
   **软复位会把 `0xa012` 打回硬件默认的 `0x00c8`**；
5. 上游的 `yt8531_config_init()` **从来不重新写这个寄存器**。

结果：probe 时正确，第一次复位后（比如 `ip link set eth0 up` 或网卡重新初始化）
就永久退化成 25 MHz。香橙派厂商版驱动是在每次 reset 之后重新应用的，
所以他们自己的固件正常、主线内核不正常。

### 实测证据

```
复位前：0xa012 = 0x00d0      复位后（ip link down/up）：0xa012 = 0x00c8
写入 0xa012 = 0x00d0 后立刻测 DHCP：
  udhcpc: lease of 192.168.8.63 obtained from 192.168.8.1
  ping 192.168.8.1 → 0% packet loss
```

一个字都不改、只写这一个寄存器，联网立刻恢复 —— 这直接锁定了根因。
（写完如果再 `ip link down/up`，联网立刻又坏，从反面再次验证。）

---

## 三、修复：一个 13 行的内核补丁

本分支新增的内核补丁（**最小改动，单 hunk**）：

```
target/linux/rockchip/patches-6.12/990-net-phy-motorcomm-reapply-clk-out-config.patch
```

```c
--- a/drivers/net/phy/motorcomm.c
+++ b/drivers/net/phy/motorcomm.c
@@ -1716,6 +1716,18 @@
 	if (ret < 0)
 		return ret;
 
+	/*
+	 * The clock output configuration set up in .probe() is lost whenever the
+	 * PHY is reset.  config_init() runs right after every reset (see
+	 * phy_init_hw()), so re-apply it here.  Without this the YT8531 clock
+	 * output stays at the reset default of 25 MHz, which breaks boards whose
+	 * GMAC reference clock comes from the PHY clock output
+	 * (clock_in_out = "input"), such as the Orange Pi R1 Plus LTS.
+	 */
+	ret = yt8531_probe(phydev);
+	if (ret < 0)
+		return ret;
+
 	return 0;
 }
```

**思路**：不新增任何寄存器魔法，只是把上游**自己那套**经过验证的 CLKOUT 配置，
挪到 `yt8531_config_init()` 的末尾再执行一次。
因为 `config_init()` 在**每次 PHY 复位之后都会被 `phy_init_hw()` 调用**，
复位就再也抹不掉 125 MHz 了。

> 说明：早期尝试过在 `yt8531_probe()` 里额外写扩展寄存器 `0x0c`（时钟门控寄存器），
> 实测证明**完全没必要**（只写 `0xa012` 就够了），因此最终补丁里没有它 ——
> 改动越少越安全。

---

## 四、本分支相对上游的全部改动

一共 **3 处修改 + 4 个新增文件**，全部列在这里，没有任何隐藏改动：

| 类型 | 路径 | 说明 |
|---|---|---|
| **新增** | `target/linux/rockchip/patches-6.12/990-net-phy-motorcomm-reapply-clk-out-config.patch` | **核心修复**：PHY CLKOUT 125 MHz 复位后重应用 |
| **新增** | `package/base-files/files/etc/uci-defaults/95-ipv6-defaults` | 首启固化 IPv6 默认开启（WAN+LAN），幂等 |
| **新增** | `package/istoreos-files/files/usr/bin/istore-install` | iStoreOS 官方扩展包一键安装脚本 |
| **新增** | `package/istoreos-files/files/etc/apk/repositories.d/istore-nas.list` | 预置 iStoreOS 官方 apk 源 |
| 修改 | `include/version.mk` | 默认软件源换成国内镜像 `mirrors.cernet.edu.cn` |
| 修改 | `package/istoreos-files/Makefile` | 把 `distfeeds.list` 里的源地址也换成上面的镜像 |
| 修改 | `package/network/services/dnsmasq/files/dhcp.conf` | `filter_aaaa` 默认值 `1` → `0`（不过滤 AAAA，否则等于对外关掉 IPv6 解析） |

另外还有一份**可选**的 feeds 补丁（不属于本仓库，见 `docs/orangepi-r1-plus-lts/`）：

| 文件 | 说明 |
|---|---|
| `feeds-docker-offline-all.patch` | 去掉 `feeds/packages/utils/{docker,dockerd}` 里访问 GitHub 的 commit 校验块，供无法直连 GitHub 的环境离线编译 |

**没有做的事**：
- 没有删减 iStoreOS 的功能；

---

## 五、不想自己编译？直接用现成镜像

已构建好的镜像（**iStoreOS 25.12.5 / Linux 6.12.94 / 389 个软件包**）：

```
istoreos-rockchip-armv8-xunlong_orangepi-r1-plus-lts-squashfs-sysupgrade.img.gz
大小   : 88,628,384 字节
SHA256 : b5ac199c9313f2cbc70a19ff0b39ff274fd1c5c127129c3226de450000acfa25
```

> 该镜像使用balenaEtcher刷写固件时会在最后检查阶段报错，但不影响实际成功，故没有修复。
> 该镜像不能用 LuCI 的「刷写固件」再刷它（那需要元数据），全新刷机没有任何影响。

**刷机**：

1. 用 ≥ 4 GB 的 TF 卡，balenaEtcher 或 Rufus（Rufus 选 DD 镜像模式）写入；
2. 插入板子，**注意网口对应关系**：

   | 口 | 设备名 | 位置 |
   |---|---|---|
   | **WAN** | `eth0` | 原生 GMAC + YT8531C，**远离 USB 接口**的那个网口 |
   | **LAN** | `eth1` | USB 网卡 RTL8153，**靠近 USB 接口**的那个网口 |

3. 上电，首次启动 1~2 分钟（生成配置、初始化 overlay）；
4. 浏览器打开 **192.168.100.1**，默认账号：root，默认密码：password，首次进入会被要求**设置 root 密码**。

---

## 六、自己编译

### 1) 取源码

```sh
git clone -b istoreos-25.12 https://github.com/istoreos/istoreos.git
cd istoreos
git remote add mine https://github.com/<你的用户名>/istoreos.git   # 或者直接用本仓库
git fetch mine istoreos-25.12 && git checkout mine/istoreos-25.12   # 带上本项目的改动
```

> 本仓库就是官方 `istoreos-25.12` 分支 + 上面那 7 处改动，所以直接从本仓库出分支即可。

### 2) 拉 feeds

```sh
./scripts/feeds update -a
./scripts/feeds install -a
```

### 3) 配置

```sh
make menuconfig
```

关键选项（本项目的完整最小配置见 `docs/orangepi-r1-plus-lts/config.diff`）：

```
Target System    -> Rockchip
Subtarget        -> armv8
Target Profile   -> Xunlong Orange Pi R1 Plus LTS     # DEVICE_xunlong_orangepi-r1-plus-lts
Target Images    -> Kernel partition size = 16 MiB
                    Root filesystem partition size = 864 MiB
```

> 分区要放大到 864 MiB 是因为内置了 Docker，否则会报
> `ext4_allocate_best_fit_partial: failed to allocate ... blocks`。

直接把 `config.diff` 复制成 `.config` 也可以，然后 `make defconfig`：

```sh
cp docs/orangepi-r1-plus-lts/config.diff .config
make defconfig
```

### 4) 编译

**建议串行编译**（这个树并行编译容易出现连锁失败）：

```sh
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin   # 清掉 Windows 路径
export FORCE_UNSAFE_CONFIGURE=1
export GOPROXY=https://goproxy.cn,direct
export GOSUMDB=off

make -j1 V=s 2>&1 | tee build.log
```

产物在 `bin/targets/rockchip/armv8/`。

### 5) 常见坑

| 现象 | 原因 / 处理 |
|---|---|
| `docker` 编译报 `error: remote origin already exists` | `feeds/packages/utils/{docker,dockerd}` 会在 `Build/Prepare` 里 `git fetch` github.com 校验 commit。无法直连 GitHub 时打 `docs/orangepi-r1-plus-lts/feeds-docker-offline-all.patch` |
| `docker-compose` 卡在 `go list`（SYN-SENT 到 proxy.golang.org） | 设 `GOPROXY=https://goproxy.cn,direct` + `GOSUMDB=off` |
| `ppp` 报 `error: 'VERSION' undeclared` | 残留的 `config.h`，`make package/network/services/ppp/clean` 后重编 |
| 编译中途莫名 `target/linux failed to build` | 终端掉线导致的 SIGHUP，用 `setsid nohup ... &` 方式跑，别挂在交互终端上 |
| 编译到一半整树重编 | `target/install` 阶段 kmod 包会调 `make modules` 触发，属正常现象，等它跑完 |
| `luci-app-store` 失败 | 需要 `istore-ui-v0.2.0-2.tar.gz`，放一份到 `dl/` |

---

## 七、实测验证结果

### 内核补丁（真机，换内核 + 重启，**零手工干预**）

```
Linux iStoreOS 6.12.94 #0 SMP aarch64

dmesg:
  eth0: PHY [stmmac-0:00] driver [YT8531 Gigabit Ethernet]
  eth0: configuring for phy/rgmii-id link mode
  eth0: Link is Up - 1Gbps/Full - flow control rx/tx
  eth0: Link is Down            ← PHY 复位了一次
  eth0: Link is Up - 1Gbps/Full

0xa012 = 0x00d0                 ← 复位后仍然是 125 MHz ✅
wan: "up": true, ipv4 192.168.8.66/24
default via 192.168.8.1 dev eth0
ping 223.5.5.5 → 0% packet loss ✅
DNS 解析 baidu.com → 正常 ✅
```

---

## 八、已知限制（如实说明）

1. **不要用官方 iStoreOS 同版本镜像覆盖升级** —— 那里面没有这个 PHY 补丁，**WAN 会再次失效**。
   要升级请用本仓库重新编译。

2. 自编译内核模块的哈希与官方源不一致，`apk update` 时 **kmods 那一行会 404**。
   这是所有自编译 OpenWrt 的共有现象，不影响使用；用户态软件包不受影响。

3. 交付镜像去掉了 sysupgrade 元数据（为了 Etcher 兼容），不适合用 LuCI 的「刷写固件」再刷它。
   若需要带元数据的版本，自己编译即可（默认产物就是带元数据的）。

---

## 九、补丁问题详情

这个补丁解决的是一个**通用问题**，不只是这一块板子：
任何 **`clock_in_out = "input"` + YT8531 系列 PHY（PHY 提供 GMAC 参考时钟）** 的设计，
只要 PHY 复位一次就会丢配置。

`phy_init_hw()` 的顺序是「软复位 → `config_init()`」，所以把 CLKOUT 配置放进
`config_init()` 是语义正确的位置。如果维护者认同，这个补丁可以直接提交到 `netdev` /
`linux-phy`。有渠道的朋友欢迎帮忙转达或提交。

> 顺带一提：2026 年 7 月上游有一批
> `net: phy: motorcomm: enable the reference clock for YT8531` 的补丁，
> 但那解决的是**反方向**的问题（SoC 给晶振less PHY 提供 25 MHz 输入时钟），
> 与本项目「PHY 给 SoC 提供 125 MHz 输出时钟、且复位后丢失」是两回事。

---

## 十、许可与致谢

* 本项目是 **[iStoreOS](https://github.com/istoreos/istoreos)** / **OpenWrt** 的衍生作品，
  全部版权归原作者所有，遵循其原有许可（GPL-2.0 等）。
  本仓库只设置**源码改动**，漏洞修复，未添加原有固件中没有的功能。
* 内核补丁部分同样以 **GPL-2.0** 发布，欢迎任意使用、转发、上游化。
* 预置的 iStoreOS 官方扩展源地址归 iStoreOS 官方所有。
* **仅供学习交流使用**，请遵守当地法律法规；刷机有风险，请自行备份数据。

---

## 附：目录说明

```
docs/orangepi-r1-plus-lts/
├── README.md                        # 本板子详细的构建 / 刷机 / 排查说明
├── 诊断记录.md                       # 完整诊断过程与实测寄存器数据
├── config.diff                      # 本项目的最小构建配置（可直接当 .config 用）
└── feeds-docker-offline-all.patch   # 可选的 feeds 离线化补丁
```
