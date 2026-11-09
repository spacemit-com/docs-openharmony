<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-07-09 11:00:00
 * @LastEditTime: 2026-11-07 10:00:00
 * @FilePath: \doc\docs-openharmony\zh\ai_application\7_OH_AI_vlmdemo.md
 * @Description: 
-->
sidebar_position: 7

# VLM 视觉问答应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明 |
|----------|------------|----------|
| 001      | 2026-07-09 | 初始版本 |
| 002      | 2026-11-07 | 对齐最新代码并依据 K3 平台结果校正：新增模型中心，更新模型大小、加载与生成性能 |

## 概述

vlmdemo 是基于 OpenHarmony 的多模态视觉问答应用，使用 SpaceMIT SMT 加速后端，在设备端运行 `llama-server` VLM 服务，无需联网即可完成图文对话推理。

已下载并选中模型时，应用启动会自动拉起镜像自带的 `llama-server`（端口 8071）；模型由内置「模型中心」从 `archive.spacemit.com` 下载并解压到应用沙箱，支持 FastVLM 与 Qwen3.5-VL 多档模型，可运行时切换。左侧内置测试图片，提问时自动以 base64 随请求发送。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ❌ 不支持             |
| K1 OpenHarmony6.1    | ❌ 不支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 构建工具 | Hvigor + DevEco Studio |
| 推理后端 | llama.cpp `llama-server`（镜像自带，版本 b10056，SMT 视觉后端） |
| 通信协议 | HTTP / OpenAI 兼容 API（`@ohos.net.http`，`127.0.0.1:8071`） |
| 视觉加速 | SpaceMIT SMT（`libspacemit_ep.so.2` 版本 2.0.7 + `libonnxruntime.so.1` 1.24.2，镜像提供） |
| 模型管理 | 应用内下载 tar.gz、md5 校验、`tar -xzf` 解压、沙箱存储 |

---

## 项目结构

```
vlmdemo/
├── AppScope/                        # 应用级资源与配置
├── entry/src/main/ets/
│   ├── entryability/
│   │   └── EntryAbility.ets         # 启动：选模型、拉起 llama-server
│   ├── pages/
│   │   ├── Index.ets                # 左图右聊界面
│   │   └── ModelCenter.ets          # 模型中心（下载/使用/删除/启停服务）
│   ├── service/
│   │   ├── VlmLauncher.ets          # llama-server 启停与健康检查
│   │   ├── VlmService.ets           # SSE 流式视觉问答
│   │   ├── ModelCatalog.ets         # VLM 模型清单
│   │   ├── ModelDownloader.ets      # 下载 + 校验 + 解压
│   │   └── ModelStore.ets           # 目录布局、选中项、旧版识别
│   ├── model/ChatMessage.ets
│   └── src/main/resources/rawfile/
│       └── default_image.jpg        # 内置测试图片
├── build-profile.json5
└── oh-package.json5
```

---

## 架构

```
┌──────────────────────────────────────────────────┐
│                  OpenHarmony 设备                 │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │                vlmdemo HAP                 │  │
│  │  ┌───────────┐      ┌──────────────────┐   │  │
│  │  │ Index.ets │      │ ModelCenter.ets  │   │  │
│  │  │ 左图右聊  │      │ 模型下载/切换     │   │  │
│  │  └─────┬─────┘      └────────┬─────────┘   │  │
│  │        │                     │             │  │
│  │  ┌─────▼──────┐      ┌───────▼──────────┐  │  │
│  │  │ VlmService │      │ ModelDownloader  │  │  │
│  │  │ HTTP + SSE │      │ ModelStore       │  │  │
│  │  └─────┬──────┘      └───────┬──────────┘  │  │
│  └────────│─────────────────────│─────────────┘  │
│           │ 127.0.0.1:8071      │ 解压到沙箱
│           ▼                     ▼                │
│  ┌────────────────────┐  ┌────────────────────┐  │
│  │ llama-server（镜像）│  │ filesDir/models/   │  │
│  │ SMT 视觉后端        │  │ <id>/config.json   │  │
│  │                    │  │ <id>/*.gguf/*.onnx │  │
│  └────────────────────┘  └────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 推理流程

### 应用启动

1. 初始化 `ModelStore`，读取 `filesDir/selection.json` 中的选中模型。
2. 未选择时，按清单顺序自动选中第一个已下载模型；没有可用模型则提示去模型中心。
3. 选中模型目录完整（含 `config.json` 和至少一个 `.gguf`）时：
   - 复用已在运行且健康检查通过的服务；
   - 否则设置 `TMPDIR/TMP/TEMP` 为应用 `tempDir`，执行：

```bash
llama-server -m <models/<id>>/<目录内第一个 .gguf> \
  --vision-backend smt --smt-config-dir <models/<id>> \
  --host 127.0.0.1 --port 8071 -t 8 -tb 8 --reasoning off
```

4. 每 2 s 轮询 `/health`，最长 90 s；就绪后解锁输入。

> SMT 后端推理需要写临时文件，`libspine_tcm.so.3` 还使用共享内存，因此必须保证 `tempDir` 可写；应用已自动处理，无需手工设置。

### 视觉问答

1. 从 `rawfile` 加载内置图片，转换为 `data:image/jpeg;base64,...`。
2. 构造 OpenAI 多模态消息：`image_url` 内容块 + `text` 内容块。
3. `@ohos.net.http` 流式 POST 到 `/v1/chat/completions`（`stream:true`，`temperature:0.7`，`max_tokens:1024`）。
4. 解析 SSE，逐 token 追加到对话气泡。

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/vlmDemo.git
cd vlmDemo
```

---

## 模型获取与管理

模型由应用内「模型中心」下载，无需 PC 下载后再推送。

| 模型 | 参数量 | 包大小（约） | 特点 |
|------|--------|-------------|------|
| FastVLM 0.5B | 0.5B | 766 MB | 文本 gguf + ONNX 视觉编码器，体积最小、加载最快 |
| Qwen3.5-VL 0.8B | 0.8B | 932 MB | 推荐首选，典型约 13 秒就绪 |
| Qwen3.5-VL 2B | 2B | 2.59 GB | 质量更好，解包需要更多空间 |
| Qwen3.5-VL 4B | 4B | 3.93 GB | 效果最强，解包峰值约需 8 GB 空闲空间 |

- 下载地址：`http://archive.spacemit.com/spacemit-ai/model_zoo/vlm/<模型>.tar.gz`（设备不信任该站点证书，统一走 HTTP）。
- 流程：流式下载为 `.part` → md5 校验 → 改名为 `.tar.gz` → `tar -xzf` 解压到 `filesDir/models/<id>/` → 删除压缩包。
- 包内 `config.json` 使用相对路径，解压后目录内文件不能改名或移动。
- 选中项保存在 `filesDir/selection.json`；切换模型会停止旧服务并以新模型重启。

---

## 设备部署

无需推送模型或推理库，安装 HAP 后首次进入「模型中心」下载模型即可。

> 部分镜像预置的 vlmdemo 可能是旧版本（无「模型」入口）；若界面没有模型中心，请按最新源码构建 HAP 后安装。

```bash
# 安装
hdc uninstall com.example.vlmdemo
hdc install -r entry-default-signed.hap

# 启动（应用自动拉起 llama-server）
hdc shell "aa start -b com.example.vlmdemo -a EntryAbility"

# 等待约 6~13 秒（模型不同），确认服务就绪
hdc shell "curl -s http://127.0.0.1:8071/health"
# 预期：{"status":"ok"}

# 查看日志
hdc shell "hilog | grep -E 'VlmLauncher|VlmService|ModelCenter'"
```

### TCM 权限

SMT 后端需要 `/dev/tcm` 为 666。K3 Pico ITX 镜像已由 init 脚本配置，重启后自动恢复；若推理崩溃或卡死，检查并手工修正：

```bash
hdc shell "ls -l /dev/tcm"          # 预期 crw-rw-rw-
hdc shell "chmod 666 /dev/tcm"
hdc shell "spacemit-tcm-smi"        # 查看占用
hdc shell "spacemit-tcm-smi -c"     # 清理残留占用
```

### 故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| 提示「尚未下载模型」 | 沙箱内没有可用模型 | 到模型中心下载并点「使用」 |
| 状态一直「加载中」 | llama-server 启动失败 | `hilog \| grep VlmLauncher` 查看命令与错误 |
| 推理崩溃 / SIGBUS | `/dev/tcm` 权限不足或 TCM 被占满 | 修正权限，`spacemit-tcm-smi -c` 后重试 |
| SMT 报 ONNX 错误 | 镜像缺少 EP/ORT 库 | 检查 `/system/lib64/libspacemit_ep.so.2` 等是否存在 |
| 下载后解压失败 | 空间不足或压缩包不完整 | 清理其他模型，重新下载；解压时保持应用在前台 |
| 解压日志出现 `tar: /etc/passwd: bad record` | toybox tar 的用户库查询告警 | 不影响解压，tar 退出码为 0，可忽略 |

---

## 性能参考数据

K3 Pico ITX（riscv64、8 核、16 GB 内存，OpenHarmony 6.1.0.35 / API 23），llama.cpp b10056，SMT 视觉后端：

| 项目 | FastVLM 0.5B | Qwen3.5-VL 0.8B |
|------|--------------|-----------------|
| 服务加载 | 5.8 秒 | 12.6 秒 |
| 纯文本生成 | 34.0 tok/s（prompt 272 tok/s） | 20.8 tok/s（prompt 88 tok/s） |
| 图像推理 | 33.6 tok/s（prompt 323 tok/s） | 20.2 tok/s（prompt 126 tok/s） |
| 识别结果 | 正确描述图片 | 正确描述图片 |

> 两种模型均按文档中的启动命令运行；请求体建议写入文件后用 `curl -d @file` 发送，避免多层 shell 吞引号。

---

## 与 llmchat 的对比

| 项目 | llmchat | vlmdemo |
|------|---------|---------|
| 后端 | CPU（llama.cpp） | SMT（SpaceMIT Multi-modal Tensor） |
| 端口 | 8080 | 8071 |
| 模型格式 | 单文件 `.gguf` | 目录（`config.json` + gguf + onnx） |
| 模型来源 | 模型中心下载 | 模型中心下载并解压 |
| 启动参数 | `--ctx-size 15360 --batch-size 512 ...` | `--vision-backend smt --smt-config-dir <dir> -t 8 -tb 8` |
| 消息格式 | 纯文本 | 含 `image_url` 内容块（base64 data URI） |
| 临时目录 | 不需要 | 需要（SMT 推理写临时文件） |

---

## FAQ

**Q: 启动后提示「模型文件缺失」？**

说明 `selection.json` 指向的模型目录不完整。进入模型中心重新下载，或删除该模型后选择其他模型。

**Q: 状态栏一直显示「加载中」？**

1. 确认模型目录中存在 `config.json` 和 `.gguf`。
2. 确认 `llama-server` 在 PATH：`hdc shell which llama-server`。
3. 查看日志：`hdc shell "hilog | grep VlmLauncher"`。
4. 手动验证：`hdc shell "curl -s http://127.0.0.1:8071/health"`。

**Q: 推理中途卡死或崩溃？**

先检查 `/dev/tcm` 权限与占用；TCM 被残留进程占满时用 `spacemit-tcm-smi -c` 释放后重试。

**Q: 响应速度很慢（<10 tok/s）？**

- 确认日志中出现 `--vision-backend smt`。
- 确认 `/dev/tcm` 为 666。
- `-t 8 -tb 8` 已是 K3 8 核的推荐值。

**Q: 如何更换为自定义图片？**

替换 `entry/src/main/resources/rawfile/default_image.jpg` 后重新编译；图片会在每次提问时自动编码为 base64 附带。

**Q: 如何支持用户自选图片？**

在 `VlmService.ets` 的图片加载处改为从 `context.filesDir` 读取用户选择的图片，并在 `Index.ets` 接入 `@ohos.file.picker`。

**Q: 能与 llmchat 同时运行吗？**

两者端口不同，但会竞争 TCM 资源，不建议同时运行；vlmdemo 在运行时也不要再启动其他 SMT 推理任务。
