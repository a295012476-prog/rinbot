# gsuid_core 游戏插件集成

## 概述

将 `gsuid_core_app`（鸣潮插件 XutheringWavesUID + 终末地插件 EndUID）通过 WebSocket 桥接到 qq-bot-py，实现单一机器人入口同时支持 AI 聊天和游戏指令。

## 架构

```
NapCat
  └─► qq-bot-py (NoneBot, :8080)
          ├─► gsuid_bridge.py (priority=3) ── 游戏指令 ──► gsuid_core (:8765)
          │                                                  ├─ XutheringWavesUID（鸣潮）
          │                                                  └─ EndUID（终末地）
          ├─► ai_chat.py      (priority=5) ── @机器人 ──► DeepSeek AI
          └─► group_chat.py   (priority=10) ─ 被动回复 ──► DeepSeek AI
```

## 变更文件

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `requirements.txt` | 新增依赖 | 添加 `websockets` |
| `plugins/gsuid_bridge.py` | 新增文件 | WebSocket 桥接插件 |

## 新增依赖

```
websockets
```

安装：
```powershell
pip install websockets
```

## gsuid_bridge.py 说明

### 触发条件

消息文本以以下前缀开头（**不区分大小写**）时触发：

| 前缀 | 对应游戏 |
|------|----------|
| `ww` | 鸣潮（XutheringWavesUID） |
| `end` | 明日方舟终末地（EndUID） |
| `zmd` | 明日方舟终末地（EndUID，备用前缀） |

### 消息路由逻辑

1. `gsuid_bridge`（priority=3）先于 `ai_chat`（priority=5）和 `group_chat`（priority=10）运行
2. 消息文本不含游戏前缀 → 直接返回，交由后续处理器处理（AI 聊天正常工作）
3. 含游戏前缀 → 通过 WebSocket 转发给 `gsuid_core`，等待响应
4. `gsuid_core` 有响应 → 发送到 QQ，并抛出 `StopPropagation` 阻止 AI 插件再次回复
5. `gsuid_core` 无响应（超时 30s）→ 允许后续处理器继续（降级处理）

### 关键配置（在 gsuid_bridge.py 顶部修改）

```python
GSUID_WS_URL = "ws://localhost:8765/ws/Nonebot"  # gsuid_core WebSocket 地址
RESPONSE_TIMEOUT = 30   # 等待首条响应的超时（秒）
DRAIN_TIMEOUT = 2       # 收到首条响应后继续收集后续消息的等待时间（秒）
```

### 连接机制

- 启动时自动建立 WebSocket 长连接
- 断开后每 5 秒自动重连（指数退避升级可按需添加）
- 每条消息使用 UUID 作为 `msg_id`，通过 `_pending` 字典路由响应，避免并发消息混淆

## 启动顺序

必须先启动 `gsuid_core_app`，再启动 `qq-bot-py`：

```powershell
# 终端 1：启动 gsuid_core
cd e:\qqbot\gsuid_core_app
core

# 终端 2：启动 qq-bot-py
cd e:\qqbot\qq-bot-py
python bot.py
```

## 验证

qq-bot-py 启动日志中应出现：
```
[GsuidBridge] 后台 WebSocket 监听器已启动
[GsuidBridge] 已连接至 gsuid_core！
```

在 QQ 群发送以下指令验证：
- `ww帮助` → 应收到鸣潮帮助图片
- `end帮助` 或 `zmd帮助` → 应收到终末地帮助图片
- `@机器人 你好` → 应收到 AI 聊天回复（DeepSeek，不受影响）

更新了sftp插件
