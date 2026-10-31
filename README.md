# libwrt-ax6

用于 **Redmi AX6（原厂 stock 分区布局，`board_name=redmi,ax6-stock`）** 的 LibWrt 25.12-nss 云编译。

- 基线：LibWrt 上游 `make defconfig` 全量默认配置，**不做任何裁剪**
- 增量：仅追加功能性软件包，且**只取上游官方库**（openwrt base / openwrt packages / openwrt luci）中存在的包
- 仓库与产物**不包含任何本地网络信息**（LAN 网段、SSID、账号、密钥、DNS、IPTV 鉴权等一律不落库；由刷机后的恢复流程注入）

## 构建

Actions → `Build LibWrt (upstream-complete) for Redmi AX6` → Run workflow（可指定 LibWrt 分支，默认 `25.12-nss`）。

流水线内置硬性断言：

- 最终 `CONFIG_PACKAGE_*=y` 数量 **≥ 上游 defconfig**
- 显式 `CONFIG_*=n` 必须为 0（禁止任何裁剪）
- 关键符号逐项 `grep -qx ... =y`

## 产物

`libwrt-qualcommax-ipq807x-redmi_ax6-stock-squashfs-sysupgrade.bin`（附 `sha256sums` 与构建用 `.config`）

刷写前用元数据自检：

```sh
strings -a 固件.bin | grep -o '"supported_devices":\[[^]]*\]'
# 期望：["redmi,ax6-stock"]
```

## 本地配置

固件保留上游默认 LAN（`192.168.1.1/24`）。本地网段、无线、拨号、IPTV、DNS 分流等一律在刷机后通过独立流程注入，不进入本仓库、也不进入镜像。
