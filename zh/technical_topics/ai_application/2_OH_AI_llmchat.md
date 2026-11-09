<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @FilePath: \doc\docs-openharmony\zh\k1_oh6.1\7_K1_OH6.1_llmchat.md
 * @Description: 
-->
sidebar_position: 2

# AI聊天应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明 |
|----------|------------|----------|
| 001      | 2026-04-27 | 初始版本 |
| 002      | 2026-11-07 | 对齐最新代码并依据 K3 平台结果校正：新增模型中心，模型改为应用内下载到沙箱 |

## 概述

LLMChat 是基于 OpenHarmony 的本地大语言模型聊天应用。选中模型后，应用拉起设备镜像自带的 `llama-server`，通过 OpenAI 兼容接口在本地完成推理；模型下载完成后即可离线对话。

模型不再需要手动推送到设备：应用内置「模型中心」，联网后可直接下载 Qwen3.5 / Qwen3 系列模型到应用沙箱，支持多模型并存、运行时切换与删除。推理库（`llama-server` 及 `libllama`、`libggml` 等）全部复用设备镜像中的 `/system/bin` 与 `/system/lib64`。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS（TypeScript 超集） |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 构建工具 | Hvigor + DevEco Studio |
| 推理后端 | llama.cpp `llama-server`（镜像自带，版本 b10056） |
| 通信协议 | HTTP / OpenAI 兼容 API（`127.0.0.1:8080`） |
| 模型管理 | 应用内下载（HTTP）、md5 校验、沙箱存储 |
| 权限 | `ohos.permission.INTERNET`、`GET_NETWORK_INFO` |

---

## 项目结构

```
llmchat/
├── AppScope/                        # 应用级资源与配置
├── entry/src/main/ets/
│   ├── entryability/
│   │   └── EntryAbility.ets         # 启动：迁移旧模型、读取选中项、拉起服务
│   ├── pages/
│   │   ├── Index.ets                # 聊天界面
│   │   └── ModelCenter.ets          # 模型中心（下载/使用/删除/启停服务）
│   ├── service/
│   │   ├── LlamaLauncher.ets        # llama-server 启停与健康检查
│   │   ├── LlamaService.ets         # SSE 流式对话
│   │   ├── ModelCatalog.ets         # 内置模型清单
│   │   ├── ModelDownloader.ets      # 流式下载 + md5 校验
│   │   ├── ModelStore.ets           # 沙箱路径、选中项持久化
│   │   ├── ChildProcessGuard.ets    # 子进程生命周期管理
│   │   └── ResourceSafe.ets
│   └── model/
│       ├── ChatMessage.ets
│       └── Conversation.ets
├── build-profile.json5
└── oh-package.json5
```

---

## 架构

```
┌──────────────────────────────────────────────────┐
│                OpenHarmony 设备                   │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │                llmchat HAP                 │  │
│  │  ┌───────────┐      ┌──────────────────┐   │  │
│  │  │Index.ets  │      │ ModelCenter.ets  │   │  │
│  │  │ 聊天 UI   │      │ 模型下载/切换     │   │  │
│  │  └─────┬─────┘      └────────┬─────────┘   │  │
│  │        │                     │             │  │
│  │  ┌─────▼──────┐      ┌───────▼──────────┐  │  │
│  │  │LlamaService│      │ ModelDownloader  │  │  │
│  │  │ HTTP + SSE │      │ ModelStore       │  │  │
│  │  └─────┬──────┘      └───────┬──────────┘  │  │
│  └────────│─────────────────────│─────────────┘  │
│           │ 127.0.0.1:8080      │ 写入沙箱
│           ▼                     ▼                │
│  ┌────────────────────┐  ┌────────────────────┐  │
│  │ llama-server（镜像）│  │ filesDir/models/   │  │
│  │ 子进程，CPU 推理    │  │ <id>.gguf          │  │
│  └────────────────────┘  └────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 流程图

### 应用启动

```
App 启动
   │
   ▼
EntryAbility.onCreate()
   ├─ migrateLegacyModel()：filesDir/model.gguf → models/qwen3-0.6b.gguf
   ├─ initModelStore()：读 filesDir/selection.json 确定选中模型
   └─ 选中模型且文件完整？
        ├─ 否 → 界面提示「请到模型中心下载并选择」
        └─ 是 → LlamaLauncher.startAndWait()
                  ├─ kill 旧进程（按记录的 pid）
                  ├─ spawn llama-server
                  └─ 每 300 ms 轮询 /health，最多 120 s
                        └─ ready → 解锁聊天界面
```

### 模型下载与切换

```
模型中心点「下载」
   │
   ▼
requestInStream → 写入 models/<id>.gguf.part
   ├─ 实时进度/速度（AppStorage 广播，退出页面后继续下载）
   ├─ 完成后按需校验 md5
   └─ rename 为 models/<id>.gguf
         │
         ▼
点「使用」→ 写 selection.json
   ├─ 服务未运行 → 直接拉起
   └─ 服务运行中 → 停止 → 以新模型重启
```

### 消息发送

```
用户发送消息
   │
   ▼
LlamaService.chat()
   ├─ 构造 OpenAI 请求：{ model, messages, stream:true,
   │                      temperature:0.7, max_tokens:2048 }
   └─ @ohos.net.http requestInStream POST /v1/chat/completions
         │
         ▼
按 SSE 解析 data: {...} → 追加 choices[0].delta.content
         │
         ▼
收到 finish_reason / [DONE] → 本轮结束
```

> 首包超时 300 s，流式空闲超时 90 s；不再以整条流总时长作为超时，避免长回答被误判。

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/llmchat.git
cd llmchat
```

---

## 模型获取与管理

模型由应用在设备侧直接下载，无需 PC 下载后再推送。

| 模型 | 参数量 | 大小（约） | md5 |
|------|--------|-----------|-----|
| Qwen3.5-0.8B | 0.8B | 483.7 MB | 有 |
| Qwen3.5-2B | 2B | 1.13 GB | 有 |
| Qwen3.5-4B | 4B | 2.41 GB | 无（界面标注「无校验」） |
| Qwen3-0.6B | 0.6B | 364.5 MB | 有 |

- 下载地址：`http://archive.spacemit.com/spacemit-ai/model_zoo/llm/`（设备不信任该站点证书，统一走 HTTP）。
- 存储路径：`<filesDir>/models/<id>.gguf`，多模型互不覆盖；下载中为 `.part`，校验通过才改名。
- 选中项持久化在 `<filesDir>/selection.json`，应用重启后自动恢复。
- 磁盘需预留「模型大小 + 32 MB」；下载耗时取决于网络，典型速度约 31 MB/s（364.5 MB 约 12 秒）。
- 删除当前选中模型时，自动切换到清单中下一个已下载模型；无可用模型时提示重新下载。

首次使用建议选择 Qwen3.5-0.8B：加载快，日常问答质量与速度较均衡。

---

## 编译构建

### 环境要求

- DevEco Studio 5.x
- OpenHarmony SDK v12
- 设备镜像内置 `llama-server` 及 llama.cpp 动态库

### 构建步骤

1. 用 DevEco Studio 打开项目，等待 Hvigor 同步依赖。
2. 配置签名：`File → Project Structure → Signing Configs`。
3. 构建：

```bash
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

产物：`entry/build/default/outputs/default/entry-default-signed.hap`。

---

## 修改定制指导

### 增加模型

在 `ModelCatalog.ets` 的 `MODEL_CATALOG` 中追加条目，字段包括 `id`、展示名、参数量、下载 URL、精确字节数与 md5 URL。`size` 必须与文件实际大小一致，用于完整性判断与进度计算。

### 调整 llama-server 参数

`LlamaLauncher.ets` 的 `buildCommand()`：

```bash
llama-server -m <model> -t 4 --host 127.0.0.1 --port 8080 \
  --ctx-size 15360 --batch-size 512 --metrics --no-mmap \
  --parallel 1 --reasoning off --reasoning-budget 0
```

- `--ctx-size 15360`：上下文长度，越大越占内存。
- `--parallel 1`：单机对话只需一个 slot，可缩短加载时间。
- `--reasoning off --reasoning-budget 0`：关闭思考段，避免 Qwen3.5 系列把预算耗在 reasoning 上。

### 调整生成参数

`LlamaService.ets` 中修改 `temperature`（默认 0.7）、`max_tokens`（默认 2048）与 `enable_thinking`（默认 false）。

### 添加系统提示词

在 `LlamaService.chat()` 构造 `messages` 时插入 system 角色：

```typescript
const messages = [
  { role: 'system', content: '你是一个有帮助的中文助手。' },
  ...this.conversationHistory
]
```

### 修改端口

同时修改 `LlamaLauncher.ets` 的 `SERVER_PORT` 与 `LlamaService.ets` 的 `serverUrl`。

---

## 设备部署

无需推送模型或推理库，安装 HAP 后首次进入「模型中心」下载模型即可。

```bash
# 安装
hdc uninstall com.example.llmchat
hdc install -r entry-default-signed.hap

# 启动
hdc shell "aa start -b com.example.llmchat -a EntryAbility"

# 服务就绪后验证（ready 后返回 {"status":"ok"}）
hdc shell "curl -s http://127.0.0.1:8080/health"

# 查看日志
hdc shell "hilog | grep -E 'ModelCenter|LlamaService|testTag'"
```

> 设备需能访问 `archive.spacemit.com`；下载状态由 AppStorage 维护，退出模型中心页面不会中断任务。

### 性能参考（K3 Pico，OpenHarmony 6.1.0.35 / API 23）

| 项目 | 结果 |
|------|------|
| 模型下载 | Qwen3-0.6B（364.5 MB），约 12 秒，md5 校验通过并自动选中 |
| 服务启动 | 启动命令按文档执行，7 秒后 `/health` 返回 `{"status":"ok"}` |
| 文本生成 | prompt 185.9 tok/s，生成 37.5 tok/s（20 + 31 token） |
| 模型落盘 | `filesDir/models/qwen3-0.6b.gguf`，大小 382,156,160 字节 |

---

## FAQ

**Q: 启动后提示「尚未选择模型」或「模型文件缺失」？**

进入「模型中心」下载模型并点「使用」。若文件被清理或大小不符，删除后重新下载即可。

**Q: 下载失败或卡住？**

- 确认设备网络可访问 `archive.spacemit.com`（HTTP）。
- 查看 hilog 中 `ModelCenter` 的失败原因（HTTP 状态码、网络错误、空间不足）。
- 下载中可在模型中心取消，取消后会删除 `.part`。

**Q: 切换模型后一直显示「启动中」？**

模型加载时间与大小相关，健康检查最长等待 120 s。查看日志确认 `llama-server` 是否成功 spawn，以及 `/health` 是否可达。

**Q: 响应速度慢？**

- 换用更小的模型（如 Qwen3-0.6B / Qwen3.5-0.8B）。
- 减小 `--ctx-size`，降低内存压力。
- 减少 `max_tokens`，缩短单次生成长度。

**Q: 多轮对话上下文超长？**

`--ctx-size 15360` 是当前默认上限，超出后服务端会截断早期内容。可在 `sendMessage()` 中只保留最近若干条历史。

**Q: 会话数据重启后丢失？**

当前会话仅保存在内存中。如需持久化，可在 `newConversation` / `sendMessage` 后序列化写入 `filesDir`，启动时读取恢复。

**Q: 模型能否离线使用？**

可以。模型下载到沙箱后，推理完全本地执行；断网只影响下载新模型。

**Q: 能否与 vlmdemo 共用模型？**

不需要。两个应用各自在沙箱内管理模型，互不影响；llmchat 下载的是纯文本 GGUF，vlmdemo 下载的是多模态模型包。
