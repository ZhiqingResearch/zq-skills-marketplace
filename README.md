# zq-skills

智清 zq-skills 平台的公共 skill 分发仓库，支持 [skills CLI](https://github.com/vercel-labs/skills) 一键安装到 Claude Code、Codex、Cursor 等 70+ agent。

> 本仓库由 zq-skills 私有源仓库的发布流水线自动同步生成（快照见 `.release.json`），请勿手工提交。

## 安装

全部安装：

```bash
npx skills add ZhiqingResearch/zq-skills-marketplace
```

只安装单个 skill（`@` 过滤）：

```bash
npx skills add ZhiqingResearch/zq-skills-marketplace@zq-video-understanding
```

## Skills

| skill | 版本 | 类型 | 说明 |
| --- | --- | --- | --- |
| [zq-amazon-product-video](skills/zq-amazon-product-video/) | 0.1.0 | 付费（计费详见平台） | 根据商品图片与资料生成与真实商品结构一致的 Amazon 商品广告视频：平台完成素材分析、缺失视角补图、多图参考视频生成与商品一致性质检，剧本在会话内组织素材、事实、提示词、成片四个确认门并渐进返修。当用户要为商品制作广告视频、商品页视频素材时使用。 |
| [zq-config](skills/zq-config/) | 0.1.2 | 免费 | 配置、校验、轮换或删除 zq-skills 平台 API key（写入本地凭据文件、验证有效性、查询积分余额），并提供沙箱环境的一次性 key 使用指引。当用户要设置、检查、更换 zq-skills API key，查看积分余额，或排查 key 问题时使用。 |
| [zq-update](skills/zq-update/) | 0.1.1 | 免费 | 检查已安装的 zq-skills 系列 skill 是否有新版本：扫描本地安装的版本标记，与平台最新清单比对，并指引通过 Skill 市场更新重装。当用户想检查 skill 更新、升级已安装的 skill，或付费 skill 报 409 版本过旧时使用。 |
| [zq-video-understanding](skills/zq-video-understanding/) | 0.2.1 | 付费（计费详见平台） | 对用户提供的视频做多模态理解（画面、对白、声音联合分析），交付结构化分析（摘要、关键词、人物、物体、场景时间轴）、按时间戳从原视频提取的代表性截图（ZIP 打包）与本次用量费用摘要。当用户给出视频文件、要求理解/分析/总结视频内容或提取关键画面时使用。 |

付费 skill 的执行与计费在 zq-skills 平台云端完成；安装后先在 agent 对话里配置平台 API key（直接说"帮我配置 zq-skills API key"，`zq-config` 会引导完成）。

---
同步自 release `20260903-1449-a510390`（commit `a510390`，2026-09-03T05:49:37.335Z）。
