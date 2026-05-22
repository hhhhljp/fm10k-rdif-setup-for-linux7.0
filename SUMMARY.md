# Intel FM10840 (fm10k) 驱动适配 Linux 7.0 — 最终代码变更总结报告

## 一、项目背景

Intel FM10840 网卡驱动及 RDIF 软件在 Linux 7.0 内核环境下，因内核 API 变更导致编译警告、ethtool 功能缺失等问题。本次修订对驱动代码进行了系统性适配与修复，使驱动在 Linux 7.0 内核上 **0 Error, 0 Warning** 编译通过，ethtool 全部功能恢复正常，RDIF / UIO 正常工作。

---

## 二、修改文件清单（7 个文件，+67 / -61 行）

| 文件 | 修改行数 | 修改类型 |
|------|----------|----------|
| `src/fm10k/src/fm10k_ethtool.c` | +42 / -26 | 内核兼容性 + ethtool 恢复 + Warning 修复 |
| `src/fm10k/src/fm10k_netdev.c` | +6 / -12 | 内核兼容性 + Warning 修复 |
| `src/fm10k/src/fm10k_pci.c` | +5 / -9 | 内核兼容性 |
| `src/fm10k/src/kcompat.h` | +10 / -2 | 内核兼容性 |
| `src/fm10k/src/fm10k_main.c` | +2 / -2 | 内核兼容性 + Warning 修复 |
| `src/fm10k/src/fm10k_mbx.c` | +3 / -3 | Warning 修复 |
| `src/fm10k/src/fm10k_pf.c` | +3 / -3 | Warning 修复 |

---

## 三、核心功能点详解

### A. 内核兼容性适配（5 类 API 变更）

| 类别 | 文件 | 原 API | 新 API | 说明 |
|------|------|--------|--------|------|
| ethtool 签名 | `fm10k_ethtool.c` | `get_ringparam(dev, ring)` | `get_ringparam(dev, ring, kernel_ring, extack)` | 内核 7.0 新增 kernel 和 extack 参数 |
| ethtool 签名 | `fm10k_ethtool.c` | `set_ringparam(dev, ring)` | `set_ringparam(dev, ring, kernel_ring, extack)` | 同上 |
| ethtool 签名 | `fm10k_ethtool.c` | `get_coalesce(dev, ec)` | `get_coalesce(dev, ec, kernel_coal, extack)` | 同上 |
| ethtool 签名 | `fm10k_ethtool.c` | `set_coalesce(dev, ec)` | `set_coalesce(dev, ec, kernel_coal, extack)` | 同上 |
| ethtool 签名 | `fm10k_ethtool.c` | `get_rssh(dev, indir, key, hfunc)` | `get_rssh(dev, rxfh_param)` | 多参数合并为结构体 |
| ethtool 签名 | `fm10k_ethtool.c` | `set_rssh(dev, indir, key, hfunc)` | `set_rssh(dev, rxfh_param, extack)` | 同上 + extack |
| 参数类型修复 | `fm10k_ethtool.c` | `get_rxnfc(dev, cmd, void *rule_locs)` | `get_rxnfc(dev, cmd, u32 *rule_locs)` | 消除类型不匹配 |
| net_device_ops | `fm10k_netdev.c` | `tx_timeout(dev)` | `tx_timeout(dev, txqueue)` | 新增参数 |
| net_device_ops | `fm10k_netdev.c` | `ether_addr_copy(dev->dev_addr, ...)` | `dev_addr_set(dev, ...)` | 内核 API 变更 |
| net_device_ops | `fm10k_netdev.c` | `u64_stats_fetch_begin_irq` / `_retry_irq` | `u64_stats_fetch_begin` / `_retry` | _irq 版本已移除 |
| NAPI | `fm10k_main.c` | `netif_napi_add(dev, napi, poll, NAPI_POLL_WEIGHT)` | `netif_napi_add(dev, napi, poll)` | 权重参数已内置 |
| PCI | `fm10k_pci.c` | `from_timer(ptr, t, member)` | `container_of(t, type, member)` | timer 结构体变更 |
| PCI | `fm10k_pci.c` | `del_timer_sync(...)` | `timer_delete_sync(...)` | API 重命名 |
| PCI | `fm10k_pci.c` | `ether_addr_copy(netdev->dev_addr, ...)` | `dev_addr_set(netdev, ...)` | 同上 |
| XDP | `kcompat.h` | `umem->pages[].addr` | `pfn_to_kaddr(page_to_pfn(umem->pgs[]))` | 页面数组接口变更 |
| XDP | `kcompat.h` | `umem->pages[].dma` | `page_to_phys(umem->pgs[])` | 同上 |

### B. ethtool 功能恢复（关键修复）

| 功能 | 修复方式 | 状态 |
|------|----------|------|
| 整体注册 | 取消注释 `fm10k_set_ethtool_ops(dev)` | ✅ |
| `set_coalesce` probe 失败 | 添加 `.supported_coalesce_params = ETHTOOL_COALESCE_USECS \| ETHTOOL_COALESCE_USE_ADAPTIVE` | ✅ 读写正常 |
| `get_rxnfc` 参数类型 | `void *rule_locs` → `u32 *rule_locs` | ✅ |
| `get_rssh` / `set_rssh` 签名 | 多参数 → `struct ethtool_rxfh_param` 统一接口 | ✅ |
| `get_ringparam` / `set_ringparam` | 添加 `kernel_ethtool_ringparam` + `extack` | ✅ |
| `get_coalesce` / `set_coalesce` | 添加 `kernel_ethtool_coalesce` + `extack` | ✅ |
| `get_ts_info` | 使用内核通用 `ethtool_op_get_ts_info` | ✅ |

> **`set_coalesce` 问题根因**：内核 7.0 的 `ethtool_check_ops` 要求提供 `set_coalesce` 时必须同时设置 `supported_coalesce_params` 字段，否则驱动 probe 失败（-EINVAL）。

### C. 编译警告消除（2 类，共 11 处）

| 警告类型 | 文件 | 位置 | 修复方式 |
|----------|------|------|----------|
| `-Wunused-function` | `fm10k_ethtool.c` | `fm10k_set_coalesce` | 启用函数后自动消除 |
| `-Wimplicit-fallthrough` | `fm10k_ethtool.c` | `get_rss_hash_opts` TCP→UDP, UDP→SCTP | `/* fall through */` → `fallthrough;` |
| `-Wimplicit-fallthrough` | `fm10k_main.c` | `tx_csum` GRE→default | 同上 |
| `-Wimplicit-fallthrough` | `fm10k_netdev.c` | `clear_macvlan_queue` MAC→VLAN | 同上 |
| `-Wimplicit-fallthrough` | `fm10k_pf.c` | `iov_supported_xcast_mode_pf` PROMISC→ALLMULTI→MULTI | 同上 |
| `-Wimplicit-fallthrough` | `fm10k_mbx.c` | `mbx_validate_msg_hdr` DISCONNECT→DATA, CONNECT→ERROR | 同上 |
| `-Wimplicit-fallthrough` | `fm10k_mbx.c` | `pfvf_mbx_init` PF→default | 同上 |

---

## 四、验证结果

### 编译
```
0 Error  0 Warning
```
仅两条内核基础设施提醒（编译器/pahole 版本差异），非代码问题。

### ethtool 功能矩阵

| 命令 | 读写 | 结果 |
|------|------|------|
| `ethtool enp1s0` | 读 | ✅ 正常 |
| `ethtool -i enp1s0` | 读 | ✅ 正常 |
| `ethtool -g enp1s0` | 读 | ✅ 正常 |
| `ethtool -G enp1s0 rx 512` | 写 | ✅ 生效 |
| `ethtool -c enp1s0` | 读 | ✅ 正常 |
| `ethtool -C enp1s0 rx-usecs N` | 写 | ✅ 生效 |
| `ethtool -C enp1s0 tx-usecs N` | 写 | ✅ 生效 |
| `ethtool -C enp1s0 adaptive-rx on/off` | 写 | ✅ 生效 |
| `ethtool -k enp1s0` | 读 | ✅ 正常 |
| `ethtool -a enp1s0` | 读 | ✅ 正常 |
| `ethtool -S enp1s0` | 读 | ✅ 正常 |
| `ethtool -d enp1s0` | 读 | ✅ 正常 |
| `ethtool -t enp1s0` | 读 | ✅ PASS |
| `ethtool -T enp1s0` | 读 | ✅ 正常 |
| `ethtool --show-priv-flags enp1s0` | 读 | ✅ 正常 |
| `get_rxfh` / `set_rxfh` | 读写 | ✅ 正常 |
| `get_rxnfc` / `set_rxnfc` | 读写 | ✅ 正常 |

### 系统集成

| 组件 | 状态 |
|------|------|
| fm10k 驱动加载 | ✅ dmesg 无 probe 失败 |
| 网卡 enp1s0 | ✅ 正常创建，MAC `00:e0:ed:78:22:8b` |
| UIO `/dev/uio0` | ✅ 正常，fm10k 依赖 uio |
| RDIF 启动 | ✅ Switch#0 插入，端口 1-6/9 UP |

### 端到端验证流程

```
git checkout -- .          → working tree clean
git apply patch            → 7 files, +67/-61
make -j$(nproc)            → 0 Error, 0 Warning
insmod fm10k.ko            → enp1s0 created, no probe error
ethtool 全功能             → 15 项读写全部通过
rdif start                 → Switch UP, all ports UP
```

---

## 五、补丁文件

| 文件路径 | 大小 |
|----------|------|
| `/root/back1/fm10k-kernel-7.0-full-compat.patch` | 409 行 |
| `/root/back1/CHANGES.md` | 详细修订记录（143 行） |
| `/root/back1/SUMMARY.md` | 本报告 |

### 应用方法

```bash
cd /root/fm10k-rdif-setup
git apply /root/back1/fm10k-kernel-7.0-full-compat.patch
cd src/fm10k/src && make -j$(nproc)
```

---

## 六、修订历史

| 日期 | 修订者 | 内容 |
|------|--------|------|
| 2026-05-22 | AI Agent | 首次修订：内核兼容性适配 + ethtool 全功能恢复 + 0 Error 0 Warning |
