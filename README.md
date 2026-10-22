# LibWrt for Redmi AX6 (stock layout) — 满血自编译

为 **Redmi AX6（stock 分区布局）** 编译 LibWrt 25.12-nss 固件，保留 NSS 硬件加速，
并补齐上游默认缺失的内核模块与网络组件。

## 为什么必须用 stock layout

本机实测（`/proc/mtd` + `ubinfo -a`）：

| 项 | 值 |
|---|---|
| board_name | `redmi,ax6-stock` |
| model | `Redmi AX6 (stock layout)` |
| 分区来源 | SMEM（`14 qcomsmem partitions found`） |
| rootfs 容器 | `mtd12 @0xa00000`，102.25 MiB |
| UBI 卷 | `kernel` 5.45 MB / `rootfs` 23.49 MB / `rootfs_data` 67.21 MB |

`redmi_ax6`（无 `-stock`）要求 `ubi_kernel@0xa00000` + `rootfs@0x2dc0000` 双分区，
本机没有 `ubi_kernel`，**刷了必砖**。

镜像兼容性可用元数据自查：

```sh
strings -a 固件.bin | grep -o '"supported_devices":\[[^]]*\]'
# 必须包含 "redmi,ax6-stock"
```

## 关键内核模块

上游默认 *未* 编译，导致 Clashoo / nikki / dae 等依赖报 `no such package`：

| 包 | 自动打开的内核选项 |
|---|---|
| `kmod-dummy` | `CONFIG_DUMMY` |
| `kmod-nft-socket` | `CONFIG_NFT_SOCKET` |
| `kmod-sched-bpf` | `CONFIG_NET_CLS_BPF` + `CONFIG_NET_ACT_BPF` |
| `kmod-sched-core` | `CONFIG_NET_SCH_INGRESS` + `CONFIG_NET_CLS_ACT` 等 19 项 |

在 `.config` 里选中这些包即可，无需改内核 config 文件。

## 构建

推送到 `main` 或手动触发 `workflow_dispatch`。

产物在 Actions 的 artifacts 里，包含 sysupgrade 镜像与 `.config`。

## 刷机

```sh
sysupgrade -n /tmp/openwrt-qualcommax-ipq807x-redmi_ax6-stock-squashfs-sysupgrade.bin
```

刷机前务必确认镜像元数据含 `redmi,ax6-stock`。
