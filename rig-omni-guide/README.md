# RIG-Omni 项目深度解读

> 原文项目：[LuwuDynamics/rig_omni](https://github.com/LuwuDynamics/rig_omni) ⭐132 ｜ C++ ｜ 2026-06

一套跑在 ESP32-S3 上的开源机器人固件，**同一套代码库**支持四种机器人形态：

| 形态 | 描述 | 核心文件 |
|------|------|---------|
| **Puppy** | 五舵机四足机器狗 | `main/boards/puppy/` |
| **Hover** | 双轮平衡小车 + 云台模式 | `main/boards/hover/` |
| **ARM** | 五自由度机械臂 | `main/boards/arm/` |
| **TARS** | 三舵机行走机器人 + 终端面板 | `main/boards/tars/` |

---

## 📚 阅读指南

按顺序读，或者跳到感兴趣的部分：

1. **[01_项目总览.md](./01_项目总览.md)** — 5 分钟搞清楚这项目是什么、能干嘛、为什么值得关注
2. **[02_系统架构.md](./02_系统架构.md)** — 从启动到运行的完整架构图，一层层拆解
3. **[03_MCP远程工具架构.md](./03_MCP远程工具架构.md)** — 全文技术重点：机器人本体如何通过 MCP 暴露能力给上位 Agent
4. **[04_运动控制体系.md](./04_运动控制体系.md)** — IMU 姿态 + 舵机驱动 + Hover 平衡控制的底层逻辑
5. **[05_语音交互与EAF动画.md](./05_语音交互与EAF动画.md)** — 从"你好"到表情变化的完整链路
6. **[06_网络与OTA.md](./06_网络与OTA.md)** — BluFi 配网、MQTT/WebSocket、OTA 升级
7. **[07_快速上手.md](./07_快速上手.md)** — 编译、烧录、menuconfig 全流程
8. **[08_硬件清单.md](./08_硬件清单.md)** — 各形态需要什么硬件组件

---

## 🎯 一句话亮点

> **机器人本体通过 MCP 协议暴露能力，上位 AI Agent 直接把机器人当"工具"调用，不用自己造通信协议。**

这个设计把"机器人"变成了 AI 世界里的一个 `tools/call`，是目前把 LLM 和硬件结合最优雅的方式之一。

---

## 🔗 相关链接

- 原版仓库：https://github.com/LuwuDynamics/rig_omni
- MCP 协议规范：https://modelcontextprotocol.io/specification/2024-11-05
- ESP-IDF 框架：https://docs.espressif.com/projects/esp-idf

---

## 📜 许可证

本解读文档采用 CC BY-NC-SA 4.0。原项目采用 Apache License 2.0。
