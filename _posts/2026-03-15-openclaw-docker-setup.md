---
layout: post
title: "OpenClaw Docker 部署完全指南"
date: 2026-03-15
categories: [AI, OpenClaw]
tags: [OpenClaw, Docker, QQBot, 部署]
author: joyfun
---

## 安装完成启动向导

下面配置了 Docker 环境，运行配置向导：

```bash
docker compose up -d
```

第一次的时候会自动运行向导，生成 `.env` 为后续的 gateway 使用。

## 环境变量配置

这里是我自己的 `.env` 文件：

```bash
OPENCLAW_CONFIG_DIR=/your/data/config
OPENCLAW_WORKSPACE_DIR=/your/data/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=YourToken
OPENCLAW_IMAGE=openclaw:local
OPENCLAW_EXTRA_MOUNTS=
OPENCLAW_HOME_VOLUME=
OPENCLAW_DOCKER_APT_PACKAGES=
OPENCLAW_EXTENSIONS=
OPENCLAW_SANDBOX=
OPENCLAW_DOCKER_SOCKET=/var/run/docker.sock
DOCKER_GID=
OPENCLAW_INSTALL_DOCKER_CLI=
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=
GITHUB_TOKEN=yourKey
WECHAT_APPID=yourKey
WECHAT_SECRET=yourKey
```

### ⚠️ 重要配置说明

如果需要**局域网访问**，需要设置：

```bash
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=YourToken
```

其中 `YourToken` 就是后续登录的密码。

以下变量是后续用到的，如果不需要可以不用设置：

```bash
GITHUB_TOKEN=yourKey
WECHAT_APPID=yourKey
WECHAT_SECRET=yourKey
```

## 修改 docker-compose.yml

在 `environment` 部分加入变量：

```yaml
environment:
  GITHUB_TOKEN: ${GITHUB_TOKEN:-}
  WECHAT_APPID: ${WECHAT_APPID:-}
  WECHAT_SECRET: ${WECHAT_SECRET:-}
  NVIDIA_API_KEY: ${NVIDIA_API_KEY:-}
  PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/node/.openclaw/workspace/bin
```

> 💡 **注意**：原来的配置文件中没有 `PATH` 这部分。因为 Docker 镜像是基于 node24，很多工具是没有的。在后续使用 GitHub 功能时，OpenClaw 自己安装软件总是失败，包括最基本的 `write` 命令都没有。而且默认的 shell 权限较低，`apt` 也装不了软件。

### 🔧 一个偷懒的方法

缺失的工具链不用自己编译了，直接从宿主的 Ubuntu 复制过去。

更好的方案是单独添加一个 `PATH`，这样就有上面的配置了。

> ⚠️ **安全提醒**：这有较大的安全隐患，如果不清楚请勿修改 `PATH`。

## openclaw.json 配置

在 `/your/data/config/openclaw.json` 里面有些配置需要修改。

### 配置本地 Ollama

具体使用模型根据实际来选择，在 `models` 下面添加：

```json
"ollama": {
  "baseUrl": "http://192.168.8.123:11434",
  "apiKey": "ollama-local",
  "api": "ollama",
  "models": [
    {
      "id": "jaahas/qwen3.5-uncensored:27b",
      "name": "jaahas/qwen3.5-uncensored:27b",
      "contextWindow": 64000,
      "maxTokens": 64000
    },
    {
      "id": "qwen3.5:9b",
      "name": "qwen3.5:9b",
      "contextWindow": 64000,
      "maxTokens": 64000
    }
  ]
}
```

### 配置 Agents

在 `agents` 部分添加：

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "ollama/jaahas/qwen3.5-uncensored:27b"
    },
    "models": {
      "nvidia/z-ai/glm5": {},
      "nvidia/moonshotai/kimi-k2.5": {},
      "nvidia/minimaxai/minimax-m2.1": {},
      "ollama/jaahas/qwen3.5-uncensored:27b": {}
    },
    "compaction": {
      "mode": "safeguard"
    }
  },
  "list": [
    {
      "id": "main",
      "model": {
        "primary": "nvidia/z-ai/glm5",
        "fallbacks": [
          "ollama/jaahas/qwen3.5-uncensored:27b"
        ]
      }
    }
  ]
}
```

## 常用命令

如果错过了配置向导，后续还可以手动运行：

```bash
docker compose run --rm openclaw-cli onboard
```

启动 Gateway：

```bash
docker compose up -d openclaw-gateway
```

打开控制面板链接：

```bash
docker compose run --rm openclaw-cli dashboard --no-open
```

更详细的配置可以访问[官方文档](https://docs.openclaw.ai/install/docker)。

## 访问 Dashboard

如果在前面的步骤配置好了，下面就可以在浏览器访问：

```
http://localhost:18789
```

20260313 版本的 Dashboard UI 进行了较大的更新，界面更易用：

![OpenClaw Dashboard](/assets/images/openclaw-dashboard-2026-03-15.png)

## 添加 Channel

默认 Dashboard 功能强大（其实 0313 之前的版本功能也很差，多媒体文件之类的也不支持）。

### Telegram 配置

Telegram 配置比较简单，推荐配置一下。

### WhatsApp

WhatsApp 要商业账号，暂时我也没配置。

### QQ Bot 配置 🎯 推荐

访问 [https://q.qq.com/qqbot/openclaw/login.html](https://q.qq.com/qqbot/openclaw/login.html) 添加 QQBot。

安装方式很简单：

```bash
docker compose run -T --rm openclaw-cli plugins install @tencent-connect/openclaw-qqbot@latest
docker compose run -T --rm openclaw-cli channels add --channel qqbot --token "qnum:token"
```

QQBot 支持的格式多，很方便。

> ⚠️ 修改这篇文章的时候，升级 OpenClaw 到最新版，Telegram channel 挂掉了，所以**多个 channel 还是很有必要的**。

![QQBot Channel 配置](/assets/images/qqbot-channel-setup.png)

## 开始使用

配置好 Channel 之后，基本上就可以正式使用 OpenClaw 了。

后续的文章我会从 OpenClaw 的实际使用说明下 Memory 和 Skill（虽然说 SOUL 很重要，但是我现在还没配置，先不用着急）。

---

*下一篇预告：OpenClaw Memory 与 Skill 实战*
