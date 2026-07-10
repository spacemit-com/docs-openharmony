<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-07-09 11:00:00
 * @LastEditTime: 2026-07-09 11:00:00
 * @FilePath: \doc\docs-openharmony\zh\ai_application\7_OH_AI_vlmdemo.md
 * @Description: 
-->
sidebar_position: 7

# VLM 视觉问答应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明       |
|----------|------------|-------------|
| 001      | 2026-07-09 | 初始版本      |

## 概述

vlmdemo 是基于 OpenHarmony 的多模态视觉问答应用，使用 SpaceMIT SMT（SpaceMIT Multi-modal Tensor）加速后端，在设备端直接运行 llama-server VLM 服务，无需联网即可完成图文对话推理。应用启动时自动拉起 SMT llama-server（端口 8071），内置测试图片，支持流式视觉问答。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ❌ 不支持             |
| K1 OpenHarmony6.1     | ❌ 不支持             |
| K3 OpenHarmony6.1     | ✅ 支持               |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS（TypeScript 超集，OpenHarmony 专用） |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 构建工具 | Hvigor + DevEco Studio |
| 推理后端 | llama.cpp llama-server（设备本地进程，SMT 视觉后端） |
| 通信协议 | HTTP / OpenAI 兼容 API（`@ohos.net.http`，本地 127.0.0.1:8071） |
| 视觉加速 | SpaceMIT SMT（libspacemit_ep.so + libonnxruntime.so.1） |

---

## 项目结构

```
vlmdemo/
├── AppScope/                        # 应用级资源与配置
│   ├── app.json5                    # 包名、版本号
│   └── resources/                   # 全局资源（图标等）
├── entry/                           # 主模块（HAP）
│   ├── src/main/ets/
│   │   ├── entryability/
│   │   │   └── EntryAbility.ets     # 应用入口，管理 llama-server 进程
│   │   ├── pages/
│   │   │   └── Index.ets            # 主界面（左图右聊布局）
│   │   └── service/
│   │       └── VlmService.ets       # 与 llama-server 通信的服务层（SSE 流式）
│   └── src/main/resources/
│       └── rawfile/
│           └── default_image.jpg    # 内置测试图片
├── hvigor/                          # 构建系统配置
├── build-profile.json5              # 构建配置（签名、SDK 版本）
└── oh-package.json5                 # 依赖声明
```

---

## 架构

```
┌──────────────────────────────────────────────────┐
│                  OpenHarmony 设备                 │
│                                                  │
│  ┌───────────────────────────────────────────┐   │
│  │               vlmdemo HAP                 │   │
│  │                                           │   │
│  │  ┌─────────────┐     ┌─────────────────┐  │   │
│  │  │ EntryAbility │     │   Index.ets     │  │   │
│  │  │  (生命周期)  │     │ (左图右聊 UI)  │  │   │
│  │  └──────┬───────┘     └────────┬────────┘  │   │
│  │         │ 启动/停止进程         │ 调用      │   │
│  │         │                      ▼           │   │
│  │         │          ┌───────────────────┐   │   │
│  │         │          │    VlmService     │   │   │
│  │         │          │  (HTTP SSE 流式)  │   │   │
│  │         │          └─────────┬─────────┘   │   │
│  └─────────│────────────────────│─────────────┘   │
│            │                    │ @ohos.net.http   │
│            ▼                    ▼                  │
│  ┌──────────────────────────────────────────────┐  │
│  │   llama-server（本地子进程，SMT 后端）        │  │
│  │   127.0.0.1:8071                             │  │
│  │   模型: /etc/vlm/model.gguf                  │  │
│  │   视觉: /etc/vlm/fastvlm_vision.f16.onnx    │  │
│  └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 推理流程

### 应用启动流程

1. `onCreate` 检查 `/etc/vlm/model.gguf` 是否存在，缺失时在界面提示错误
2. `pkill -9 llama-server`（清理旧进程，等待 500ms）
3. 设置 `TMPDIR` / `TMP` / `TEMP` 环境变量指向应用可写目录（`context.tempDir`）
   > SMT 后端（libspacemit_ep.so）推理时需要写临时文件；libspine_tcm.so 硬编码使用 `/tmp/tcm_sync_standalone.shm`，故必须保证临时目录可写
4. 延迟 1500ms 后启动 llama-server：

   ```bash
   export TMPDIR=<tempDir>; export TMP=<tempDir>; export TEMP=<tempDir>;
   export PATH=/usr/bin:/bin:/system/bin;
   mkdir -p <tempDir>; chmod 777 <tempDir>;
   llama-server -m /etc/vlm/model.gguf \
     --vision-backend smt --smt-config-dir /etc/vlm \
     --host 127.0.0.1 --port 8071 \
     -t 8 -tb 8 --reasoning off
   ```

5. 前端每秒轮询 `/health` 端点（最多 60 次），就绪（`{"status":"ok"}`）后解锁输入界面
6. `onDestroy` 时发送 SIGTERM（signal 15）关闭 llama-server

### 视觉问答推理过程

每次用户发送消息时：

1. 从应用资源目录加载内置图片 `default_image.jpg`，转换为 base64 data URI
2. 构造 OpenAI 兼容的多模态消息格式：

   ```json
   {
     "messages": [
       {
         "role": "user",
         "content": [
           { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,..." } },
           { "type": "text", "text": "用户输入的问题" }
         ]
       }
     ],
     "stream": true
   }
   ```

3. 通过 `@ohos.net.http` 发送 POST 请求到 `http://127.0.0.1:8071/v1/chat/completions`
4. 解析 SSE 流式响应（`data: {...}` 格式），逐 token 追加到对话气泡
5. 收到 `finish_reason: stop` 或 `[DONE]` 后结束流式接收

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/vlmDemo.git
cd vlmDemo
```

---

## 模型文件获取

vlmdemo 使用 FastVLM 0.5B 多模态模型（文件位于仓库 `fastvlm-mm-0.5b-q4_1/` 目录）：

| 文件 | 大小 | 说明 |
|------|------|------|
| `fastvlm-text-0.5B-Q4_1.gguf` | 约 876 MB | 文本解码器（Q4_1 量化） |
| `fastvlm_vision.f16.onnx` | 约 241 MB | SMT 视觉编码器（FP16 ONNX） |
| `config.json` | 248 B | 多模态配置（指定 vision/text 模型路径） |

`config.json` 内容示例：

```json
{
  "architectures": ["LlavaQwen2ForCausalLM"],
  "vision_model": { "model_path": "fastvlm_vision.f16.onnx" },
  "text_model": { "model_path": "fastvlm-text-0.5B-Q4_1.gguf", "hidden_size": 896 }
}
```

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
hdc file send libllama.so.0              /system/lib64/libllama.so.0
hdc file send libggml.so.0               /system/lib64/libggml.so.0
hdc file send libggml-base.so.0          /system/lib64/libggml-base.so.0
hdc file send libggml-cpu.so.0           /system/lib64/libggml-cpu.so.0
hdc file send libllama-common.so.0       /system/lib64/libllama-common.so.0
hdc file send libmtmd.so.0               /system/lib64/libmtmd.so.0
hdc file send libllama-server-impl.so    /system/lib64/libllama-server-impl.so

# OnnxRuntime EP 及 TCM 支持库
hdc file send libonnxruntime.so.1.24.2+spacemit.a1  /system/lib64/libonnxruntime.so.1
hdc file send libspacemit_ep.so.2.0.4               /system/lib64/libspacemit_ep.so.2
hdc file send libspine_tcm.so.0.2.0                 /system/lib64/libspine_tcm.so.0

# 创建不带版本号的符号链接
hdc shell "ln -sf /system/lib64/libonnxruntime.so.1  /system/lib64/libonnxruntime.so"
hdc shell "ln -sf /system/lib64/libspacemit_ep.so.2  /system/lib64/libspacemit_ep.so"
hdc shell "ln -sf /system/lib64/libspine_tcm.so.0    /system/lib64/libspine_tcm.so"

# llama-server 启动器
hdc file send llama-server /system/bin/llama-server
hdc shell "chmod 755 /system/bin/llama-server"
```

> 实测说明：**K3 Pico ITX 镜像已预置上述全部库**（SONAME 实体 + 符号链接完整）与 `llama-server`，无需手动推送。若设备已烧录完整镜像，**可直接跳到步骤 3**，仅需推送模型文件。

### 2. 设置 TCM 设备权限

K3 SoC 的 TCM（Tightly Coupled Memory）设备节点默认权限为 660，SMT 后端需要 666：

```bash
hdc shell "chmod 666 /dev/tcm"

# 验证（预期输出）：
# crw-rw-rw- 1 root root 10, 260 /dev/tcm
```

> K3 Pico ITX 镜像已在 init 脚本中配置 `chmod 666 /dev/tcm`，重启后自动恢复。实测该节点已为 666，**可跳过此步骤**。

### 3. 部署 VLM 模型文件

```bash
hdc shell "mkdir -p /etc/vlm"

# 模型配置（248 B）
hdc file send fastvlm-mm-0.5b-q4_1/config.json /etc/vlm/config.json

# SMT 视觉编码器 ONNX（约 241 MB，实测约 10 秒）
hdc file send fastvlm-mm-0.5b-q4_1/fastvlm_vision.f16.onnx /etc/vlm/fastvlm_vision.f16.onnx

# 文本解码器（约 876 MB，实测约 34 秒）
hdc file send fastvlm-mm-0.5b-q4_1/fastvlm-text-0.5B-Q4_1.gguf /etc/vlm/model.gguf

# llmchat 共用路径硬链接（link count 变为 2）
hdc shell "ln -f /etc/vlm/model.gguf /etc/model.gguf"
```

部署完成后验证（实测输出）：

```text
$ hdc shell "ls -lh /etc/vlm/; ls -lh /etc/model.gguf"
-rw-r--r-- 1 root root  248 config.json
-rw-r--r-- 1 root root 241M fastvlm_vision.f16.onnx
-rw-r--r-- 2 root root 876M model.gguf
-rw-r--r-- 2 root root 876M /etc/model.gguf        # link count=2，硬链接生效
```

### 4. 编译安装 HAP

```bash
# 构建（在 vlmdemo 项目根目录执行）
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon

# 安装到设备（需先卸载旧版本）
hdc uninstall com.example.vlmdemo
hdc install -r entry/build/default/outputs/default/entry-default-signed.hap
```

### 5. 启动验证

```bash
# 启动应用（llama-server 由应用自动拉起）
hdc shell "aa start -b com.example.vlmdemo -a EntryAbility"

# 等待约 15-20 秒模型加载完成，确认服务就绪
hdc shell "curl -s http://127.0.0.1:8071/health"
# 预期返回：{"status":"ok"}

# 查看应用日志
hdc shell "hilog | grep -E 'VlmService|EntryAbility'"
```

### 故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| 界面提示"模型文件不存在" | `/etc/vlm/model.gguf` 未部署 | 执行步骤 3 |
| 状态栏一直显示"加载中" | llama-server 启动失败 | `hdc shell "hilog \| grep EntryAbility"` 查看错误 |
| 推理崩溃 / SIGBUS / 卡死 | `/dev/tcm` 权限不足 | `hdc shell "chmod 666 /dev/tcm"` |
| SMT 报 ONNX 错误 | `libspacemit_ep.so` 或 `libonnxruntime.so` 缺失/版本不符 | 检查步骤 1 的库是否完整 |
| 应用闪退 / 无日志 | TMPDIR 不可写 | 代码已自动设置，检查 `context.tempDir` 权限 |
| TCM 资源占满（卡死） | 残留进程占用 TCM 块 | `hdc shell "spacemit-tcm-smi -c"` 强制释放后重试 |

---

## 实测验证记录

以下为在 **SpaceMIT K3 Pico ITX** 真机（riscv64、内核 6.18.3、16 核、16 GB 内存）上的实测结果（`system_fingerprint: b9488-4532ab632`）。

命令行验证方式（应用内由 `EntryAbility` 自动完成，参数一致）：

```bash
hdc shell 'pkill -9 llama-server; sleep 1; \
  export TMPDIR=/data/local/tmp; \
  mkdir -p /data/local/tmp; chmod 777 /data/local/tmp; \
  nohup llama-server -m /etc/vlm/model.gguf \
    --vision-backend smt --smt-config-dir /etc/vlm \
    --host 127.0.0.1 --port 8071 -t 8 -tb 8 --reasoning off \
    > /data/local/tmp/vlm_server.log 2>&1 &'
```

**服务就绪**（约 15 秒完成加载）：

```text
$ hdc shell "curl -s http://127.0.0.1:8071/health"
{"status":"ok"}
```

**纯文本推理**（`stream:false`，提问「用一句话介绍你自己」）：

```text
"finish_reason":"stop"
"usage":{"completion_tokens":48,"prompt_tokens":23,"total_tokens":71}
"timings":{"prompt_per_second":357.17,"predicted_per_second":51.78}
```

生成速度约 **51.8 tok/s**，prompt 处理约 357 tok/s。

**图像推理**（内置 `default_image.jpg` 转 base64，提问「请描述这张图片」）：

```text
"finish_reason":"length"
"usage":{"completion_tokens":128,"prompt_tokens":87,"total_tokens":215}
"timings":{"prompt_per_second":394.27,"predicted_per_second":48.61}
```

模型正确识别出图片内容（四人肖像照），生成速度约 **48.6 tok/s**。SMT 视觉后端与 TCM 加速均生效。

> 图像请求 JSON 可用 PowerShell 生成：
> ```powershell
> $img = [Convert]::ToBase64String([IO.File]::ReadAllBytes("<图片路径>"))
> $body = @{ messages=@(@{ role="user"; content=@(
>     @{ type="image_url"; image_url=@{ url="data:image/jpeg;base64,$img" } },
>     @{ type="text"; text="请描述这张图片" }) }); stream=$false; max_tokens=128 } |
>   ConvertTo-Json -Depth 10 -Compress
> [IO.File]::WriteAllText("img_req.json", $body)
> # 然后：hdc file send img_req.json /data/local/tmp/img_req.json
> #       hdc shell "curl -s http://127.0.0.1:8071/v1/chat/completions -H 'Content-Type: application/json' -d @/data/local/tmp/img_req.json"
> ```

---

## 与 llmchat 的对比

| 项目 | llmchat | vlmdemo |
|------|---------|---------|
| 后端 | CPU（llama.cpp） | SMT（SpaceMIT Multi-modal Tensor） |
| 端口 | 8080 | 8071 |
| 模型路径 | `/etc/model.gguf` | `/etc/vlm/model.gguf` |
| 启动参数 | `-t 4 --ctx-size 4096` | `--vision-backend smt --smt-config-dir /etc/vlm -t 8 -tb 8 --reasoning off` |
| 消息格式 | 纯文本 | 含 `image_url` 内容块（base64 data URI） |
| 图片来源 | — | 内置 `default_image.jpg`，随每次提问自动附带 |
| 需要 TMPDIR | 否 | 是（SMT 推理写临时文件） |
| 需要 ONNX 库 | 否 | 是（libspacemit_ep.so + libonnxruntime.so.1） |

---

## FAQ

**Q: 状态栏一直显示「加载中」，无法对话？**

1. 确认模型文件已部署：`hdc shell "ls -lh /etc/vlm/"`，三个文件均需存在
2. 确认 `llama-server` 已在 PATH：`hdc shell "which llama-server"`
3. 查看日志：`hdc shell "hilog | grep EntryAbility"`
4. 手动验证：`hdc shell "curl -s http://127.0.0.1:8071/health"`

**Q: 推理中途卡死或崩溃？**

- 先检查 TCM 权限：`hdc shell "ls -l /dev/tcm"`，需为 `crw-rw-rw-`
- TCM 可能被残留进程占满：`hdc shell "spacemit-tcm-smi"` 查看，`spacemit-tcm-smi -c` 释放

**Q: 响应速度很慢（<10 tok/s）？**

- 确认 SMT 后端生效：日志中应出现 `vision-backend smt` 字样
- 确认 `/dev/tcm` 权限为 666（TCM 加速必须）
- `-t 8 -tb 8` 已是针对 K3 16 核的推荐值

**Q: 如何更换为自定义图片？**

替换 `entry/src/main/resources/rawfile/default_image.jpg`，重新编译 HAP 即可。图片会在每次发送时自动编码为 base64 附带到请求中。

**Q: 如何支持用户上传图片（而非内置图片）？**

在 `VlmService.ets` 的图片加载逻辑处改为从 `context.filesDir` 读取用户选择的图片，并在 `Index.ets` 接入 `@ohos.file.picker` 实现图片选择。

**Q: 可以与 llmchat 共用模型吗？**

可以。将模型推送到 `/etc/vlm/model.gguf` 后，用硬链接共享给 llmchat：

```bash
hdc shell "ln -f /etc/vlm/model.gguf /etc/model.gguf"
```

两个应用互不干扰（端口不同），但不能同时运行（同时启动会竞争 TCM 资源）。
