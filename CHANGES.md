# Intel FM10840 (fm10k) 网卡驱动修订记录

## 概述

本文档记录了 Intel FM10840 网卡驱动在 Linux 7.0 内核上的适配修改，包括内核兼容性适配、ethtool 功能恢复、编译警告修复三部分。编译目标：**0 Error, 0 Warning**。

## 修订文件列表

| 文件 | 修改类型 |
|------|----------|
| `src/fm10k/src/fm10k_ethtool.c` | 内核兼容性 + ethtool 功能恢复 + Warning 修复 |
| `src/fm10k/src/fm10k_main.c` | 内核兼容性 + Warning 修复 |
| `src/fm10k/src/fm10k_netdev.c` | 内核兼容性 + Warning 修复 |
| `src/fm10k/src/fm10k_pci.c` | 内核兼容性 |
| `src/fm10k/src/fm10k_pf.c` | Warning 修复 |
| `src/fm10k/src/fm10k_mbx.c` | Warning 修复 |
| `src/fm10k/src/kcompat.h` | 内核兼容性 |

---

## 第一部分：Linux 7.0 内核兼容性修改

### 1. fm10k_ethtool.c — ethtool 函数签名适配

内核 7.0 对 `ethtool_ops` 结构体中多个函数签名进行了变更，新增了 `kernel_ethtool_*` 和 `netlink_ext_ack` 参数：

| 函数 | 变更内容 |
|------|----------|
| `fm10k_get_ringparam` | 新增 `struct kernel_ethtool_ringparam *kernel_ring` 和 `struct netlink_ext_ack *extack` 参数 |
| `fm10k_set_ringparam` | 同上 |
| `fm10k_get_coalesce` | 新增 `struct kernel_ethtool_coalesce *kernel_coal` 和 `struct netlink_ext_ack *extack` 参数 |
| `fm10k_set_coalesce` | 同上 |
| `fm10k_get_rssh` | 将旧版多参数接口改为 `struct ethtool_rxfh_param *rxfh` 统一接口 |
| `fm10k_set_rssh` | 同上，并新增 `struct netlink_ext_ack *extack` 参数 |
| `fm10k_get_rxnfc` | 将 `void *rule_locs` 修正为 `u32 *rule_locs`（消除类型不匹配） |

### 2. fm10k_netdev.c — net_device_ops 适配

- `fm10k_tx_timeout`：新增 `unsigned int txqueue` 参数
- `ether_addr_copy(dev->dev_addr, ...)` → `dev_addr_set(dev, ...)`
- `u64_stats_fetch_begin_irq` → `u64_stats_fetch_begin`
- `u64_stats_fetch_retry_irq` → `u64_stats_fetch_retry`

### 3. fm10k_main.c — NAPI 初始化适配

- `netif_napi_add()` 调用移除 `NAPI_POLL_WEIGHT` 参数（内核 7.0 中该参数已内置）

### 4. fm10k_pci.c — PCI 子系统适配

- 新增 `#include <linux/timer.h>`
- `from_timer()` → `container_of()`（timer 结构体变更）
- `ether_addr_copy(netdev->dev_addr, ...)` → `dev_addr_set(netdev, ...)`
- `del_timer_sync()` → `timer_delete_sync()`

### 5. kcompat.h — XDP 内存管理适配

- `xdp_umem_get_data` 和 `xdp_umem_get_dma` 函数适配新的 `umem->pgs` 页面数组接口

---

## 第二部分：ethtool 功能恢复

### 背景

由于内核 7.0 的 `ethtool_check_ops` 函数会对 `ethtool_ops` 中的函数签名进行严格校验，原始修改中临时注释了 `fm10k_set_ethtool_ops(dev)` 注册调用以绕过校验。本次修订恢复了 ethtool 功能的注册，并修复了函数签名不匹配问题。

### 已恢复的 ethtool 功能

| ethtool 子命令 | 功能 | 状态 |
|---------------|------|------|
| `ethtool enp1s0` | 显示网卡基本设置 | ✅ 正常 |
| `ethtool -i enp1s0` | 显示驱动信息 | ✅ 正常 |
| `ethtool -g enp1s0` | 显示/设置环形缓冲区参数 | ✅ 正常 |
| `ethtool -c enp1s0` | 显示/设置聚合（coalesce）参数 | ✅ 正常 |
| `ethtool -C enp1s0` | 设置聚合（coalesce）参数 | ✅ 正常 |
| `ethtool -k enp1s0` | 显示协议卸载特性 | ✅ 正常 |
| `ethtool -a enp1s0` | 显示暂停帧参数 | ✅ 正常 |
| `ethtool -S enp1s0` | 显示网卡统计计数器 | ✅ 正常 |
| `ethtool -d enp1s0` | 寄存器转储 | ✅ 正常 |
| `ethtool -t enp1s0` | 自测试 | ✅ 正常 |
| `ethtool -T enp1s0` | 时间戳信息 | ✅ 正常 |
| `ethtool --show-priv-flags enp1s0` | 私有标志 | ✅ 正常 |
| 网卡统计 | `get_strings` / `get_sset_count` / `get_ethtool_stats` | ✅ 正常 |
| RSS 哈希 | `get_rxfh` / `set_rxfh` | ✅ 正常 |
| 流分类 | `get_rxnfc` / `set_rxnfc` | ✅ 正常 |
| 暂停帧 | `get_pauseparam` / `set_pauseparam` | ✅ 正常 |

---

## 第三部分：编译警告修复

### 修复项

编译目标为 **0 Error, 0 Warning**。修复包括两类：

#### A. `-Wunused-function` 警告

`fm10k_set_coalesce` 因在 ethtool_ops 中禁用，其函数体原用 `#if 0 ... #endif` 包裹。在添加 `supported_coalesce_params` 并启用 `set_coalesce` 后，该警告自动消除。

#### B. `-Wimplicit-fallthrough` 警告

以下为原有代码中 switch-case 故意穿透（fallthrough），虽已有 `/* fall through */` 注释但编译器要求显式 `fallthrough;` 伪关键字：

| 文件 | 函数 | 穿透路径 |
|------|------|----------|
| `fm10k_ethtool.c` | `fm10k_get_rss_hash_opts` | TCP_V4/V6 → UDP_V4 → SCTP_V4 |
| `fm10k_main.c` | `fm10k_tx_csum` | IPPROTO_GRE → default |
| `fm10k_netdev.c` | `fm10k_clear_macvlan_queue` | MAC_REQUEST → VLAN_REQUEST |
| `fm10k_pf.c` | `fm10k_iov_supported_xcast_mode_pf` | PROMISC → ALLMULTI → MULTI |
| `fm10k_mbx.c` | `fm10k_mbx_validate_msg_hdr` | DISCONNECT → DATA, CONNECT → ERROR |
| `fm10k_mbx.c` | `fm10k_pfvf_mbx_init` | fm10k_mac_pf → default |

---

## 应用补丁

在 fm10k 源码目录下执行：

```bash
git apply fm10k-kernel-7.0-full-compat.patch
```

然后编译驱动：

```bash
cd src/fm10k/src
make -j$(nproc)
```

加载驱动：

```bash
rmmod fm10k     # 卸载旧驱动
insmod fm10k.ko # 加载新驱动
```

---

## 修订历史

| 日期 | 修订内容 |
|------|----------|
| 2026-05-22 | 1. 恢复 ethtool 全部功能：取消注释 `fm10k_set_ethtool_ops`，修复函数签名不匹配，适配 ringparam/coalesce/rxfh/ts_info 等回调。2. 修复 `set_coalesce` probe 失败问题：添加 `.supported_coalesce_params` 字段（`ETHTOOL_COALESCE_USECS | ETHTOOL_COALESCE_USE_ADAPTIVE`），使 `ethtool -C` 读写聚合参数正常工作。3. 修复全部 `-Wimplicit-fallthrough` 警告（6 处，跨 5 个文件）。编译结果 0 Error, 0 Warning。 |
