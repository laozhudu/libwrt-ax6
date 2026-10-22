# ax6ops 模块 → 现成 LuCI 插件映射

> 前提：nikki 只承担"出国网关"这一层（已确认足够）。其余能力尽量用现成插件 + 少量自建脚本，不重写编排层。

---

## 一、总览

| ax6ops 模块 | 作用 | 替代方案 | 类型 | 结论 |
|---|---|---|---|---|
| `gw` | 出国网关开关/节点/订阅 | **nikki** | 编译+后装 | ✅ 更规范 |
| `adb` | 去广告 | **adblock-fast + luci-app-adblock-fast** | 编入固件 | ✅ 更优 |
| `ddns` | CF 动态域名 | **ddns-scripts-cloudflare**（内置）／可选 **ddns-go** | 编入／后装 | ✅ |
| `wg` | WireGuard 组网 | **luci-proto-wireguard**（含二维码） | 编入固件 | ✅ 需迁移枢纽 |
| `ipv6` | 三档开关+体检 | `luci-proto-ipv6` + 手工 uci | 编入 | ⚠️ 无面板 |
| `egress` | 出口实测 | `luci-app-commands` + 脚本 | 编入+自建 | ⚠️ |
| `guard` | 泡泡挂掉自动降级 | **无现成插件** | 自建脚本 | ❌ |
| `tick` | 每分钟网关表自愈 | **无现成插件** | 自建脚本 | ❌ |
| `paopao` | 泡泡状态/验收 | 无 | 手工 nslookup | ❌ |
| `dns` | 分流体检/名单/探针 | 无 | 手工 | ❌ |
| `config-check` | 配置与实机核对 | 无 | — | ❌ |
| `config` | 可视化配置+回滚 | `luci-app-commands` | 部分 | ⚠️ |
| `backup` | 备份轮转 | 无（tar+cron） | 自建 | ⚠️ |
| `history` | 操作审计 | LuCI 系统日志 | 部分 | ⚠️ |
| `mod` | 模块总开关 | 各插件自身开关 | — | ⚠️ |

**可现成插件解决：gw / adb / ddns / wg（4 项）**
**需自建脚本：guard / tick（2 项，最关键）**
**接受损失：paopao / dns / config-check / egress / history（5 项）**

---

## 二、去广告：adblock-fast（推荐）

选它不选传统 `adblock` 的理由（均取自 25.12.2 源）：

| | adblock-fast 1.2.4 | adblock 4.5.9 |
|---|---|---|
| 体积 | **31 KB**（+UI 20KB） | 需同时装 `coreutils`、`gawk` |
| 依赖 | `curl` / `resolveip` / `ucode*`（条件依赖，busybox 有 awk/sed 就不拉 gawk） | `jshn` `jsonfilter` `coreutils` `coreutils-sort` `gawk` `rpcd` `rpcd-mod-rpcsys` |
| 后端 | dnsmasq / unbound / smartdns / nft | dnsmasq / unbound |
| 维护 | 活跃（mossdef-org） | 经典但重 |

对 128MB 闪存 + 412MB 内存的机器，`adblock-fast` 明显更合适。

配置要点：

```
系统 → AdBlock Fast
  启用 → 勾选
  DNS 后端 → dnsmasq
  阻止列表 → 选 hagezi light（或自定义 URL）
```

内核侧清单量参考：hagezi `dnsmasq/light.txt` 当前 **68,940 条**，与 ax6ops 的 66,390 条基本同源。

> 与 ax6ops 的差异：ax6ops 自己写 `address=/域名/0.0.0.0` 到 `/etc/ax6ops/dnsmasq.d`；
> adblock-fast 自己维护清单与 dnsmasq 配置片段，**不需要手工放 conf-dir**。

---

## 三、DDNS：两档

### 内置档（推荐，已编入固件）

`ddns-scripts` + `ddns-scripts-cloudflare`（4 KB）：

```
服务 → 动态 DNS
  → 添加
    查询主机名 = 你的域名
    域名 = 同
    用户名 = CF API Token
    密码 = 同
    服务商 = cloudflare.com
```

轻量、开机即用、有中文界面。缺点：只有单条 A 记录，双栈要建两条。

### 增强档（可选，后装）

`ddns-go`（4 MB，Go）+ `luci-app-ddns-go`：

- 自带独立 Web UI（`:9876`）
- 原生支持 Cloudflare、多域名、IPv4/IPv6 双栈、自动探测网卡
- 界面比 ddns-scripts 直观得多

```sh
apk add ddns-go luci-app-ddns-go luci-i18n-ddns-go-zh-cn
```

> **为什么不编进固件**：`luci-app-ddns-go` 的 `LUCI_DEPENDS:=+ddns-go` 会强制拉 4MB 的 Go 二进制，
> 且源码编译需要 `golang/host` 工具链（CI 多烧 20–40 分钟）。后装零成本。

---

## 四、WireGuard 图形化（重点）

### 现状（实机取证）

```
AX6：无 WG 接口、无 WG 包，只做 DNAT
     nft: udp dport 51820 dnat ip to 192.168.99.2:51820
     （另有 hairpin reflection 两条规则）
 .2：wg0 服务端，172.16.0.1/24，listen 51820，私钥 CFBjKxUK...
     OpenWrt SNAPSHOT / 内核 5.4.272 / 内存 251MB / overlay 剩 57.8MB
     只有 Lua 版 proto_wireguard.lua —— 老界面，无二维码
```

**关键判断：`.2` 的固件太老（内核 5.4 + Lua LuCI），拿不到现代 WireGuard 界面。**

### 三种方案

| 方案 | 傻瓜化程度 | 代价 | 评价 |
|---|---|---|---|
| **A. WG 枢纽迁到 AX6** | ⭐⭐⭐⭐ 二维码 + 客户端配置导出 | 要迁移 peer 清单；`.2` 不再是枢纽 | **推荐** |
| B. 给 `.2` 升级固件 | ⭐⭐⭐⭐ | CR660x 老设备刷机风险 | 不划算 |
| C. `.223` 跑 wg-easy（Docker） | ⭐⭐⭐⭐⭐ 最傻瓜 | NAS 不是网关，路由要额外配 | 不选 |

### 为什么 A 最合适

AX6 **本来就持有公网 IP**（PPPoE 拨号方）。现在靠 DNAT 转给 `.2`，多一跳、多两条规则。
把枢纽搬到 AX6 后：

- **客户端零改动**：endpoint 还是同一个公网 IP、同一端口、同一 172.16.0.0/24
- **只要沿用 `.2` 的私钥**，连密钥都不用换，各站点无感知
- 去掉 DNAT + hairpin reflection 两条规则，链路更短
- `.2` 在同一个二层网络里（192.168.99.2），本来就不需要 WG

### 内置界面能做到什么

`luci-proto-wireguard`（25.12，htdocs 中 `wireguard.js` **904 行**）：

```
buildSVGQRCode(peer_config, node)      ← 生成客户端配置二维码
form.JSONMap({config: {endpoint, allowed_ips, addresses, dns_servers}})
Generate ×10                            ← 一键生成密钥对
PresharedKey / PrivateKey / AllowedIPs  ← 完整字段
```

也就是说：**添加 peer → 填/生成公钥 → 界面直接出客户端配置 + 二维码 → 手机扫码导入**。
这就是 wg-easy 的核心体验，内置已经具备。

### 迁移步骤

```sh
# ① 在 .2 上导出全部 peer（先备份！）
uci show network | grep -E 'wireguard_wg0|private_key|public_key|preshared|allowed_ips|endpoint'

# ② 在 AX6 上建同名接口，沿用 .2 的私钥与端口
#    LuCI: 网络 → 接口 → 添加新接口
#      协议 = WireGuard VPN
#      私钥 = （粘贴 .2 的 private_key）
#      监听端口 = 51820
#      地址 = 172.16.0.1/24
#    然后逐个添加 peer（公钥 / allowed_ips / 可选 PSK）

# ③ 删掉 AX6 上转发到 .2 的 DNAT
uci delete firewall.@redirect[0]      # name=WireGuard-51820
uci commit firewall

# ④ 放行 AX6 本机 51820
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WG-In'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall && /etc/init.d/firewall restart

# ⑤ 关掉 .2 的 wg0，避免抢端口
ssh root@192.168.99.2 'uci set network.wg0.disabled=1; uci commit network; ifup wg0'

# ⑥ 逐站点验证握手
wg show
```

⚠️ **顺序不能错**：先建好 AX6 的 WG 并确认 peer 齐全，再删 DNAT、再停 `.2`。
中间任何一步断，远程访问就没了。

---

## 五、必须自建的两个脚本（无现成插件）

### `guard` —— 泡泡兜底

ax6ops 语义：探 `www.baidu.com`，连续失败 3 次降级直连，连续成功 5 次切回，最短停留 600 秒。

```
每 60s:
  nslookup 探针 → 失败计数++
  失败 >= 3 且 当前非直连 → 把 dnsmasq 上游切成运营商 DNS，标记降级
  成功计数 >= 5 且 当前降级 → 切回 192.168.99.223
  距上次切换 < 600s → 不动
```

### `tick` —— 表/进程一致性

```
每 60s:
  mihomo 进程在 && nft 劫持表存在 → OK
  进程在 && 表没了     → 重建表
  进程没了 && 表还在   → 摘表（否则假 IP 黑洞）
```

这两条是 412MB 机器 + 外部 DNS 依赖下最要紧的自愈能力，**没有现成 LuCI 插件可替代**。

---

## 六、编译决策汇总

| 包 | 编入固件 | 理由 |
|---|---|---|
| `luci-proto-wireguard` | ✅ 12 KB | WG 图形界面 |
| `rpcd-mod-wireguard` | ✅ 12 KB | WG 状态/二维码数据源 |
| `adblock-fast` + UI + zh-cn | ✅ 58 KB | 去广告 |
| `resolveip` | ✅ | adblock-fast 依赖 |
| `ddns-scripts-cloudflare` | ✅ 4 KB | CF DDNS |
| `luci-app-ddns-go` | ❌ | 强制拉 4MB daemon + Go 工具链 |
| `ddns-go` | ❌ 后装 | 4 MB，纯用户态 |
| `nikki` | ❌ 后装 | 3.4 MB |
| `cloudflared` | ❌ 后装 | 8.6 MB |
