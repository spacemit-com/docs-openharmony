<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-04-27 15:51:15
 * @FilePath: \doc\docs-openharmony\zh\k1_oh6.1\7_K1_OH6.1_llmchat.md
 * @Description: 
-->
sidebar_position: 2

# AI聊天应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明       |
|----------|------------|-------------|
| 001      | 2026-04-27 | 初始版本      |

## 概述

LLMChat是基于 OpenHarmony 的本地大语言模型聊天应用，在设备端直接运行 llama-server，无需联网即可与 AI 对话。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS（TypeScript 超集，OpenHarmony 专用） |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 构建工具 | Hvigor + DevEco Studio |
| 推理后端 | llama.cpp llama-server（设备本地进程） |
| 通信协议 | HTTP / OpenAI 兼容 API（本地 127.0.0.1:8080） |
| 测试框架 | @ohos/hypium + @ohos/hamock |

---

## 项目结构

```
llmchat/
├── AppScope/                        # 应用级资源与配置
│   ├── app.json5                    # 包名、版本号
│   └── resources/                   # 全局资源（图标等）
├── entry/                           # 主模块（HAP）
│   ├── src/main/ets/
│   │   ├── entryability/
│   │   │   └── EntryAbility.ets     # 应用入口，管理 llama-server 进程
│   │   ├── pages/
│   │   │   └── Index.ets            # 主界面（聊天 UI）
│   │   ├── service/
│   │   │   └── LlamaService.ets     # 与 llama-server 通信的服务层
│   │   └── model/
│   │       ├── ChatMessage.ets      # 消息数据模型
│   │       └── Conversation.ets     # 会话数据模型
│   ├── src/main/resources/          # 多语言资源、颜色、图片
│   ├── src/test/                    # 单元测试
│   └── src/ohosTest/                # 设备测试
├── hvigor/                          # 构建系统配置
├── build-profile.json5              # 构建配置（签名、SDK 版本）
├── oh-package.json5                 # 依赖声明
└── code-linter.json5                # 代码规范配置
```

---

## 架构

```
┌──────────────────────────────────────────────────┐
│                  OpenHarmony 设备                 │
│                                                  │
│  ┌───────────────────────────────────────────┐   │
│  │               llmchat HAP                 │   │
│  │                                           │   │
│  │  ┌─────────────┐     ┌─────────────────┐  │   │
│  │  │ EntryAbility │     │   Index.ets     │  │   │
│  │  │  (生命周期)  │     │   (聊天 UI)    │  │   │
│  │  └──────┬───────┘     └────────┬────────┘  │   │
│  │         │ 启动/停止进程         │ 调用      │   │
│  │         │                      ▼           │   │
│  │         │          ┌───────────────────┐   │   │
│  │         │          │   LlamaService    │   │   │
│  │         │          │ (HTTP + 流式解析) │   │   │
│  │         │          └─────────┬─────────┘   │   │
│  └─────────│────────────────────│─────────────┘   │
│            │                    │ curl / HTTP      │
│            ▼                    ▼                  │
│  ┌──────────────────────────────────────────────┐  │
│  │         llama-server（本地子进程）            │  │
│  │         127.0.0.1:8080                       │  │
│  │         模型文件: /etc/xxx.gguf              │  │
│  └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 流程图

### 应用启动流程

```
App 启动
   │
   ▼
EntryAbility.onCreate()
   │
   └─ setTimeout 100ms（异步，不阻塞主线程）
         │
         ▼
   pkill -9 llama-server（清理旧进程）
         │
         └─ 等待 500ms
               │
               ▼
         launchLlamaServer()
         llama-server -m /etc/xxx.gguf \
           -t 4 --host 127.0.0.1 --port 8080 \
           --ctx-size 15360 --no-mmap ...
               │
               ▼
   onWindowStageCreate() → 加载 pages/Index
               │
               ▼
   Index.aboutToAppear()
         ├─ newConversation()（创建初始会话）
         ├─ startDotAnimation()（加载动画）
         └─ setTimeout 6s → pollHealth()
                                 │
                          ┌──────┴──────┐
                          ▼             ▼
                       ready         loading
                          │             │
                     停止轮询        2s 后重试
                     显示就绪提示
```

### 消息发送流程

```
用户输入消息 → 点击「发送」
   │
   ▼
sendMessage()
   ├─ 创建 ChatMessage(USER)，追加到当前会话
   ├─ 首条消息自动截取前 20 字作为会话标题
   ├─ 创建空 ChatMessage(ASSISTANT) 占位
   └─ llamaService.setHistory(历史消息)
         │
         ▼
LlamaService.chat()
   ├─ 构造 OpenAI 格式 JSON body
   │   { model, messages, stream:true, temperature:0.7, max_tokens:2048 }
   ├─ 删除旧的 llm_stream.txt
   └─ process.runCmd(curl POST /v1/chat/completions -o llm_stream.txt)
         │
         ▼
readStream()（轮询文件）
   ├─ 等待文件出现（每 200ms，最多 10s）
   ├─ 每 50ms 读取文件新增内容
   ├─ 按行解析 SSE 格式（data: {...}）
   ├─ 提取 choices[0].delta.content
   └─ 逐字符压入 charQueue
         │
         ▼
字符队列输出定时器（每 30ms 出队一个字符）
   │
   ▼
onChunk(char) → ASSISTANT 消息内容追加
   │
   ▼
收到 data: [DONE] → 流结束 → isLoading = false
```

### 健康检查流程

```
pollHealth()（Index 初始化后 6s 触发）
   │
   ▼
LlamaService.checkHealth()
   ├─ curl GET /health -m 3 -o health_resp.txt
   └─ 轮询文件（每 200ms，超时 5s）
         │
         ├─ 含 "ok"      → resolve('ready')
         ├─ 含 "loading" → resolve('loading')
         └─ 超时         → resolve('unreachable')
               │
   ┌───────────┴────────────┐
   ▼                        ▼
'ready'                 'loading' / 'unreachable'
   │                        │
modelStatus = 'ready'   modelStatus = 'loading'
停止动画，显示就绪提示    2s 后重新调用 pollHealth()
```

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/llmchat.git
cd llmchat
```

---

## 模型文件获取

llmchat 使用 GGUF 格式的量化大语言模型，推荐 Q4_K_M 或 Q4_1 量化（兼顾速度与质量）。

- 可从 [Hugging Face](https://huggingface.co/models?library=gguf) 下载社区提供的 GGUF 模型
- 也可与 vlmdemo 共用 FastVLM 文本解码器 `fastvlm-text-0.5B-Q4_1.gguf`（位于代码仓库 `fastvlm-mm-0.5b-q4_1/` 目录，约 876 MB）
- 或使用 llama.cpp 自带的 `llama-quantize` 工具，将 FP16 模型量化为 GGUF 格式

模型文件推送到设备的方法见下文「设备部署 → 部署模型文件」。

---

## 编译构建

### 环境要求

- **DevEco Studio** 5.x（推荐最新版）
- **OpenHarmony SDK v12**
- 已连接的 OpenHarmony 设备或模拟器
- 设备上 `llama-server` 可执行文件已在 PATH 中
- 模型文件已放置于设备 `/etc/model.gguf`（路径可配置）

### 构建步骤

1. 用 DevEco Studio 打开项目根目录
2. 等待 Hvigor 自动同步依赖（`oh-package.json5`）
3. 配置签名：编辑 `build-profile.json5` → `signingConfigs`
4. 选择目标设备，点击 **Run**

命令行构建：

```bash
./hvigorw assembleHap --mode module -p module=entry@default
```

### 构建产物

```
entry/build/default/outputs/default/
├── entry-default-unsigned.hap   # Debug（未签名）
└── entry-default-signed.hap     # Release（需配置签名）
```

### 代码检查

```bash
./hvigorw lint --mode module -p module=entry@default
```

### 运行测试

```bash
# 设备单元测试
./hvigorw test --mode module -p module=entry@ohosTest
```

---

## 修改定制指导

### 更换模型文件

需同步修改两处：

`EntryAbility.ets:32` — llama-server 启动路径：
```typescript
const modelPath = '/etc/model.gguf';  // 改为实际路径
```

`LlamaService.ets:30` — 请求 body 中的 model 字段：
```typescript
private modelPath: string = '/etc/model.gguf'  // 改为实际路径
```

### 调整推理参数

`EntryAbility.ets:42` 启动命令各参数说明：

```bash
llama-server -m <model>
  -t 4              # CPU 线程数，建议 = 设备物理核心数
  --ctx-size 15360  # 上下文长度（token），越大越占内存
  --n-gpu-layers 0  # GPU 卸载层数，0 = 纯 CPU
  --batch-size 512  # 批处理大小，影响首 token 延迟
  --no-mmap         # 禁用内存映射，嵌入式设备推荐开启
  --metrics         # 暴露 /metrics 端点（可选）
```

### 调整生成参数

`LlamaService.ets:53`：

```typescript
const body = JSON.stringify({
  temperature: 0.7,   // 创造性，0.0（确定）~ 1.0（随机）
  max_tokens: 2048,   // 单次最大生成 token 数
  stream: true,       // 保持 true 以启用流式输出
})
```

### 添加系统提示词

在 `LlamaService.chat()` 构造 messages 时插入 system 角色：

```typescript
const messages = [
  { role: 'system', content: '你是一个有帮助的中文助手。' },
  ...this.conversationHistory
]
```

### 修改打字机速度

`LlamaService.ets:89` — 调小数值更快，调大更慢：

```typescript
outputTimer = setInterval(() => { ... }, 30)  // 单位 ms/字
```

### 修改服务器地址或端口

`LlamaService.ets:29`：

```typescript
private serverUrl: string = 'http://127.0.0.1:8080'
```

同步修改 `EntryAbility.ets:42` 中 `--port` 参数。

### 修改 UI 主题色

| 元素 | 当前颜色 | 位置 |
|------|----------|------|
| 发送按钮 / 就绪色 | `#07C160`（微信绿） | `Index.ets:198` |
| 用户消息气泡 | `#95EC69` | `Index.ets:33` |
| 消息列表背景 | `#EDEDED` | `Index.ets:179` |
| 顶栏 / 输入栏背景 | `#F7F7F7` | `Index.ets:165` |

### 替换头像图片

替换 `entry/src/main/resources/base/media/` 下的图片文件：

- `user_avatar.png` — AI 助手头像（左侧气泡）
- `user_avatar1.png` — 用户头像（右侧气泡）

---

## 设备部署

目标设备：搭载 SpaceMIT K3 SoC 的 RISC-V 开发板（如 K3 Pico ITX，X100 为 CPU 核心型号）。

> 以下步骤均在 **SpaceMIT K3 Pico ITX** 真机（riscv64、内核 6.18.3、16 核、16 GB 内存）实测验证。

### 1. 部署动态库与可执行文件

以下文件来自编译产物 `device/soc/spacemit/common/hardware/`，需推送到设备系统分区。

```bash
# 挂载系统分区为可写
hdc shell "mount -o rw,remount /"

# llama.cpp 动态库（带版本 SONAME）
hdc file send libllama.so.0            /system/lib64/libllama.so.0
hdc file send libggml.so.0             /system/lib64/libggml.so.0
hdc file send libggml-base.so.0        /system/lib64/libggml-base.so.0
hdc file send libggml-cpu.so.0         /system/lib64/libggml-cpu.so.0
hdc file send libllama-common.so.0     /system/lib64/libllama-common.so.0
hdc file send libllama-server-impl.so  /system/lib64/libllama-server-impl.so

# llama-server 启动器（依赖 libllama-server-impl.so）
hdc file send llama-server /system/bin/llama-server
hdc shell "chmod 755 /system/bin/llama-server"
```

> 实测说明：K3 Pico ITX 镜像已预置上述库（`ls /system/lib64/libggml*.so.0 /system/lib64/libllama*.so.0` 可见）与 `llama-server`。若设备已烧录完整镜像，**可直接跳到步骤 2**，仅需推送模型文件。

### 2. 部署模型文件

```bash
# 将量化模型推送到设备（路径与代码中的 modelPath 保持一致）
hdc shell "mkdir -p /etc"
hdc file send <model>.gguf /etc/model.gguf
```

若同时部署 vlmdemo，可与其共用同一模型：先推送到 `/etc/vlm/model.gguf`，再创建硬链接（实测后 link count = 2）：

```bash
hdc file send fastvlm-text-0.5B-Q4_1.gguf /etc/vlm/model.gguf
hdc shell "ln -f /etc/vlm/model.gguf /etc/model.gguf"
```

### 3. 编译安装 HAP

```bash
# 构建（在 llmchat 项目根目录执行）
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon

# 安装到设备（需先卸载旧版本）
hdc uninstall com.example.llmchat
hdc install -r entry/build/default/outputs/default/entry-default-signed.hap
```

### 4. 启动验证

```bash
# 启动应用（llama-server 由应用自动拉起）
hdc shell "aa start -b com.example.llmchat -a EntryAbility"

# 等待约 10-30 秒模型加载完成，确认服务就绪
hdc shell "curl -s http://127.0.0.1:8080/health"
# 预期返回：{"status":"ok"}

# 查看应用日志
hdc shell "hilog | grep -E 'testTag|LlamaService'"
```

---

## FAQ

**Q: 应用启动后一直显示「模型加载中」，无法进入聊天？**

检查以下几点：
1. 确认设备上 `llama-server` 在 PATH 中：`hdc shell which llama-server`
2. 确认模型文件存在且可读：`hdc shell ls -lh /etc/model.gguf`
3. 查看应用日志确认进程是否启动：`hdc shell hilog | grep testTag`
4. 手动验证服务是否响应：`hdc shell curl http://127.0.0.1:8080/health`

---

**Q: 发送消息后 AI 没有任何回复？**

可能原因：
- llama-server 尚未完全加载模型，等待就绪提示出现后再发送
- 流式文件写入失败，检查 `context.filesDir` 目录权限
- 查看日志：`hdc shell hilog | grep LlamaService`

---

**Q: 响应速度很慢？**

调整方向：
- 增加 `-t` 线程数（不超过设备物理核心数）
- 减小 `--ctx-size`（降低内存压力）
- 若设备有 NPU/GPU，增加 `--n-gpu-layers`
- 换用量化更激进的模型（如 Q4_K_M → Q3_K_M）
- 减小 `max_tokens` 限制单次生成长度

---

**Q: 如何查看运行日志？**

```bash
hdc shell hilog | grep -E "testTag|LlamaService"
```

---

**Q: 多轮对话上下文超长怎么办？**

`LlamaService.setHistory()` 每次发送前传入完整历史，超出 `--ctx-size` 时 llama-server 会自动截断早期内容。可在 `sendMessage()` 中限制传入的历史条数：

```typescript
// 只保留最近 10 条历史
this.llamaService.setHistory(conv.messages.slice(-10, assistantIndex))
```

---

**Q: 会话数据重启后丢失？**

当前版本会话仅保存在内存中。如需持久化，在 `newConversation` / `sendMessage` 后将 `conversations` 序列化写入 `context.filesDir`，在 `aboutToAppear` 时读取恢复。

---

**Q: 如何修改应用包名？**

修改 `AppScope/app.json5` 中的 `bundleName`，同步更新签名配置中的包名，重新签名后安装。

---

**Q: Release 构建如何开启代码混淆？**

`entry/build-profile.json5` 中 `releaseType` 下已配置混淆规则（property、toplevel、filename、export），确保 `ruleOptions.enable: true` 即可在 Release 构建时自动生效。


