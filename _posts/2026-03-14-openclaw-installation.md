---
layout: post
title: "AI 系列 (5)：OpenClaw 安装部署指南 - 从 Docker 到物理机"
date: 2026-03-14 01:30:00 +0800
categories: AI
tags: [OpenClaw, Docker, 部署，安装指南]
description: "详细介绍 OpenClaw 的多种安装方式，包括 Docker 部署、物理机部署、云服务器部署的对比分析，以及安装后的配置要点"
---

## 📚 前言

本篇将系统介绍 OpenClaw 的安装部署方案。作为 AI Agent 平台，OpenClaw 支持多种部署方式，选择合适的方案对于后续使用至关重要。

---

## 一、安装方式对比

### 1.1 四种部署方案

| 方案 | 适用场景 | 优点 | 缺点 | 推荐指数 |
|------|----------|------|------|----------|
| **Mac mini 物理机** | 个人开发、长期使用 | 性能稳定、完全掌控、无额外成本 | 需要硬件投入、受限于设备 | ⭐⭐⭐⭐⭐ |
| **Docker 部署** | 快速体验、多环境切换 | 隔离环境、易于迁移、版本管理 | 权限问题、工具链兼容 | ⭐⭐⭐⭐ |
| **Linux 物理机** | 服务器部署、团队使用 | 性能最优、完全控制 | 需要运维知识 | ⭐⭐⭐⭐⭐ |
| **VPS 云部署** | 远程访问、多设备共享 | 随时访问、无需本地硬件 | 持续费用、网络延迟 | ⭐⭐⭐ |

### 1.2 选择建议

```
┌─────────────────────────────────────────────────────────────┐
│                    部署方案选择流程                          │
└─────────────────────────────────────────────────────────────┘

有 Mac mini？
    ├── 是 → 推荐 Mac mini 物理机部署
    └── 否
          ├── 需要远程访问？
          │     ├── 是 → VPS 云部署
          │     └── 否
          │           ├── 有 Linux 服务器？
          │           │     ├── 是 → Linux 物理机部署
          │           │     └── 否 → Docker 本地部署
          │           └── 想快速体验？
          │                 └── 是 → Docker 部署
```

---

## 二、安装方式详解

### 2.1 本地安装（推荐）

官方推荐的安装方式，适合长期使用。

**一键安装（macOS/Linux）：**

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

**一键安装（Windows PowerShell）：**

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

**手动安装：**

```bash
# 安装 Node.js（推荐 Node 24）
# macOS
brew install node@24

# Linux (Ubuntu/Debian)
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs

# 安装 OpenClaw
npm install -g openclaw@latest

# 验证安装
openclaw --version
```

**运行向导：**

```bash
# 运行配置向导，安装服务
openclaw onboard --install-daemon

# 检查 Gateway 状态
openclaw gateway status

# 打开控制面板
openclaw dashboard
```

**目录结构：**

```
~/.openclaw/
├── openclaw.json      # 主配置文件
├── state/             # 运行状态
├── workspace/         # 工作目录（重要！）
│   ├── MEMORY.md      # 长期记忆
│   ├── USER.md        # 用户信息
│   ├── SOUL.md        # Agent 人格
│   └── memory/        # 记忆存储
└── logs/              # 日志文件
```

---

### 2.2 Docker 安装

适合快速体验和多环境切换。

**准备工作：**

```bash
# macOS
brew install docker docker-compose

# Linux (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

**克隆仓库：**

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

**目录结构：**

```
openclaw/
├── Dockerfile
├── docker-compose.yml
├── config/           # 配置文件目录
├── workspace/        # 工作目录（重要！）
└── scripts/
```

**配置文件夹映射（docker-compose.yml）：**

```yaml
version: '3.8'

services:
  openclaw:
    build:
      context: .
      dockerfile: Dockerfile
    image: openclaw-local
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "19001:19001"   # Gateway 端口
      - "19002:19002"   # Browser 端口
    volumes:
      - ./config:/app/config
      - ./workspace:/app/workspace
      - ./skills:/app/skills
    environment:
      - OPENCLAW_STATE_DIR=/app/workspace
      - TZ=Asia/Shanghai
    user: "${UID}:${GID}"
```

**构建和运行：**

```bash
# 构建镜像
docker build -t openclaw-local .

# 启动服务
docker compose up -d

# 查看日志
docker compose logs -f

# 停止服务
docker compose down
```

---

### 2.3 Docker 安装的局限性

| 问题 | 说明 | 解决方案 |
|------|------|----------|
| **root 权限问题** | 容器内默认 root 运行 | 使用 `user: "${UID}:${GID}"` |
| **PATH 工具链问题** | 容器内缺少系统工具 | 在 Dockerfile 中安装 |
| **网络访问** | 容器网络隔离 | 使用 `network_mode: host` |
| **GPU 支持** | 需要 nvidia-docker | 配置 NVIDIA Container Toolkit |

**Dockerfile 补充工具链：**

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y \
    git curl jq bash \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="/usr/local/bin:/usr/bin:/bin:${PATH}"

WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```

---

### 2.4 本地 vs Docker 对比

| 对比项 | 本地安装 | Docker 安装 |
|--------|----------|-------------|
| 安装速度 | 快（一键脚本） | 中等（需构建镜像） |
| 环境隔离 | 否 | 是 |
| 工具链访问 | 直接访问 | 需要配置 |
| 权限问题 | 无 | 需要处理 |
| 更新维护 | 简单 | 需重建镜像 |
| 推荐场景 | 日常使用 | 测试/多版本 |

---

## 三、安装后配置

### 3.1 配置 Token

**获取 AI 模型 Token：**

| 模型提供商 | 获取方式 | 环境变量 |
|------------|----------|----------|
| OpenAI | platform.openai.com → API Keys | `OPENAI_API_KEY` |
| Anthropic | console.anthropic.com | `ANTHROPIC_API_KEY` |
| Google AI | aistudio.google.com | `GOOGLE_API_KEY` |
| 本地 Ollama | 无需 Token | - |

**配置方式：**

```bash
# 方式一：环境变量（推荐）
export OPENAI_API_KEY="sk-xxx..."
export ANTHROPIC_API_KEY="sk-ant-xxx..."

# 方式二：写入 config.yaml
# config/config.yaml
models:
  default: openai/gpt-4
  
credentials:
  openai:
    api_key: ${OPENAI_API_KEY}
```

### 3.2 局域网访问配置

**默认配置：** OpenClaw Gateway 默认监听 `127.0.0.1:19001`

**启用局域网访问：**

```yaml
# config/config.yaml
gateway:
  host: 0.0.0.0
  port: 19001
  
# 或使用环境变量
export GATEWAY_HOST=0.0.0.0
export GATEWAY_PORT=19001
```

**防火墙配置：**

```bash
# Ubuntu/Debian
sudo ufw allow 19001/tcp
sudo ufw reload

# CentOS/RHEL
sudo firewall-cmd --add-port=19001/tcp --permanent
sudo firewall-cmd --reload
```

**访问地址：**

```
本机：http://localhost:19001
局域网：http://<服务器 IP>:19001
示例：http://192.168.1.100:19001
```

---

## 四、常见问题

### Q1：Docker 容器无法访问宿主机网络

```yaml
# 使用 host 网络模式
services:
  openclaw:
    network_mode: host
```

### Q2：权限问题导致文件无法写入

```bash
# 设置正确的用户权限
sudo chown -R $USER:$USER ./workspace
sudo chmod -R 755 ./workspace
```

### Q3：Token 配置不生效

```bash
# 检查环境变量
echo $OPENAI_API_KEY

# 重启服务
docker compose restart
```

### Q4：局域网无法访问

```bash
# 检查端口监听
netstat -tlnp | grep 19001

# 检查防火墙
sudo ufw status
```

---

## 五、总结

### 核心要点

1. **部署选择** - 根据需求选择合适的部署方式
2. **Docker 注意** - 注意权限和工具链问题
3. **Token 配置** - 优先使用环境变量
4. **局域网访问** - 配置 `host: 0.0.0.0` 和防火墙

### 推荐配置

| 场景 | 推荐方案 |
|------|----------|
| 个人日常使用 | Mac mini 物理机 |
| 快速体验 | Docker 部署 |
| 团队协作 | Linux 服务器 |
| 远程访问 | VPS + Tailscale |

---

> **下一篇预告：** 深入详解 OpenClaw 的具体配置项和 Channel 配置指南，带你完成完整的系统搭建！

---

*发布于 2026 年 3 月 14 日*
*AI 系列 · 第五篇文章*
