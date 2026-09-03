---
name: zq-update
version: 0.1.2
description: 检查已安装的 zq-skills 系列 skill 是否有新版本：扫描本地安装的版本标记，与公共分发仓库最新清单比对，并指引通过 Skill 市场或 skills CLI 更新重装。当用户想检查 skill 更新、升级已安装的 skill，或付费 skill 报 409 版本过旧时使用。
---
<!-- zq-skills: zq-update v0.1.2 target=free -->

# zq-update — skill 更新检查助手

检查已安装的 zq-skills 系列 skill（付费编排型 + 免费）是否有新版本，
并指引更新。免费 skill，本地扫描 + 比对公共分发仓库清单，全程不需要
API key、不请求平台 API、不产生计费。

## 何时使用 / 何时不使用

- 使用：用户想检查已装 skill 是否有新版；付费 skill 报 409（版本过旧，
  低于 min_client_version 被平台拒绝）；用户想知道更新了什么；
- 不使用：配置 API key（→ `zq-config`）；安装全新 skill（→ Skill 市场）；
  skill 执行报错排查（与本 skill 无关，看对应剧本的出错处理章节）。

## 第一步：扫描已安装版本

已安装的编排型 skill 文件头部带机器可读版本标记：

```
<!-- zq-skills: <skill-id> v<版本> target=<目标产品> -->
```

桌面端：在本产品存放 skill 的目录里搜标记（路径按产品适配，如
`~/.claude/skills/`、Codex/Cursor 各自的 skill 目录）：

```sh
grep -rn -E '<!-- zq-skills: [a-z0-9-]+ v[0-9.]+' <skill 目录> 2>/dev/null
```

网页端（粘贴型，无本地目录）：向用户说明当前会话内没有可扫描的安装
目录，直接进入"更新方式"按市场最新版重新粘贴。

汇总列出：skill id、已装版本、目标产品（付费为 claude-code/codex/
cursor/paste，免费为 free）。免费 skill（zq-config 等）同样带版本
标记，一并纳入检查。

## 第二步：查询最新版本（可选，更准确）

从公共分发仓库拉取机器可读的最新版本清单——不需要 API key，不请求
平台 API，不产生任何计费：

```sh
curl -s https://raw.githubusercontent.com/ZhiqingResearch/zq-skills-marketplace/main/.release.json
```

读取 `skills[].id` 与 `skills[].version`，与第一步的本地扫描结果比对。
该快照由发版流水线自动同步（`release` 字段为来源发版号），与市场发布
同源。raw 地址不可达时降级：解析同一仓库 README 的 Skills 版本表，或
让用户打开 `https://github.com/ZhiqingResearch/zq-skills-marketplace`
目视比对。

## 第三步：比对与更新指引

| 比对结果 | 处理 |
| --- | --- |
| 已装 = 最新 | 告知用户已是最新版，无需操作 |
| 已装 < 最新 | 指引到 Skill 市场该 skill 页重新一键安装（覆盖旧版）；说明新版版本号 |
| 平台返回 409 场景 | 解释原因：剧本版本低于该 skill 的 min_client_version，平台拒绝执行是防止旧剧本配新平台出错；市场重装后即可解除 |
| 清单里没有该 skill | 可能已下线或更名：引导用户到市场确认 |

更新方式（当前阶段，按用户环境选一）：

1. **市场重装（推荐，全端通用）**：市场 skill 页选同一目标产品重新
   一键安装，新文件覆盖旧版（同目录同名覆盖）；
2. **skills CLI（桌面端）**：`npx skills add ZhiqingResearch/zq-skills-marketplace@<skill-id>`
   重新安装单个 skill 覆盖旧版（与市场同源，都来自分发仓库）；
3. 网页端：市场复制最新粘贴块，替换旧提示词保存。

更新完成后建议重新触发一次该 skill 验证正常（首次运行仍会做 key
检查，属正常流程）。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| raw 清单拉取失败 | 重试一次；仍失败降级用 README 版本表或仓库页面目视比对 |
| 清单 JSON 解析失败 | 分发仓库同步中或格式变化；改看 README 版本表 |
| 扫不到版本标记 | 该 skill 可能不是 zq-skills 系列；如实告知 |
