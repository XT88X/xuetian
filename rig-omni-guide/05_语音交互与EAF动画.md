# 05 · 语音交互与 EAF 动画

## 一条语音从输入到输出的完整链路

```
用户说话
  │
  ▼
I2S 硬件 → 麦克风采集原始 PCM 数据
  │
  ▼
AudioService (音频处理前端)
  ├── AFE AEC（回声消除，因为机器人自己也在说话）
  ├── AGC（自动增益控制）
  └── Noise Suppression（降噪）
  │
  ▼
Wake Word Detection
  ├── ESP-SR AFE（芯片内置）—— 本地离线跑
  └── 自定义模型（可打包进 SPIFFS）
  │ （被唤醒！）
  ▼
VAD（语音活动检测）—— 判断用户说完了
  │
  ▼
Opus 编码 → 打包 → 通过 WebSocket 发给云端
  │
  ▼
云端 ASR（语音识别）→ TTS（语音合成）
  │
  ▼
Opus 解码 → I2S → 扬声器
  │
  ▼
说话同时：EAF 表情动画跟着变（listen → thinking → speaking）
```

## 音频硬件

### 方案一：专用编解码芯片

```
I2S ──▶ ES8311 / ES8374 / ES8388 / ES8389
        ↑ 支持多路麦克风输入
```

这些是常见的音频 Codec 芯片，支持硬件 DSP。

### 方案二：直连（NoAudioCodec）

Puppy 和 Hover 用的就是方案二——麦克风和扬声器直接连 ESP32-S3 的 I2S 引脚，没有额外编解码芯片：

```cpp
// 双工模式
static NoAudioCodecDuplex audio_codec(
    AUDIO_INPUT_SAMPLE_RATE,   // 16kHz
    AUDIO_OUTPUT_SAMPLE_RATE,  // 16kHz
    AUDIO_I2S_GPIO_BCLK, AUDIO_I2S_GPIO_WS,
    AUDIO_I2S_GPIO_DOUT, AUDIO_I2S_GPIO_DIN
);
```

ESP32-S3 内部有足够的硬件支持，直连能省成本。

## 音频编解码

### Opus 编码

云端传数据用 Opus 格式——低延迟、高质量、带宽占用小：

```
原始 16kHz PCM（16bit × 2 = 32kbps × 16k = 512 kbps）
          ↓ Opus 编码
Opus 压缩流（典型 16-64 kbps）
```

### OGG 封装

本地资源（提示音、语音包）用 OGG 格式封装 Opus：

```cpp
// 播放"你好"提示音的代码路径
Application::GetInstance().PlaySound(Lang::Sounds::OGG_WIFI_SUCCESS());
// ↑ Lang::Sounds::OGG_WIFI_SUCCESS() 就是内嵌资源的内存地址范围
```

`.ogg` 文件被 CMake 编译进固件，以 `_binary_xxx_start` / `_binary_xxx_end` 符号暴露。

## 唤醒词检测

### 三种实现（可切换）

| 实现 | 位置 | 说明 |
|------|------|------|
| **ESP-SR AFE** | ESP-IDF 内置 | 芯片加速，低功耗，支持"你好"等固定词 |
| **AFE Wake Word** | 自研 / ESP-SR | 更灵活，可换模型 |
| **Custom Wake Word** | SPIFFS | 可以自定义唤醒词，模型打包进资源镜像 |

### 模型打包

唤醒词模型不是编译时固定的，而是打包进 **SPIFFS 资源分区**：

```
main/boards/puppy/wakenet/wn9_xiaolutongxue/
├── _MODEL_INFO_
├── wn9_data
└── wn9_index
```

构建时用 `python3 tools/spiffs_assets/build.py` 把所有表情、字体、唤醒模型、语言配置打包成一个 SPIFFS 镜像，烧录到 Flash 的 resources 分区。

## 语音活动检测（VAD）

VAD 的作用：**什么时候用户说完了？**

机器人被唤醒后持续监听，等用户说完话后自动停止采集、打包发送。VAD 检测到一段静音（默认几百毫秒）就触发 `MAIN_EVENT_VAD_CHANGE`。

## EAF 表情动画引擎

### 什么是 EAF？

EAF（Emote Animation Format）是这套项目自己定义的表情动画格式，放在 `main/boards/<board>/emoji/*.eaf` 文件里。每个 `.eaf` 文件是一段动画序列，机器人圆屏上的"脸"就是播放这些动画。

### 有哪些表情？

每个板型的 emoji 目录下有 30+ 个 `.eaf`：

| 表情名 | 场景 |
|--------|------|
| `neutral` | 平时待机 |
| `happy` / `sad` / `angry` | AI 生成内容的情感匹配 |
| `listen` | 正在听用户说话 |
| `thinking` | 等 AI 回复 |
| `calibration` | 标定模式 |
| `wificonfig` | 配网（展示二维码） |
| `launch` | 开机 |
| `remote_mode` | 蓝牙遥控模式 |
| `sleepy` | 困倦 |
| `cool` | 自信 |
| ... | ... |

### EmoteDisplay 怎么驱动？

```cpp
// 构造函数里初始化
display_ = new emote::EmoteDisplay(panel, panel_io, DISPLAY_WIDTH, DISPLAY_HEIGHT);

// 切表情
display_->SetEmotion("happy");

// 插播一段临时动画（对话表情）
display_->InsertAnimDialog("hi", 3000);  // 插入 "hi" 动画，持续 3 秒
```

EmoteDisplay 继承自抽象的 `Display` 基类，所有显示操作（亮度、主题、截图、预览图）都走这个基类——MCP 工具 `self.screen.*` 调的就是这些方法。

### 对话中的表情联动

状态机里，每次设备状态变化时（Starting → Listening → Speaking → Idle），都会触发表情切换：

```cpp
void OnStateChanged(DeviceState old_state, DeviceState new_state) {
    switch (new_state) {
        case kDeviceStateListening: display_->SetEmotion("listen"); break;
        case kDeviceStateSpeaking:  display_->SetEmotion("thinking"); break;
        case kDeviceStateIdle:      display_->SetEmotion("neutral"); break;
    }
}
```

## TARS 特殊显示

TARS 是行走机器人，不用圆屏表情——它用 320×240 横屏做**终端面板**，致敬《星际穿越》的 TARS：

```
┌──────────────────────┬─────────────────┐
│  蓝色日志 (6px)      │  绿色状态 (8px)  │
│                      │                 │
│  [大量滚动日志行]    │  对话状态       │
│                      │  舵机角度       │
│                      │  IMU 姿态       │
│                      │  电量           │
├──────────────────────┴─────────────────┤
│   HONESTY 90%    HUMOR 75%             │  ← 底部状态条
└─────────────────────────────────────────┘
```

底部的 HONESTY 和 HUMOR 是"人格参数"，可以通过 MCP 工具 `self.tars.set_personality` 修改。

## 声音资源管理

### 语音包结构

```
main/assets/locales/
├── zh-CN/
│   ├── activation.ogg      ← 激活成功
│   ├── calibration_enter.ogg
│   ├── wifi_success.ogg
│   ├── welcome.ogg
│   └── language.json        ← 语音包配置
└── en-US/                   ← 英文版
    └── ...
```

### 板级音效

每个板型还可以有专属音效：

| 板型 | 专属音效 |
|------|---------|
| Puppy | `woof.ogg`（汪汪叫） |
| Hover | `engine_startup.ogg`（引擎启动声）、`engine_throttle.ogg`（油门声） |
| TARS | 整套 `tars_*.ogg` 系列（不同音色的英文提示音） |

### 用火山引擎生成 TARS 语音包

```bash
export VOLC_TTS_SPEAKER='your-speaker-id'
python3 tools/generate_tars_voice_pack.py --only message_send,over --overwrite
# 自动生成 → FFmpeg 转采样率 → 保存为 OGG
```

---

**下一篇 → [06_网络与OTA.md](./06_网络与OTA.md)**
