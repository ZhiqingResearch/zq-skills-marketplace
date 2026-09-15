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
| [zq-amazon-product-video](skills/zq-amazon-product-video/) | 1.1.1 | 付费（按平台实际费用结算） | 当用户提供商品图片、名称和卖点，希望制作 Amazon 商品广告视频或对成片质检返修时使用。通过平台分析、补图、生成和质检能力完成。 |
| [zq-config](skills/zq-config/) | 1.0.1 | 免费 | 当用户首次配置、检查、更换或删除 zq-skills 平台 KeyB，或排查 API 凭据问题时使用。支持本地 REST 凭据和 MCP 连接鉴权。 |
| [zq-listing](skills/zq-listing/) | 0.4.0 | 付费（按平台实际费用结算） | 当用户要为 Amazon、Walmart、eBay、Ozon、Wildberries 或 TikTok 市场生成商品 Listing 文案，或对已有 Listing 独立评分和获得修改建议时使用。 |
| [zq-update](skills/zq-update/) | 1.0.0 | 免费 | 当用户检查或升级已安装的 zq-skills 客户端剧本时使用。读取本地版本标记，并与市场发布清单或用户提供的安装包版本比较。 |
| [zq-video-understanding](skills/zq-video-understanding/) | 2.1.1 | 付费（按平台实际费用结算） | 上传用户视频到平台，异步生成结构化视觉分析，并由平台脚本按模型选出的时间点提取 6 至 10 张真实关键帧。 |

平台能力在云端执行；收费与可用性以对应部署为准：operator-funded 表示运营方承担成本，selleros 表示经 SellerOS 按平台实际费用（网关 CNY 账单）逐笔扣积分，不预设单价。安装后先在 agent 对话里配置平台 KeyB（直接说"帮我配置 zq-skills API key"，`zq-config` 会引导完成）。

---
同步自 release `20260915-0202-fa469a8`（commit `fa469a8`，2026-09-15T02:02:37.279Z）。
