---
layout: post
title: "Docker 环境下 OpenClaw 接入微信教程及错误解决"
date: 2026-03-22
categories: [AI, OpenClaw]
tags: [OpenClaw, Docker, 微信, 接入教程]
author: joyfun
---

## 背景

今天有个好消息：**微信官方支持 OpenClaw 了**！

早上看到新闻就兴冲冲地开始安装，按照官方教程操作，安装倒是成功了，但在登录环节遇到了几个坑。折腾了大半天，终于搞定，把过程记录下来分享给大家。

## 官方安装步骤

### 前置要求

微信需要升级到最新版 **8.0.69**，Play 市场的版本还是 `.67`，需要从其他渠道更新。

### 安装插件

```bash
docker compose run -T --rm openclaw-cli plugins install @tencent-weixin/openclaw-weixin-cli@latest
```

### 登录微信

按照官方教程，运行：

```bash
docker compose run -T --rm openclaw-cli channels login --channel openclaw-weixin
```

然后问题来了...

## 遇到的错误

### 错误 1：resolvePreferredOpenClawTmpDir is not a function

```
[plugins] openclaw-weixin failed to load from /home/node/.openclaw/extensions/openclaw-weixin/index.ts: 
TypeError: (0 , _pluginSdk.resolvePreferredOpenClawTmpDir) is not a function
```

### 错误 2：normalizeAccountId is not a function

修复第一个问题后，又遇到：

```
Channel login failed: TypeError: (0 , _pluginSdk.normalizeAccountId) is not a function
```

## 问题分析

折腾了一上午没搞定，下午静下心来翻了翻 OpenClaw 和插件的代码。

**问题原因：** 腾讯的插件代码引用的 `openclaw/plugin-sdk` 版本比较老，API 已经发生了变化，需要从特定子模块导入。

## 解决方案

### 修复 1：messaging/process-message.ts

先用 grep 找到相关代码：

```bash
grep -r "resolvePreferredOpenClawTmpDir"
```

输出：
```
messaging/process-message.ts:    resolvePreferredOpenClawTmpDir,
messaging/process-message.ts:const MEDIA_OUTBOUND_TEMP_DIR = path.join(resolvePreferredOpenClawTmpDir(), "weixin/media/outbound-temp");
```

修改 `messaging/process-message.ts`：

```typescript
// 修改前
import { resolvePreferredOpenClawTmpDir } from "openclaw/plugin-sdk";

// 修改后
import { createTypingCallbacks } from "openclaw/plugin-sdk/channel-runtime";
import { 
  resolveDirectDmAuthorizationOutcome, 
  resolveSenderCommandAuthorizationWithRuntime 
} from "openclaw/plugin-sdk/command-auth";
```

### 修复 2：auth/accounts.ts 和 channel.ts

找到相关代码：

```bash
grep -r "normalizeAccountId"
```

输出：
```
auth/accounts.ts:import { normalizeAccountId } from "openclaw/plugin-sdk";
auth/accounts.ts:    const id = normalizeAccountId(raw);
channel.ts:import { normalizeAccountId } from "openclaw/plugin-sdk";
channel.ts:    const normalizedId = normalizeAccountId(waitResult.accountId);
channel.ts:    const normalizedId = normalizeAccountId(result.accountId);
```

修改 `auth/accounts.ts` 和 `channel.ts`：

```typescript
// 修改前
import { normalizeAccountId } from "openclaw/plugin-sdk";

// 修改后
import { normalizeAccountId } from "openclaw/plugin-sdk/account-id";
```

### 修复 3：messaging/send.ts

修改 `messaging/send.ts`：

```typescript
// 修改前
import { stripMarkdown } from "openclaw/plugin-sdk";

// 修改后
import { stripMarkdown } from "openclaw/plugin-sdk/bluebubbles";
```

## 完整流程

修改完成后，需要重新构建 Docker 镜像：

```bash
# 拉取最新代码
git pull

# 重新构建镜像
docker build -t openclaw:local . --build-arg OPENCLAW_DOCKER_APT_PACKAGES="python3-pip wget"

# 重启服务
docker compose up -d

# 登录微信
docker compose run -T --rm openclaw-cli channels login --channel openclaw-weixin
```

这次终于成功了！

## 总结

| 错误 | 解决方案 |
|------|----------|
| `resolvePreferredOpenClawTmpDir is not a function` | 从 `openclaw/plugin-sdk/channel-runtime` 导入 |
| `normalizeAccountId is not a function` | 从 `openclaw/plugin-sdk/account-id` 导入 |
| `stripMarkdown` 导入错误 | 从 `openclaw/plugin-sdk/bluebubbles` 导入 |

**教训：**

1. 插件 SDK 版本不匹配时，优先检查导入路径
2. 翻源码比盲目尝试更高效
3. 记得重新构建镜像让修改生效
4. 修复问题要一步步来，每修复一个错误可能暴露下一个

---

希望这篇教程能帮到同样遇到问题的朋友！如果有其他问题，欢迎在评论区交流。
