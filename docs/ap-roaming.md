# AP 级无缝漫游基线（802.11r / k / v）

> 结论：**当前网络已经实现了 AP 级无缝漫游**，AX6 与 CR660x 两侧参数完全对齐。
> 刷机后只要把这份配置原样恢复，漫游不会断。**无需任何额外软件包**。

---

## 一、实机取证（2026-10-22）

### AX6（本机）

```
radio0 (5G)  channel=60  disabled=0
  ssid=MYDD_5G  encryption=psk2+ccmp
  ieee80211r=1  mobility_domain=4d59
  ieee80211k=1  ieee80211v=1  bss_transition=1

radio1 (2.4G) channel=6   disabled=0
  ssid=MYDD     encryption=psk2+ccmp
  ieee80211r=1  mobility_domain=4d59
  ieee80211k=1  ieee80211v=1  bss_transition=1
```

### CR660x（192.168.99.2）

```
radio0 (2.4G) channel=11
  ssid=MYDD     encryption=psk2+ccmp
  ieee80211r=1  mobility_domain=4d59
  ieee80211k=1  ieee80211v=1  bss_transition=1

radio1 (5G)   channel=149
  ssid=MYDD_5G  encryption=psk2+ccmp
  ieee80211r=1  mobility_domain=4d59
  ieee80211k=1  ieee80211v=1  bss_transition=1

驱动: mt7915e    wpad-openssl 2022-07-29
```

**判据全部满足**：

| 条件 | 状态 |
|---|---|
| 同 SSID | ✅ MYDD / MYDD_5G |
| 同加密与密钥 | ✅ psk2+ccmp |
| 同 Mobility Domain | ✅ `4d59`（两台完全一致） |
| 802.11r 已开 | ✅ 两端 |
| 802.11k / 11v 已开 | ✅ 两端 |
| 信道不重叠 | ✅ 2.4G: 6 vs 11；5G: 60 vs 149 |
| 同二层网段 | ✅ 都在 192.168.99.0/24 |

---

## 二、软件包要求：不需要额外装

| 项 | 依据 |
|---|---|
| `wpad-openssl` | 变体为 `wpad-full-openssl`，`DRIVER_MAKEOPTS` 含 `CONFIG_AP=y CONFIG_MESH=y` |
| **`CONFIG_IEEE80211R=y`** | 已在 `hostapd-basic.config` 与 `hostapd-full.config` 中默认开启 |
| `ieee80211k` / `bss_transition` | hostapd 运行时特性，UCI 开关即可，无需编译选项 |
| ath11k 驱动 | `iw list` 扩展特性含 **RRM**，接口模式含 mesh point（6 项） |
| mt7915e 驱动（.2） | `iw list` 扩展特性含 **RRM** |

> ⚠️ **不要换成 `wpad-basic`** —— 那是精简版，`CONFIG_IEEE80211R` 被注释掉，11r 会直接失效。
> 新固件配置里已显式写死 `CONFIG_PACKAGE_wpad-openssl=y`，防止默认包变动后掉档。

---

## 三、LuCI 操作路径

```
网络 → 无线 → 编辑（对应 radio）→ 接口配置 → 漫游 标签页
```

该标签页提供：

| 选项 | UCI 键 | 应填 |
|---|---|---|
| 802.11r 快速切换 | `ieee80211r` | 1 |
| 移动域 | `mobility_domain` | `4d59`（两台必须相同） |
| FT protocol | `ft_over_ds` | 保持默认 |
| Generate PMK locally | `ft_psk_generate_local` | 1（PSK 场景默认值） |
| 802.11k RRM | `ieee80211k` | 1 |
| BSS Transition (802.11v) | `bss_transition` | 1 |

`ft_psk_generate_local=1` 是关键：两端用**相同的 SSID + 相同的预共享密钥**时，
各自本地推导出同一个 PMK，不需要配 R0KH/R1KH 密钥。
**所以两台的密码必须完全一致**，否则 11r 握手失败、退化成普通漫游。

---

## 四、刷机后必须恢复的配置

### 4.1 无线（`/etc/config/wireless`）

```uci
# 5G
config wifi-iface 'default_radio0'
    option mode 'ap'
    option network 'lan'
    option ssid 'MYDD_5G'
    option encryption 'psk2+ccmp'
    option key '<与原机一致>'
    option ieee80211r '1'
    option mobility_domain '4d59'
    option ieee80211k '1'
    option ieee80211v '1'
    option bss_transition '1'

# 2.4G
config wifi-iface 'default_radio1'
    option mode 'ap'
    option network 'lan'
    option ssid 'MYDD'
    option encryption 'psk2+ccmp'
    option key '<与原机一致>'
    option ieee80211r '1'
    option mobility_domain '4d59'
    option ieee80211k '1'
    option ieee80211v '1'
    option bss_transition '1'
```

**信道**：AX6 用 `60`(5G) / `6`(2.4G)；CR660x 用 `149`(5G) / `11`(2.4G)。
两边**不要选同一信道**，否则漫游时同频干扰严重，反而更卡。

### 4.2 其他必须恢复的（与本主题相关）

| 项 | 说明 |
|---|---|
| `kmod-ath11k` 系列 | DEFAULT_PACKAGES 已含 |
| 国家码（`country`） | 必须在两个 radio 上填同一个（如 `CN`），否则 149 等高信道不可用 |
| `disabled=0` | 新固件首次启动无线默认关闭，需手动开 |

---

## 五、常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 完全不漫游，粘在原 AP | 客户端（尤其 iOS/部分安卓）有粘性 | 11k/11v 已开可缓解；仍粘性是客户端行为 |
| 切换要 3–5 秒 | 11r 未生效 | 检查 `mobility_domain` 是否两端一致、密码是否一致 |
| 切过去连不上 | `wpad-basic`（无 11r） | 换 `wpad-openssl` |
| 5G 找不到 | `country` 未设或两端不一致 | 两台都设 `CN` |
| 漫游后断网 | 新 AP 的 LAN 不在同一二层 | 都接到同一 `br-lan` / 同一交换机 |

---

## 六、验证方法

```sh
# 1. 确认 11r/11k/11v 已下发到 hostapd
ps w | grep hostapd                       # 看启动参数里的 conf 文件
iw dev phy0-ap0 station dump | head -20   # 已关联客户端列表

# 2. 看客户端是否发生快速切换（在 AX6 上抓 4-way handshake 次数）
#    11r 生效时，从 .2 切过来不会有完整的 4 次握手
tcpdump -i br-lan -n 'ether proto 0x888e' -c 20

# 3. 实测漫游延迟
#    手机开一个持续 ping，在房间之间走动，看丢包与 RTT 抖动
ping -i 0.2 192.168.99.1
```

---

## 七、与 Mesh 的关系

- **AP 级无缝漫游**（本文档）= 多个 AP 用**有线**回程 + 11r/k/v 快速切换。**当前就是这个模式**。
- **802.11s / batman-adv Mesh** = **无线**回程组网，没有网线时才用。

两者的 11r/k/v 配置是一样的，但 **Mesh 会占用一个 radio 做回程**。
AX6 只有 2 个 radio，开 Mesh 后必定牺牲一个频段的覆盖。

> **建议**：家里能拉网线就保持现在的「有线回程 + 11r/k/v」，
> 固件里编入 `MAC80211_MESH` 和 `batman-adv` 只是留作**将来没网线时的备选**，默认不启用。
