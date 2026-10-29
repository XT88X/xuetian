# 03 · MCP 远程工具架构

> 这是整个项目中最有参考价值的部分。如果你只看一篇，就看这篇。

## 为什么 MCP 值得关注？

传统方式做 AI + 硬件：你要自己发明一套通信协议（比如 `{"cmd": "move_forward", "dist": 5}`），自己定义字段、自己解析、自己处理错误、自己做版本兼容。每次加个新功能，协议又要改。

**MCP（Model Context Protocol）** 的思路是：既然大家都想让 Agent 调用工具，那不如定个通用协议。Agent 按协议发 `tools/list` 拿到所有可用工具的名称、描述、参数 Schema，然后按协议发 `tools/call` 调用。硬件端只要按协议响应就行。

就像 HTTP 之于 Web 应用——你不用关心对方是 Python 还是 Java，只要说 HTTP 就能通信。MCP 之于 AI Agent 也是同理。

## rig_omni 的 MCP 实现

### 实现的协议版本

参考 [modelcontextprotocol.io/specification/2024-11-05](https://modelcontextprotocol.io/specification/2024-11-05)，只实现了设备端需要的最小子集：

| 客户端（Agent）发来的方法 | 设备端（RIG-Omni）处理 |
|--------------------------|----------------------|
| `initialize` | 返回 `protocolVersion`、`capabilities`、`serverInfo`（名字 + 固件版本） |
| `tools/list` | 返回所有注册工具的列表（支持分页 cursor） |
| `tools/call` | 执行工具回调，返回结果文本或图片 |

用的是 **JSON-RPC 2.0** 作为消息格式。

### 传输层

MCP 本身不规定怎么传。RIG-Omni 用的是 **WebSocket**（也支持 MQTT）：

```cpp
// mcp_server.cc 里的消息入口
void McpServer::ParseMessage(const std::string& message) {
    cJSON* json = cJSON_Parse(message.c_str());
    // 检查 jsonrpc == "2.0"，检查 method，分发处理
}
```

WebSocket 收消息 → `Application` 层 → `McpServer::ParseMessage` → 分发到对应 handler。

## 工具注册的三种方式

### 1. AddTool（AI 和用户都能看到）

```cpp
mcp_server.AddTool(
    "self.audio_speaker.set_volume",                          // 工具名（点分层级）
    "Set the volume of the audio speaker.",                    // AI 能读懂的描述
    PropertyList({
        Property("volume", kPropertyTypeInteger, 0, 100)       // 参数 + 范围
    }),
    [&board](const PropertyList& props) -> ReturnValue {      // 回调
        auto codec = board.GetAudioCodec();
        codec->SetOutputVolume(props["volume"].value<int>());
        return true;
    }
);
```

### 2. AddUserOnlyTool（只有用户能看到，AI 看不到）

加了 `annotations.audience: ["user"]` 标注。适合放调试工具，比如 `self.reboot`、`self.upgrade_firmware`——不想让 AI 自动重启机器人对吧？

### 3. 板级工具（Board::InitializeTools 里注册）

每个形态在自己的 `InitializeTools()` 里注册专属工具。Puppy 注册 dog 系列，Hover 注册 robot 系列。这是"多形态共用框架"的关键。

## 参数系统

参数类型只支持三种：`boolean`、`integer`、`string`。整数参数可以加范围：

```cpp
Property("distance", kPropertyTypeInteger, -20, 20)     // 必填，范围 -20~20
Property("time",     kPropertyTypeInteger, 0, 10000)     // 必填，范围 0~10000
Property("volume",   kPropertyTypeInteger, 50, 0, 100)   // 有默认值 50，范围 0~100
```

参数类封装了 JSON Schema 生成、默认值填充、范围校验。

## 工具执行流程

```
Agent 发 {"jsonrpc":"2.0", "id":42, "method":"tools/call",
          "params":{"name":"self.dog.Wave","arguments":{}}}
        │
        ▼
McpServer::DoToolCall(id, "self.dog.Wave", arguments)
        │
        ├─ 找到这个工具（线性搜索 tools_ 数组）
        ├─ 把 JSON arguments 合并到工具的 PropertyList
        │
        ▼
app.Schedule([...](){
    // ★ 关键：排队到主任务执行
    // 这样回调里可以安全操作 LVGL、播放音效
    ReplyResult(id, (*tool_iter)->Call(arguments));
})
        │
        ▼
Agent 收到 {"jsonrpc":"2.0", "id":42, "result":{"content":[...],"isError":false}}
```

### 为什么要 Schedule 到主任务？

回顾 [02_系统架构.md](./02_系统架构.md) 的线程模型：MCP 消息可能来自 WebSocket 任务，而 LVGL 显示、`Application::PlaySound` 等只能在主任务里操作。如果直接在 WebSocket 回调里调，很容易引发竞态或崩溃。

## 返回值类型

工具回调返回 `std::variant<bool, int, std::string, cJSON*, ImageContent*>`：

| 返回类型 | 例子 | MCP 响应 |
|---------|------|---------|
| `bool` | `return true;` | `content:[{"type":"text","text":"true"}]` |
| `int` | `return 42;` | `content:[{"type":"text","text":"42"}]` |
| `string` | `return "已切换云台模式";` | `content:[{"type":"text","text":"已切换云台模式"}]` |
| `cJSON*` | `return board.GetDeviceStatusJson();` | JSON 对象序列化成文本 |
| `ImageContent*` | 摄像头工具返回图片 | `content:[{"type":"image","image":{...base64...}}]` |

## 实际可用工具一览

### 通用工具（所有形态都有）

| 工具名 | 功能 |
|--------|------|
| `self.get_device_status` | 返回音频、屏幕、电池、网络等实时状态（JSON） |
| `self.audio_speaker.set_volume` | 设置扬声器音量（0-100） |
| `self.screen.set_brightness` | 设置屏幕亮度（0-100） |
| `self.screen.set_theme` | 切换 light/dark 主题 |
| `self.camera.take_photo` | 拍照 + AI 分析（需要摄像头） |
| `self.camera.preview` | 拍照只显示，不分析 |
| `self.set_power_mode` | 切换 performance/balanced/low_power |
| `self.power_off` | 3 秒后进入深度睡眠（让告别语音播完） |
| `self.set_language` | zh-CN / en-US |

### Puppy 专属

| 工具名 | 功能 |
|--------|------|
| `self.dog.move` | 前后移动 + 转向 |
| `self.dog.set_motor_angle` | 即兴设定 5 个舵机角度 |
| `self.dog.calibrate` | 进入/退出标定模式 |
| `self.dog.Wave / Naughty / Swing / Sit ...` | 20+ 预设动作 |
| `self.dog.action_loop` | 开启/关闭表演模式（动作循环） |
| `self.laser.control` | 激光剑开关 |
| `self.ble.remote_control` | 开启蓝牙遥控 |

### Hover 专属

| 工具名 | 功能 |
|--------|------|
| `self.robot.move` | 前进后退（厘米） |
| `self.robot.rotate` | 原地旋转（度） |
| `self.robot.head_angle` | 头部角度（±75°） |
| `self.robot.set_mode` | 平衡小车 ↔ 云台模式 |
| `self.robot.pitch` | 云台俯仰 |
| `self.status.battery` | 查电量 |

## 几个精巧的设计

### 1. prompt cache 友好排序

```cpp
void McpServer::AddCommonTools() {
    // Important: To speed up the response time, we add the common tools
    // to the beginning of the tools list to utilize the prompt cache.
    auto original_tools = std::move(tools_);
    // ... 先加通用工具到前面
    tools_.insert(tools_.end(), original_tools.begin(), original_tools.end());
}
```

LLM 有 prompt cache，把高频用的工具放前面能让缓存命中率更高。

### 2. tools/list 分页 + 大小限制

嵌入式 Flash 有限，`tools/list` 返回的 JSON 不能太大。代码里设了 `max_payload_size = 8000` 字节，超了就用 `nextCursor` 分页：

```cpp
if (json.length() + tool_json.length() + 30 > max_payload_size) {
    next_cursor = (*it)->name();  // 下次从这里继续
    break;
}
```

### 3. tools/call 错误隔离

每个工具回调都包在 try-catch 里。如果某个工具抛异常（比如摄像头没初始化好），McpServer 返回 `{"isError": true, "error":{"message":"..."}}` 而不崩溃。

### 4. user_only 工具隔离

user_only 工具加了 `annotations.audience: ["user"]`，Agent 调 `tools/list` 时默认看不到。只有显式传 `withUserTools: true` 才列出来。防止 AI 误操作危险工具。

### 5. deep sleep 延迟执行

```cpp
AddTool("self.power_off", ..., [&board](...) {
    esp_timer_start_once(power_off_timer, 3 * 1000000ULL);  // 3 秒后睡
    return std::string("Powering off in 3 seconds");
});
```

让告别语音先播完再关机。细节见功力。

## 给你的工程启示

### 什么时候用 MCP？

当你有多个设备/应用需要被 AI Agent 调用时：
- 智能家具（灯、空调、窗帘都注册成 MCP 工具）
- 机器人（如本项目）
- 内部开发工具链（构建、部署、查询状态）

### 怎么在 ESP32 上实现 MCP？

最小需要：
1. **JSON-RPC 2.0 解析器**（用 cJSON，本项目用的）
2. **工具注册表**（`std::vector<McpTool*>`）
3. **一个 ParseMessage 入口**，分发到 initialize/tools/list/tools/call
4. **一个传输层**（WebSocket/MQTT/UART 都行）
5. **主线程调度机制**（回调里操作 UI 必须 Schedule 回去）

### 可以进一步增强的地方

- 加服务端推送 notifications（比如电池电压变化主动通知）
- 实现 `resources/list` 和 `resources/read`（Agent 读传感器历史数据）
- 实现 `prompts/list` 和 `prompts/get`（预设好的对话模板）
- 加 Token 认证或设备密钥验证

---

**下一篇 → [04_运动控制体系.md](./04_运动控制体系.md)**
