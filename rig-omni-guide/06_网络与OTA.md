# 06 · 网络与 OTA

## WiFi 连接流程

```
上电 → WiFiManager 尝试连接已保存的 WiFi
  ├─ 成功 → 继续启动 → 连接 WebSocket/MQTT → OTA 检查
  └─ 失败 → 进入 BluFi 配网模式
              │
              ├─ 手机 App/小程序通过 BLE 下发 WiFi SSID + 密码
              ├─ 同时在屏幕显示二维码
              ├─ 连接成功 → 保存凭据 → 继续启动
              └─ 超时 → 红灯闪烁 / 等待用户
```

## BluFi 蓝牙配网

ESP32-S3 自带 Bluetooth Classic 和 BLE。配网走 BLE：

```cpp
// boards/common/blufi.cpp
// 手机端扫描到 BLE 广播 → 连接 → GATT Write 下发 SSID + 密码
```

优势：用户不用先手动连一个临时 WiFi，手机 App 直接通过蓝牙发。适合第一次使用。

## 两种通信协议

项目同时支持 MQTT 和 WebSocket，可在固件里选择：

### WebSocket（默认）

MCP 协议就跑在 WebSocket 上：

```cpp
// protocols/websocket_protocol.cc
websocket_->Open("wss://...");
websocket_->SendText(jsonrpc_message);   // MCP 消息就是 JSON-RPC 文本
```

### MQTT

适合资源受限场景或需要消息发布/订阅的架构：

```cpp
// protocols/mqtt_protocol.cc
mqtt_client_->Subscribe("device/response/...");
mqtt_client_->Publish("device/request/...", payload);
```

## OTA 升级

### 分区表

16MB Flash 的分区：

```
┌──────────────────────────────────────┐
│ Bootloader         ~0x1000           │
│ Partition Table    ~0x1000           │
│ App OTA_0          3.5 MB            │  ← 当前运行的固件
│ App OTA_1          3.5 MB            │  ← 备用槽位
│ Resources          4 MB              │  ← SPIFFS（表情、唤醒词、字体、语言包）
│ NVS                24 KB             │  ← 标定数据、WiFi 凭据、设置
└──────────────────────────────────────┘
```

**双 OTA 槽位**的好处：下载到备用槽位，验证通过后才重启切换。如果新固件崩了，还能 rollback 回旧版本。

### OTA 流程

```
Application::Run() 启动时
  │
  ▼
Ota::CheckVersion()
  ├── HTTP GET xl-api.xgorobot.com (国内) / xl-api.luwudynamics.ai (海外)
  │   ?device_id=xxx&version=1.2.3
  └── 返回：有新版本吗？固件 URL？服务器时间？MQTT 地址？WebSocket 地址？
  │
  ▼
有新版本？
  ├─ 否 → 继续运行
  └─ 是 → 调用 MCP 工具 self.upgrade_firmware(url) 或自动升级
             │
             ▼
             esp_ota_begin() → esp_ota_write()（循环写）→ esp_ota_end()
             │
             ▼
             esp_restart() → Bootloader 切换槽位 → 新固件启动
```

### 设备激活（HMAC）

首次启动时需要激活：

```cpp
// Ota::GetActivationPayload()
// 设备发：serial_number + HMAC(challenge, device_secret)
// 服务器验签 → 返回激活通过
```

防止未授权设备使用云端服务。

### MCP 工具触发的 OTA

```cpp
AddUserOnlyTool("self.upgrade_firmware",
    "从指定 URL 升级固件，下载安装后自动重启",
    PropertyList({Property("url", kPropertyTypeString)}),
    [this](...) {
        app.Schedule([url, &app]() {
            app.UpgradeFirmware(url);
        });
        return true;
    });
```

用户在 App 里点"升级" → MCP `tools/call` → 后台下载 → 完成重启。

## 区域化构建（国内 / 海外）

同一个代码库能生成国内版和海外版固件：

```bash
idf.py menuconfig
# → RIG-Omni → Firmware Region → Domestic (China) / Overseas
```

| 配置 | 国内版 | 海外版 |
|------|--------|--------|
| OTA 地址 | `xl-api.xgorobot.com` | `xl-api.luwudynamics.ai` |
| 默认语言 | zh-CN | en-US |
| 唤醒词 | 板型模型（"你好"系列） | 板型模型 + "Hey Kira" |
| 配网表情 | 国内二维码 | 海外二维码 |

## Resources 分区升级

OTA 不只升级 App 固件，还能升级 Resources 分区（SPIFFS）——更新新的表情动画、唤醒词模型、语言包：

```python
# tools/spiffs_assets/build.py — 打资源包
# 工具脚本 generate_bin_package.py — 生成 OTA 用的资源升级包
```

MCP 工具 `self.assets.set_download_url` 可以设置资源包下载地址。

---

**下一篇 → [07_快速上手.md](./07_快速上手.md)**
