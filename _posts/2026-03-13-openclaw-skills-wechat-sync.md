---
layout: post
title: "AI 系列 (3)：OpenClaw Skills 与微信公众号自动同步"
date: 2026-03-13 06:10:00 +0800
categories: AI
tags: [OpenClaw, Skills, 微信公众号, 自动化, API]
description: "记录从 ClawHub 安装 mp-draft-push Skill 并配置博客自动同步到微信公众号草稿箱的完整流程，包含踩坑经验与解决方案"
---

## 📚 前言

在 AI 系列 (1) 中，我们介绍了 Ollama 的本地部署。本篇记录如何让博客文章自动同步到微信公众号草稿箱，实现「博客首发 → 公众号分发」的自动化流程。

核心工具是 OpenClaw 的 **Skills 机制**，通过社区贡献的 `mp-draft-push` Skill，我们可以在发布博客的同时，自动将内容推送到微信公众号。

---

## 一、安装 Skills

### 1.1 ClawHub 简介

[ClawHub](https://clawhub.ai) 是 OpenClaw 的官方 Skills 市场，类似于 VS Code 的插件市场。社区开发者可以在这里分享各种技能包，扩展 OpenClaw 的能力。

### 1.2 方式一：发送链接给 OpenClaw

最简单的方式，直接把 ClawHub 的 Skill 链接发给 OpenClaw：

```
安装这个 skill https://clawhub.ai/uncleyannng/mp-draft-push
```

OpenClaw 会自动下载并安装。

### 1.3 方式二：手动下载安装

如果需要手动安装，可以下载 zip 包后解压：

```bash
# 下载 Skill 压缩包
curl -L -o mp-draft-push.zip \
  "https://wry-manatee-359.convex.site/api/v1/download?slug=mp-draft-push"

# 解压到 workspace 的 skills 目录
mkdir -p ~/workspace/skills
unzip -o mp-draft-push.zip -d ~/workspace/skills/
```

安装完成后，OpenClaw 会自动识别并加载新的 Skill。

---

## 二、微信公众平台配置

## 二、微信公众平台配置

登录微信公众平台后台：https://mp.weixin.qq.com

进入「设置与开发」→「基本配置」，获取以下信息：

| 凭证 | 说明 | 示例 |
|------|------|------|
| **AppID** | 公众号唯一标识 | `wxXXXXXXXXXXXXXXXX` |
| **AppSecret** | 公众号密钥 | `your_app_secret_here` |

⚠️ **安全提示**：AppSecret 非常重要，不要泄露给他人！

### 2.2 配置 IP 白名单

微信 API 有安全限制：只有白名单内的 IP 才能调用接口。

**查看服务器 IP：**

```bash
curl -s ifconfig.me
# 输出：1.2.3.4
```

**添加到白名单：**

1. 公众号后台 →「基本配置」→「IP白名单」
2. 点击「修改」
3. 添加服务器 IP：`1.2.3.4`
4. 保存生效

如果不配置白名单，API 调用会返回错误：

```json
{"errcode":40164,"errmsg":"invalid ip 1.2.3.4, not in whitelist"}
```

### 2.3 配置环境变量

将凭证配置为环境变量，供脚本使用：

```bash
# 临时配置（当前终端会话）
export WECHAT_APPID="your_appid_here"
export WECHAT_SECRET="your_appsecret_here"

# 永久配置（添加到 ~/.bashrc 或 Gateway 环境变量）
echo 'export WECHAT_APPID="your_appid_here"' >> ~/.bashrc
echo 'export WECHAT_SECRET="your_appsecret_here"' >> ~/.bashrc
source ~/.bashrc
```

---

## 三、微信 API 调用流程

### 3.1 获取 access_token

每次调用微信 API 都需要 `access_token`，有效期 2 小时。

**请求示例：**

```bash
curl -s "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=${WECHAT_APPID}&secret=${WECHAT_SECRET}"
```

**返回示例：**

```json
{
  "access_token": "ACCESS_TOKEN_HERE",
  "expires_in": 7200
}
```

### 3.2 上传封面图

公众号草稿必须有封面图。我们需要先上传封面到微信素材库，获取 `media_id`。

**下载封面图：**

```bash
# 使用随机图片服务
curl -sL "https://picsum.photos/900/383" -o /tmp/cover.jpg
```

**上传到素材库：**

```bash
curl -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
  -F "media=@/tmp/cover.jpg"
```

**返回示例：**

```json
{
  "media_id": "MEDIA_ID_HERE",
  "url": "http://mmbiz.qpic.cn/sz_mmbiz_jpg/XXXXX/0?wx_fmt=jpeg"
}
```

### 3.3 创建草稿

构建草稿 JSON 并推送。**重要注意事项**：

1. **内容必须使用内联 CSS 样式** - 微信会过滤 `<style>` 标签和外部样式表
2. **必须包含 thumb_media_id** - 即封面图的 `media_id`
3. **标题不超过 64 字节** - 约 21 个中文字符
4. **文章内图片域名限制** - 只能使用 `mmbiz.qpic.cn` 域名的图片

**草稿 JSON 示例：**

```json
{
  "articles": [{
    "title": "AI系列(1)：Ollama入门",
    "author": "joyfun",
    "digest": "从零开始使用Ollama，详解模型选择与配置优化",
    "content": "<section style=\"font-family: -apple-system, sans-serif; line-height: 1.8; color: #333; padding: 15px;\"><h2 style=\"border-bottom: 1px solid #eee;\">前言</h2><p style=\"margin-bottom: 20px;\">文章正文...</p></section>",
    "thumb_media_id": "MEDIA_ID_HERE",
    "need_open_comment": 1,
    "only_fans_can_comment": 0
  }]
}
```

**推送命令：**

```bash
curl -X POST "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=${TOKEN}" \
  -H "Content-Type: application/json" \
  -d @draft.json
```

**成功返回：**

```json
{
  "media_id": "DRAFT_MEDIA_ID_HERE"
}
```

---

## 四、遇到的问题与解决方案

### 问题 1：invalid ip, not in whitelist

**错误信息：**

```json
{"errcode":40164,"errmsg":"invalid ip 14.0.154.54, not in whitelist"}
```

**原因**：服务器 IP 未添加到公众号白名单。

**解决方案**：
1. 查看 IP：`curl -s ifconfig.me`（假设输出 `1.2.3.4`）
2. 公众号后台 →「基本配置」→「IP白名单」→ 添加 IP
3. 保存后立即生效

---

### 问题 2：invalid media_id

**错误信息：**

```json
{"errcode":40007,"errmsg":"invalid media_id"}
```

**原因**：创建草稿时缺少 `thumb_media_id`（封面图）。

**解决方案**：
必须先上传封面图，获取 `media_id`，再创建草稿：

```json
{
  "articles": [{
    "title": "标题",
    "thumb_media_id": "封面图的media_id",  // ← 必填！
    ...
  }]
}
```

---

### 问题 3：缺少 jq 命令

**错误信息：**

```
ERROR: missing command: jq
```

**原因**：`scripts.sh` 依赖 `jq` 解析 JSON，系统未安装。

**解决方案**：

```bash
# Ubuntu/Debian
apt-get install jq

# 或下载二进制
curl -sL https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 -o /usr/local/bin/jq
chmod +x /usr/local/bin/jq
```

---

### 问题 4：样式丢失

**现象**：文章发布后，排版混乱，样式丢失。

**原因**：微信会过滤 `<style>` 标签和外部 CSS。

**解决方案**：使用**内联样式**：

```html
<!-- ❌ 错误：会被过滤 -->
<style>.title { color: red; }</style>
<h1 class="title">标题</h1>

<!-- ✅ 正确：内联样式 -->
<h1 style="color: red; font-size: 24px;">标题</h1>
```

---

## 五、完整同步流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    博客发布同步流程                           │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  1. 写博客   │ ──▶ │ 2. Git Push  │ ──▶ │ 3. 博客发布   │
│  (Markdown)  │     │  到 GitHub   │     │ (Pages 托管)  │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 6. 提示用户  │ ◀── │ 5. 创建草稿  │ ◀── │ 4. 获取      │
│ (后台检查)   │     │ (POST API)   │     │ access_token │
└──────────────┘     └──────────────┘     └──────────────┘
                            ▲
                            │
                     ┌──────┴───────┐
                     │ 7. 上传封面  │
                     │ (获取 media_id) │
                     └──────────────┘
```

**流程说明：**

| 步骤 | 操作 | 输出 |
|------|------|------|
| 1 | 撰写 Markdown 博客 | `.md` 文件 |
| 2 | Git commit & push | 推送到 GitHub |
| 3 | GitHub Pages 构建 | 博客上线 |
| 4 | 调用微信 API | 获取 `access_token` |
| 5 | 上传封面图 | 获取 `thumb_media_id` |
| 6 | 创建草稿 | 草稿保存到公众号后台 |
| 7 | 提示用户检查 | 完成 |

---

## 六、后续优化方向

### 已实现 ✅

- 博客发布后自动同步到公众号草稿箱
- 封面图自动上传
- 内联样式转换
- IP 白名单自动检测

### 待优化 📋

| 功能 | 说明 | 优先级 |
|------|------|--------|
| **图片处理** | 文章内图片需上传到微信素材库，替换为 `mmbiz.qpic.cn` URL | 高 |
| **标题长度检查** | 微信限制 64 字节，需自动截断或提示 | 中 |
| **定时发布** | 结合 cron 实现定时推送 | 中 |
| **多平台同步** | 知乎、小红书、头条等平台适配 | 低 |
| **错误重试** | API 调用失败时自动重试 | 中 |
| **缓存 Token** | `access_token` 有效期 2 小时，可缓存复用 | 中 |

---

## 七、总结

通过 OpenClaw 的 Skills 机制，我们实现了博客到微信公众号的自动同步，核心要点：

1. **Skills 扩展** - ClawHub 提供丰富的社区 Skills，安装即用
2. **API 对接** - 微信公众号 API 文档完善，调用简单
3. **踩坑记录** - IP 白名单、封面图、内联样式是主要坑点

下一篇将介绍如何用 OpenClaw + Ollama 构建 AI Agent，实现更复杂的自动化任务。

---

## 附录：常用 API 端点

| API | 方法 | 说明 |
|-----|------|------|
| `/cgi-bin/token` | GET | 获取 access_token |
| `/cgi-bin/material/add_material` | POST | 上传素材到素材库 |
| `/cgi-bin/draft/add` | POST | 创建草稿 |
| `/cgi-bin/draft/get` | GET | 获取草稿详情 |
| `/cgi-bin/freepublish/submit` | POST | 发布草稿 |

---

*发布于 2026 年 3 月 13 日*
*AI 系列 · 第三篇文章*
