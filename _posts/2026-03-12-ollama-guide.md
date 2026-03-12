---
layout: post
title: "AI 系列 (1)：Ollama 入门 - 模型选择与配置优化"
date: 2026-03-12 15:00:00 +0800
categories: AI
tags: [Ollama, LLM, 大模型]
description: "从零开始使用 Ollama，详解模型选择、显存优化与生产环境配置"
---

## 📚 前言

AI 系列文章将记录我使用 **Ollama** 配合 **OpenClaw** 的探索与实践。这是第一篇，带你了解如何选择合适的模型并进行高效配置。

---

## 一、为什么选择 Qwen3.5 27B？

### 1.1 显存与性能平衡

对于配备 **24GB 显存** 的 GPU（如 RTX 3080/4080），`qwen3.5:27b` 是黄金选择：

```bash
# 加载 27B 参数的 4bit 量化模型
ollama pull jaahas/qwen3.5-uncensored:27b
```

**为什么是 27B？**

| 模型规模 | 显存占用 (4bit) | 适用场景 |
|---------|----------------|---------|
| 7B      | ~5GB           | 快速响应，简单任务 |
| **27B**   | **~18GB**      | ⭐ 平衡点：复杂任务 + 留余量 |
| 70B     | ~45GB          | 需要更高显存 |

### 1.2 模型特点

- ✅ **工具调用能力 (Tool Use)** - 支持 Function Calling，适合 Agent 架构
- ✅ **多模态支持** - 后续可扩展视觉任务
- ✅ **中文优化** - Qwen 系列对中文理解优秀
- ✅ **推理速度快** - 在消费级显卡上可达 10+ tokens/s

---

## 二、Fallback 模型推荐

当主模型负载过高或需要降级时，可准备 fallback 模型：

```bash
# Fallback 模型列表（根据显存需求选择）
ollama pull qwen3.5:7b      # 轻量级fallback
ollama pull llava           # 视觉任务fallback
```

**推荐策略：**

| 场景 | 主模型 | Fallback |
|-----|--------|----------|
| 复杂推理 | `jaahas/qwen3.5-uncensored:27b` | `qwen3.5:7b` |
| 快速问答 | `llama3.2:3b` | - |
| 视觉任务 | `llava` | `bakLLaVA` |

> 💡 **重要提示**：选择模型时务必确认支持 **Tool/Function Calling** 功能，这是 Agent 系统的核心能力！

---

## 三、Context Length 配置（推荐 64000）

### 3.1 为什么要调整 Context Length？

默认情况下 Ollama 使用 2048 或 4096 的上下文窗口。对于 AI Agent 来说，这远远不够：

- ✅ **长文本处理** - 阅读整篇文档/报告
- ✅ **多轮对话** - 保持更长的对话历史
- ✅ **复杂任务** - 保存更多工具调用状态

### 3.2 systemd 配置示例（Linux）

编辑 `/etc/systemd/system/ollama.service`：

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
Type=simple
Restart=always
RestartSec=3
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
# 推荐配置：64000 context length
ExecStart=/usr/bin/ollama serve --ctx-size 64000
User=ollama

[Install]
WantedBy=multi-user.target
```

**应用配置：**

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 3.3 验证 Context Length

```bash
# 查看当前上下文长度
ollama run jaahas/qwen3.5-uncensored:27b "你当前的上下文窗口大小是多少？" --ctx-size 64000
```

---

## 四、局域网访问配置

### 4.1 启用公网可访问

默认 Ollama 仅监听 localhost，需要修改服务配置：

```ini
# /etc/systemd/system/ollama.service
[Service]
ExecStart=/usr/bin/ollama serve --host 0.0.0.0 --ctx-size 64000
```

**重启服务：**

```bash
sudo systemctl restart ollama

# 检查状态
systemctl status ollama
```

### 4.2 验证局域网访问

从其他设备浏览器访问：`http://<你的IP>:11434`

或测试 API：

```bash
curl http://你的局域网IP:11434/api/tags
```

---

## 五、指定 Models 路径

### 5.1 配置自定义模型存储路径

如果希望将模型存放在非默认位置（如大容量硬盘）：

```ini
# /etc/systemd/system/ollama.service
[Service]
Environment="OLLAMA_MODELS=/data/models"
ExecStart=/usr/bin/ollama serve --host 0.0.0.0 --ctx-size 64000
```

**创建目录并设置权限：**

```bash
sudo mkdir -p /data/models
sudo chown -R ollama:ollama /data/models
```

### 5.2 复制/迁移模型

```bash
# 查看当前模型
ollama list

# 导出模型（复制到另一台机器）
ollama cp qwen3.5:27b custom/qwen-27b

# 导入模型
ollama import model.gguf mymodel
```

---

## 六、监控 GPU 使用

### 6.1 查看 Ollama 进程

```bash
ollama ps
```

输出示例：

```
NAME              IDENTITY             SIZE     PROCESSOR    USED BY
qwen3.5:27b       sha256:...          16.8 GB  GPU          qwen-12345
```

### 6.2 结合 nvidia-smi 查看显存占用

```bash
# 简洁模式（适合终端）
watch -n 1 nvidia-smi

# 详细模式（查看占用进程）
nvidia-smi -c 0 --query-gpu=utilization.gpu,memory.used,memory.free --format=csv
```

**典型输出：**

```
| GPU | Name        | Memory-Usage | Temperature |
|-----|-------------|--------------|-------------|
|  0  | RTX 3080    | 18432 / 24576 MiB | 65°C      |
```

---

## 七、完整配置示例（一键部署）

### 7.1 systemd 服务文件

```ini
# /etc/systemd/system/ollama.service
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
Type=simple
Restart=always
RestartSec=3
User=ollama
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_MODELS=/data/models"
ExecStart=/usr/bin/ollama serve --host 0.0.0.0 --ctx-size 64000

[Install]
WantedBy=multi-user.target
```

### 7.2 一键部署脚本

```bash
#!/bin/bash
# ollama-setup.sh

set -e

# 1. 下载模型
echo "📥 Pulling jaahas/qwen3.5-uncensored:27b..."
ollama pull jaahas/qwen3.5-uncensored:27b

# 2. 拉取 fallback 模型
echo "📥 Pulling fallback model..."
ollama pull qwen3.5:7b

# 3. 重启服务应用新配置
echo "🔄 Restarting Ollama service..."
sudo systemctl daemon-reload
sudo systemctl restart ollama

# 4. 检查状态
echo "✅ Checking status..."
ollama ps
echo ""
echo "🎉 Setup complete! Models ready."
```

**执行脚本：**

```bash
chmod +x ollama-setup.sh
./ollama-setup.sh
```

---

## 八、总结与建议

### 推荐配置清单（24GB 显存）

| 配置项 | 推荐值 |
|-------|--------|
| 主模型 | `jaahas/qwen3.5-uncensored:27b` (4bit) |
| Fallback 模型 | `qwen3.5:7b` |
| Context Length | **64000** |
| 并行数 (`NUM_PARALLEL`) | 1-2（显存紧张时） |
| 监听地址 | `0.0.0.0`（局域网访问） |
| Model 路径 | `/data/models`（根据磁盘规划） |

### 下一步

下一篇将介绍 **OpenClaw 如何调用 Ollama 模型**，并展示实际 Agent 场景中的使用技巧。

---

*发布于 2026 年 3 月 12 日*  
*AI 系列 · 第一篇文章*
