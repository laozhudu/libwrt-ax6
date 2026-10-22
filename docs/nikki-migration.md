# nikki 替换 ax6ops —— 实施方案

> 目标：新固件上用 nikki(mihomo) 承接 ax6ops 的透明代理角色，保住泡泡 DNS 体系与去广告。
> 依据：nikki 官方仓库 + Wiki + FAQ 全量资料，以及 AX6/.223 实机取证（2026-10-22）。

---

## 一、结论先行

| 事项 | 结论 |
|---|---|
| nikki 能否承接 7.0.0.0/8 fake-IP 流量 | ✅ 能，且域名还原不依赖映射表（靠 sniffer） |
| 泡泡侧要不要改 | ❌ 不用改。所有改动都在 AX6 |
| ax6ops 的去广告会不会丢 | ⚠️ **会丢**。泡泡侧没有任何去广告能力，必须自建 |
| 主要风险 | DNS 链路切换点唯一，配错即全网解析失败 |

---

## 二、泡泡 DNS 体系（实测）

```
客户端 ──> dnsmasq(192.168.99.1:53)
              │ resolv-file = /etc/ax6ops/resolv.conf
              ▼
        192.168.99.223:53   泡泡客户端口  → 返回 fake-IP 7.0.0.0/8
              │
              ▼
        192.168.99.223:5304 泡泡网关口    → 返回真实 IP（CN/境外分流）
```

实测数据：

| 查询目标 | www.google.com | github.com | 性质 |
|---|---|---|---|
| `.1`（AX6 dnsmasq） | `7.0.0.37` | `7.0.0.34` | fake-IP |
| `.223:53` | `7.0.0.37` | `7.0.0.34` | fake-IP |
| `.223:5304` | `142.251.155.119` | `140.82.113.3` | 真实 IP |
| `.1:1053`（ax6ops mihomo） | `7.0.0.37` | `7.0.0.34` | fake-IP |

泡泡容器内组件：`mosdns` + `unbound`(forward/raw) + `dnscrypt-proxy` ×2 + `redis`。
mosdns 插件清单里**只有** `domain_set` / `forward` / `mmdb` / `sequence` / `udp_server`，
**没有 adlist、没有 reject/blackhole** —— 即泡泡不做去广告。

---

## 三、关键机制：sniffer 才是域名还原的真正手段

ax6ops 的 mihomo 配置：

```yaml
sniffer:
  enable: true
  override-destination: true
  sniff:
    TLS: {ports: [443]}
    HTTP: {ports: [80, 8080-8880]}

dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 7.0.0.0/8
  store-fake-ip: true
  nameserver: ["192.168.99.223:5304", "202.96.134.133", ...]
```

`override-destination: true` 意味着 mihomo 从 **TLS SNI / HTTP Host** 嗅出真实域名并覆盖连接目标。
因此：**谁发 fake-IP 不影响代理能否工作**，sniffer 兜底即可。

这条是本次迁移能低风险落地的根本原因。

---

## 四、接入方案（二选一）

### 方案 B：nikki 自己发号（推荐）

```
客户端 ─> dnsmasq(.1:53) ─> 127.0.0.1#1053 (nikki core, fake-ip 198.18.0.1/16)
                                    │ nameserver
                                    ▼
                              192.168.99.223:5304（泡泡真实解析+分流）
```

AX6 侧改动（泡泡不动）：

```sh
uci set dhcp.@dnsmasq[0].resolvfile=''          # 移除 ax6ops 的 resolv.conf 依赖
uci set dhcp.@dnsmasq[0].server='127.0.0.1#1053'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

优点：fake-IP 映射原生在 mihomo 缓存里，不依赖 sniffer 兜底（sniffer 仍开着做双保险）。
缺点：客户端看到的 fake-IP 段从 `7.x` 变 `198.18.x`（仅影响观感与日志）。

### 方案 A：泡泡继续发号（最小改动）

保持 `dnsmasq → .223:53` 不变，nikki 只接管流量、关掉自己的 DNS 劫持：

```sh
uci set nikki.proxy.ipv4_dns_hijack='0'
uci set nikki.proxy.ipv6_dns_hijack='0'
uci set nikki.proxy.fake_ip_ping_hijack='0'
uci set nikki.mixin.fake_ip_range='7.0.0.0/8'
```

优点：DNS 链路零改动，风险最低。
缺点：完全依赖 sniffer 还原域名（QUIC/非 TLS 流量可能漏），且需实测 mihomo 对"非自己发的 7.x"的处理行为。

> **建议**：先上 B。若发现 QUIC 类应用异常，再评估 A。
> 两者可切换，只需改 dnsmasq 的 `server` 一行。

---

## 五、去广告：必须自建（重点）

### 现状

| 层 | 实现 | 规模 | 内存 |
|---|---|---|---|
| ax6ops | `/etc/ax6ops/dnsmasq.d/ax6ops-ads.conf`（`address=/域名/0.0.0.0`） | **66,390 条 / 2.4MB** | dnsmasq 总 RSS 8.26MB |
| 泡泡 | **无** | — | — |

dnsmasq 通过 `conf-dir=/etc/ax6ops/dnsmasq.d` 加载。放弃 ax6ops = 去广告归零。

### 替代方案

**S1（推荐）：dnsmasq 清单，与现状 1:1**

清单源（hagezi，与 ax6ops 同源，当前 68,940 条）：

```
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/dnsmasq/light.txt
```

格式是 `local=/域名/`（返回 NXDOMAIN，比 `address=/域名/0.0.0.0` 更干净）。

部署（后装，不进固件）：

```sh
mkdir -p /etc/dnsmasq.d
wget -O /etc/dnsmasq.d/adblock.conf \
  https://raw.githubusercontent.com/hagezi/dns-blocklists/main/dnsmasq/light.txt
uci add_list dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'
uci commit dhcp && /etc/init.d/dnsmasq restart
```

白名单：在同目录另建 `whitelist.conf`，用 `server=/域名/#` 放行，dnsmasq 按文件名字母序加载，
`aaa-whitelist.conf` 之类命名可确保先于黑名单生效。

**S2（补充）：mihomo rule-provider REJECT**

```yaml
# 混入配置或 profile
rule-providers:
  adblock:
    type: http
    behavior: domain
    format: text
    url: https://raw.githubusercontent.com/hagezi/dns-blocklists/main/wildcard/light.txt
    path: ./ruleset/adblock.txt
    interval: 86400

rules:
  - RULE-SET,adblock,REJECT
  - GEOSITE,category-ads-all,REJECT
```

`wildcard/light.txt` 是 `*.域名` 格式（1.59MB / 68,940 条），mihomo `behavior: domain` 可直接解析。

> **推荐组合**：S1 为主（DNS 层拦截，省流量、与现状一致），S2 为辅（拦 TLS 层漏网的）。
> 不建议加 AdGuardHome —— 412MB 内存上太重。

### 自动更新

```sh
# /etc/cron.d/adblock（每周一 04:00）
0 4 * * 1 wget -q -O /tmp/ad.conf https://raw.githubusercontent.com/hagezi/dns-blocklists/main/dnsmasq/light.txt && \
  mv /tmp/ad.conf /etc/dnsmasq.d/adblock.conf && /etc/init.d/dnsmasq restart
```

---

## 六、nikki 关键配置项速查

### 核心端口（默认值）

| 用途 | 配置项 | 默认 |
|---|---|---|
| Redirect | `mixin.redir_port` | 7891 |
| TPROXY | `mixin.tproxy_port` | 7892 |
| mixed | `mixin.mixed_port` | 7890 |
| **DNS 监听** | `mixin.dns_listen` | `[::]:1053` |
| API/面板 | `mixin.api_listen` | `[::]:9090` |
| TUN 设备 | `mixin.tun_device` | nikki |

### DNS 角色（UCI 结构化，可写 `IP:端口`）

```sh
config nameserver
    option enabled 1
    option type 'default-nameserver'      # bootstrap
    list nameserver '192.168.99.223:5304'
config nameserver
    option enabled 1
    option type 'nameserver'              # 主解析
    list nameserver '192.168.99.223:5304'
```

可选 `type`：`default-nameserver` / `nameserver` / `proxy-server-nameserver` / `direct-nameserver`。
另有 `nameserver_policy`（按 `matcher` 分域名指派解析器）。

### 内存限制（412MB 必调）

`procd` 段可注入环境变量，默认未设：

```sh
uci set nikki.procd.env_go_max_procs='2'
uci set nikki.procd.env_go_mem_limit='200MiB'
```

对应 `GOMAXPROCS` / `GOMEMLIMIT`，直接约束 Go runtime，是防 OOM 最有效的一招。

### 其它要点

- `failsafe`：`mixin.tun_enabled` 默认 1；不用 TUN 时设 0，省一个 dev 和一份内存
- `config.config.test_profile=1`：启动前校验 YAML，坏配置不落盘
- `config.config.core_only=0`：仅核心模式会禁用混入与防火墙规则，排障时可用
- 访问控制：`router_access_control` / `lan_access_control` 各含 `dns` / `proxy` 开关
- 默认路由标记：`routing.tproxy_fw_mark=0x80`、`tun_fw_mark=0x81`（与 Clashoo/passwall 不冲突）

---

## 七、迁移步骤（刷机后）

```sh
# 0. 确认内核依赖已加载
lsmod | grep -E '^inet_diag|^nft_socket|^nft_tproxy|^tun|^dummy'

# 1. 装 nikki
wget -O - https://github.com/nikkinikki-org/OpenWrt-nikki/raw/refs/heads/main/feed.sh | ash
apk add nikki luci-app-nikki luci-i18n-nikki-zh-cn

# 2. 内存限制
uci set nikki.procd.env_go_max_procs='2'
uci set nikki.procd.env_go_mem_limit='200MiB'

# 3. DNS 上游指向泡泡真实解析
uci add nikki nameserver
uci set nikki.@nameserver[-1].enabled='1'
uci set nikki.@nameserver[-1].type='nameserver'
uci add_list nikki.@nameserver[-1].nameserver='192.168.99.223:5304'

# 4. 方案 B：dnsmasq 指向 nikki
uci set dhcp.@dnsmasq[0].resolvfile=''
uci set dhcp.@dnsmasq[0].server='127.0.0.1#1053'

# 5. 去广告
mkdir -p /etc/dnsmasq.d
wget -O /etc/dnsmasq.d/adblock.conf \
  https://raw.githubusercontent.com/hagezi/dns-blocklists/main/dnsmasq/light.txt
uci add_list dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'

uci commit
/etc/init.d/dnsmasq restart
/etc/init.d/nikki enable && /etc/init.d/nikki start
```

验收：

```sh
# fake-IP 是否由新核心发号
nslookup www.google.com 192.168.99.1
# 出国
curl -s -m 10 -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' https://www.google.com
# 去广告生效
nslookup doubleclick.net 127.0.0.1     # 应 NXDOMAIN
# 内存
free -m
```

---

## 八、ax6ops 功能对照

| ax6ops 能力 | 新方案 | 状态 |
|---|---|---|
| 透明代理 + 路由劫持 | nikki `fw4` + nft | ✅ 等价 |
| DNS 上游托管 | dnsmasq `server` / nikki nameserver | ✅ 需手工重建 |
| 广告拦截（66k 条） | dnsmasq 清单 / mihomo rule-set | ⚠️ 需自建 |
| DDNS | 固件内置 `luci-app-ddns` + `ddns-scripts-cloudflare` | ✅ |
| 泡泡 NAS 集成 | 无对应 | ❌ 丢失 |
| 自定义直连清单 `gw-ipdirect` | nikki 访问控制 / mihomo rules | ⚠️ 需重写 |
| 自愈 `sc-autoheal` | nikki procd respawn + 定时重启 | ⚠️ 语义不同 |
| 出口实测/操作审计面板 | 无对应 | ❌ 丢失 |

---

## 九、风险与回退

| 风险 | 缓解 |
|---|---|
| DNS 配错 → 全网解析失败 | 刷机前把 `uci` 命令写进脚本；保留 SSH 会话；掉线后可从 `192.168.99.1` 直连恢复 |
| nikki 起不来 | `config.config.enabled=0` 即回到纯直连；`core_only=1` 可裸核排障 |
| 内存不足 OOM | `GOMEMLIMIT=200MiB` + `zram-swap`（已编入固件） |
| 去广告误杀 | 白名单文件优先加载；先只跑 1 天观察 |

回退到 ax6ops 思路：`/etc/init.d/nikki stop && /etc/init.d/nikki disable`，
把 dnsmasq 的 `resolvfile` 指回 `/etc/ax6ops/resolv.conf` 即可恢复原链路（前提是 ax6ops 文件仍在）。

---

## 十、待实测确认

1. mihomo 收到**非自己发号**的 `7.x` 假 IP 时的行为（方案 A 的关键）：
   是按 fake-ip 处理、当普通 IP、还是走 sniffer 覆盖。
2. `GOMEMLIMIT=200MiB` 是否触发频繁 GC 拖慢转发 —— 需观察 24h。
3. hagezi `local=/域名/` 格式在 25.12 的 dnsmasq 上是否完全等价于 `address=/域名/0.0.0.0`。
