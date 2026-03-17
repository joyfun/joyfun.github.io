---
layout: post
title: "AI 系列 (6)：OpenClaw Skills 详解——以博客发布为例"
date: 2026-03-17 03:45:00 +0800
categories: AI
tags: [OpenClaw, Skills, 自动化, 博客, 微信公众号]
description: "深入介绍 OpenClaw 的 Skills 机制，通过博客自动发布的实际案例，展示如何创建、配置和使用 Skill 扩展 AI Agent 的能力"
---

## 📚 前言

在前几篇文章中，我们介绍了 Ollama 本地部署、OpenClaw 安装配置以及微信公众号自动同步。本篇将深入探讨 OpenClaw 的核心机制——**Skills（技能）**，并通过一个实际的博客发布案例，展示如何创建和定制自己的 Skill。

---

## 一、什么是 OpenClaw Skill？

### 1.1 定义

**Skill** 是 OpenClaw 的能力扩展机制，类似于 VS Code 的插件或 Chrome 的扩展。每个 Skill 封装了一组相关的功能和知识，让 AI Agent 能够执行特定的任务。

### 1.2 核心组成

一个典型的 Skill 包含以下部分：

| 组成部分 | 文件 | 作用 |
|---------|------|------|
| **SKILL.md** | 必需 | 技能定义文件，包含触发条件、使用说明、命令示例 |
| **scripts/** | 可选 | Shell/Python 脚本，实现具体功能 |
| **references/** | 可选 | 参考资料、API 文档等 |

### 1.3 为什么需要 Skills？

> **问题**：AI 模型本身只知道"知道什么"，但不知道"怎么做"。

Skills 解决的就是这个问题：

```
用户说："北京天气"
    ↓
AI 理解意图：查询天气
    ↓
查找匹配的 Skill：weather
    ↓
读取 SKILL.md 获取指令
    ↓
执行：curl wttr.in/Beijing?format=3
    ↓
返回结果：Beijing: ⛅️ +6°C
```

---

## 二、内置 Skills 示例

OpenClaw 自带了丰富的内置 Skills：

### 2.1 常用 Skills 列表

| Skill 名称 | 功能 | 触发词 |
|-----------|------|--------|
| `weather` | 查询天气 | "天气"、"温度"、"forecast" |
| `github` | GitHub 操作 | "PR 状态"、"创建 issue" |
| `gh-issues` | GitHub Issues 处理 | "处理 issues"、"自动修复" |
| `healthcheck` | 系统健康检查 | "安全审计"、"防火墙配置" |
| `canvas` | 画布操作 | "截图"、"渲染 UI" |
| `coding-agent` | 代码助手 | "写代码"、"重构" |
| `skill-creator` | 创建新 Skill | "创建技能"、"新建 skill" |

### 2.2 Skill 文件示例：weather

```markdown
---
name: weather
description: "Get current weather and forecasts via wttr.in"
---

# Weather Skill

## When to Use
- "What's the weather?"
- "Temperature in [city]"
- "Weather forecast for the week"

## Commands

### Local Script (Recommended)
```bash
python3 /home/node/.openclaw/workspace/weather_check.py Shenzhen
```

### API Fallback
```bash
curl "wttr.in/London?format=3"
```
```

---

## 三、实战案例：博客发布 Skill

### 3.1 需求背景

我们希望实现这样的工作流：

```
撰写 Markdown 博客
    ↓
OpenClaw 自动转换为 HTML
    ↓
推送到微信公众号草稿箱
    ↓
返回草稿链接供人工审核
```

### 3.2 mp-draft-push Skill 结构

```
mp-draft-push/
├── SKILL.md           # 技能定义
├── scripts/
│   └── push_to_wechat.sh  # 推送脚本
└── references/
    └── wechat_api.md      # API 文档
```

### 3.3 SKILL.md 核心内容

```markdown
---
name: mp-draft-push
description: "将博客文章发布到微信公众号草稿箱"
---

## 触发词
- "发布文章"
- "推送到微信"
- "发布到公众号"

## 工作流程
1. 读取最新博客文章 (~/.openclaw/workspace/blog/_posts/*.md)
2. 提取标题、摘要、内容
3. 转换为带内联样式的 HTML
4. 上传封面图到微信素材库
5. 创建草稿并返回 media_id

## 执行命令
```bash
LATEST_POST=$(ls -t ~/.openclaw/workspace/blog/_posts/*.md | head -1)
python3 scripts/publish.py "$LATEST_POST"
```

## 重要配置
- WECHAT_APPID: 公众号 AppID
- WECHAT_SECRET: 公众号密钥
- IP 白名单：必须添加服务器 IP
```

### 3.4 实际使用

**用户指令：**
```
推送到微信
```

**OpenClaw 执行过程：**
```
1. 识别意图：发布文章到微信公众号
2. 匹配 Skill：mp-draft-push
3. 读取 SKILL.md 获取工作流程
4. 执行：
   - 查找最新博客：2026-03-17-openclaw-skills.md
   - 提取标题："AI 系列 (6)：OpenClaw Skills 详解"
   - 转换 Markdown → HTML（内联样式）
   - 上传封面图 → 获取 thumb_media_id
   - 调用微信 API 创建草稿
5. 返回结果：
   ✅ 文章已推送到草稿箱
   草稿 ID: xxxxx
   访问链接：https://mp.weixin.qq.com
```

---

## 四、如何创建自己的 Skill？

### 4.1 使用 skill-creator

OpenClaw 提供了 `skill-creator` Skill 来帮助创建新技能：

```
用户：创建一个 skill 用于查询股票价格
OpenClaw：好的，我来帮你创建 stock-query Skill...
```

### 4.2 手动创建步骤

**Step 1：创建目录结构**

```bash
mkdir -p ~/workspace/skills/my-skill
cd ~/workspace/skills/my-skill
```

**Step 2：编写 SKILL.md**

```markdown
---
name: my-skill
description: "技能描述，用于触发匹配"
homepage: https://example.com/docs
metadata:
  openclaw:
    emoji: 🔧
    requires:
      bins: ["curl", "jq"]
---

# My Skill

## When to Use
- 触发词 1
- 触发词 2

## Commands
```bash
# 示例命令
curl -s "https://api.example.com/data" | jq .
```

## Notes
- 注意事项 1
- 注意事项 2
```

**Step 3：添加脚本（可选）**

```bash
mkdir scripts
cat > scripts/run.sh << 'EOF'
#!/bin/bash
echo "执行任务..."
EOF
chmod +x scripts/run.sh
```

**Step 4：测试**

```
用户：[触发词]
OpenClaw：[应该匹配到你的 Skill 并执行]
```

---

## 五、Skill 高级特性

### 5.1 触发优先级

当多个 Skill 可能匹配时，OpenClaw 按以下规则选择：

1. **精确匹配** — 描述中明确提到的触发词
2. **上下文匹配** — 根据对话上下文推断意图
3. **默认行为** — 使用最通用的 Skill

### 5.2 Skill 组合

多个 Skill 可以协作完成复杂任务：

```
用户：把最新的博客文章发布到微信公众号，然后查一下北京天气

执行流程：
1. mp-draft-push: 发布博客
2. weather: 查询天气
```

### 5.3 本地脚本集成

Skill 可以调用本地脚本扩展能力：

```markdown
## Commands

### 使用本地脚本
```bash
python3 /path/to/script.py --city Shenzhen
```

### 直接 API
```bash
curl "wttr.in/Shenzhen"
```
```

我们之前就更新了 `weather` Skill，让它优先使用本地 Python 脚本：

```bash
# 新版：使用本地脚本（更详细的输出）
python3 /home/node/.openclaw/workspace/weather_check.py Shenzhen

# 输出：
# 温度: 22°C (体感 25°C)
# 天气: Partly cloudy
# 湿度: 73%
# 风速: 15 km/h (ESE)
# ...
```

---

## 六、Skills 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenClaw 架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│   │   用户输入   │ ──▶ │  意图识别   │ ──▶ │  Skill 匹配 │      │
│   └─────────────┘     └─────────────┘     └──────┬──────┘      │
│                                                   │              │
│                                                   ▼              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    Skills Registry                       │  │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│   │  │ weather │ │ github  │ │ canvas  │ │mp-draft │ ...   │  │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    Execution Layer                       │  │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│   │  │  exec   │ │ process │ │ browser │ │ message │ ...   │  │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────┐     ┌─────────────┐                         │
│   │  Shell/CLI  │ ──▶ │   返回结果   │ ──▶ 用户                 │
│   └─────────────┘     └─────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、最佳实践

### 7.1 Skill 设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **单一职责** | 每个 Skill 只做一件事 | `weather` 只查天气，不处理其他 |
| **清晰触发** | 触发词要明确，避免歧义 | "天气" → weather，"代码" → coding-agent |
| **文档完善** | 包含使用示例和注意事项 | SKILL.md 中详细说明命令格式 |
| **错误处理** | 提供备选方案 | 本地脚本失败时，回退到 API |
| **安全考虑** | 不暴露敏感信息 | Token 通过环境变量传递 |

### 7.2 常见陷阱

❌ **避免：**

```markdown
## Commands
curl "https://api.example.com?key=SECRET_KEY"
```

✅ **推荐：**

```markdown
## Commands
curl "https://api.example.com?key=${API_KEY}"

## 配置
export API_KEY="your_key_here"
```

---

## 八、ClawHub：Skills 市场

### 8.1 简介

[ClawHub](https://clawhub.ai) 是 OpenClaw 官方的 Skills 市场，类似于 VS Code 的插件市场。

### 8.2 安装方式

**方式一：发送链接给 OpenClaw**

```
安装这个 skill https://clawhub.ai/xxx/mp-draft-push
```

**方式二：手动下载**

```bash
curl -L -o skill.zip "https://clawhub.ai/download?slug=xxx/mp-draft-push"
unzip skill.zip -d ~/workspace/skills/
```

### 8.3 发布自己的 Skill

如果你创建了有用的 Skill，可以分享到 ClawHub：

1. 准备好 Skill 目录
2. 编写 README.md 说明文档
3. 提交到 ClawHub
4. 审核通过后即可被其他用户安装

---

## 九、总结

OpenClaw 的 Skills 机制是其核心特性之一：

| 特点 | 说明 |
|------|------|
| **模块化** | 每个 Skill 封装独立功能，易于管理 |
| **可扩展** | 用户可以创建自己的 Skill |
| **社区驱动** | ClawHub 提供丰富的社区 Skills |
| **知识驱动** | SKILL.md 包含使用指南，AI 自动学习 |
| **脚本集成** | 支持调用本地脚本扩展能力 |

通过博客发布的实际案例，我们看到了 Skills 如何让 AI Agent 从"知道什么"进化到"知道怎么做"。这正是 OpenClaw 区别于普通聊天机器人的关键——它不仅能理解你的意图，还能执行具体的操作。

---

## 十、延伸阅读

- [AI 系列 (1)：Ollama 入门指南]({% post_url 2026-03-12-ollama-guide %})
- [AI 系列 (2)：OpenClaw 本地部署]({% post_url 2026-03-13-local-openclaw-deployment %})
- [AI 系列 (3)：OpenClaw Skills 与微信公众号同步]({% post_url 2026-03-13-openclaw-skills-wechat-sync %})
- [AI 系列 (4)：OpenClaw 安装部署详解]({% post_url 2026-03-14-openclaw-installation %})
- [ClawHub 官网](https://clawhub.ai)
- [OpenClaw 文档](https://docs.openclaw.ai)

---

*发布于 2026 年 3 月 17 日*  
*AI 系列 · 第六篇文章*
