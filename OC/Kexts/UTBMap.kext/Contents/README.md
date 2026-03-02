# UTBMap.kext Info.plist 变体说明

## 文件列表

| 文件 | 状态 | 说明 |
|------|------|------|
| `Info.plist` | **活跃** | 当前加载的配置，Tahoe 兼容（含 `port-number` / `usb-connector-type`） |
| `pre-tahoe-Info.plist` | 备份 | Tahoe 之前的版本，仅含旧属性（`port` / `UsbConnector`） |
| `0Info.plist` | 变体 | 端口精简方案 0 — 禁用部分端口以绕 macOS 15 端口限制 |
| `1Info.plist` | 变体 | 端口精简方案 1 — 禁用不同的端口组合 |
| `allInfo.plist` | 变体 | 全端口开启 — 调试/测试用 |

> macOS 只读取 `Info.plist`。切换方案：将目标文件重命名为 `Info.plist` 即可。

## USB 控制器

| 控制器 | pcidebug | 说明 | 包含该控制器的文件 |
|--------|----------|------|--------------------|
| `3:0:2` | 3:0:2 | 附加 USB 控制器（3 端口） | Info.plist, pre-tahoe-Info.plist |
| `_SB.PC00.RP05.PXSX.TBDU.XHCI` | 14:0:0 | Thunderbolt 3 USB 控制器 | 全部 |
| `_SB.PC00.XHCI` | 0:20:0 | 主板 XHCI 控制器（主控） | 全部 |

## Tahoe 兼容机制

macOS Tahoe 更改了 USB 驱动读取端口信息的属性名。当前 `Info.plist` 同时包含新旧属性以兼容所有版本：

| 旧属性（pre-Tahoe） | 新属性（Tahoe） | 关系 |
|---------------------|----------------|------|
| `port`（data） | `port-number`（data） | 值相同 |
| `UsbConnector`（integer） | `usb-connector-type`（integer） | 值相同 |

## 端口禁用对比（`_SB.PC00.XHCI` 主控）

`#` 前缀表示该端口被禁用（同时 `port` 键也改为 `#port`）。

| 端口 | Info.plist | 0Info | 1Info | allInfo | 连接器类型 |
|------|-----------|-------|-------|---------|-----------|
| HS01 | ✅ | ✅ | ✅ | ✅ | 9 (TypeC+Switch) |
| HS02 | ✅ | ✅ | ✅ | ✅ | 255 (内部) |
| HS03 | ✅ | ✅ | ✅ | ✅ | 9 (TypeC+Switch) |
| HS04 | ✅ | ✅ | ✅ | ✅ | 255 (内部) |
| HS05 | ✅ | ✅ (type=0) | ✅ (type=0) | ✅ (type=3) | 3 (USB-C) |
| HS06 | ✅ | ✅ (type=0) | ✅ (type=0) | ✅ (type=3) | 3 (USB-C) |
| HS07 | ✅ | **#禁用** | **#禁用** | ✅ | 3 (USB-C) |
| HS08 | ✅ | **#禁用** | **#禁用** | ✅ | 3 (USB-C) |
| HS09 | ✅ | ✅ (type=0) | **#禁用** (type=2) | ✅ (type=3) | 3 (USB-C) |
| HS10 | ✅ | **#禁用** | ✅ (type=255) | ✅ (type=3) | 3 (USB-C) |
| HS11 | ✅ | ✅ | ✅ | ✅ | 9 (TypeC+Switch) |
| HS12 | ✅ | **#禁用** | **#禁用** | ✅ | 9 (TypeC+Switch) |
| HS13 | ✅ (type=0) | ✅ (type=255) | ✅ (type=255) | ✅ (type=255) | 0 (USB-A 2.0) |
| HS14 | ✅ | — | — | — | 255 (内部) |
| HS15 | ✅ | — | — | — | 0 (USB-A 2.0) |
| HS16 | ✅ | — | — | — | 0 (USB-A 2.0) |
| SS01–SS08 | ✅ 全部 | SS06 禁用 | SS06 禁用 | ✅ 全部 | 混合 |

## UsbConnector 类型参考

| 值 | 含义 |
|----|------|
| 0 | USB Type-A (USB 2.0) |
| 2 | USB Type-A (USB 3.0) |
| 3 | USB Type-C (不带方向切换) |
| 9 | USB Type-C (带方向切换) |
| 255 | 内部设备 |
