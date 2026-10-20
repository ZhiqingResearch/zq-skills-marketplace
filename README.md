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
| [zq-amazon-product-video](skills/zq-amazon-product-video/) | 2.0.0 | 付费（按平台实际费用结算） | 当用户提供商品图片、名称和卖点，希望制作 Amazon 商品广告视频或对成片质检返修时使用。根据已确认的商品事实与创意生成视频，按可用通道进行补图、分析和可选质检。 |
| [zq-asin-lookup](skills/zq-asin-lookup/) | 0.1.6 | 付费（按平台实际费用结算） | 当用户给出 Amazon ASIN 需要查询商品公开信息（标题、品牌、卖点、图片、价格、评分等），用于选品核对或作为后续 Listing、视频任务的商品事实来源时使用。 |
| [zq-config](skills/zq-config/) | 1.0.6 | 免费 | 当用户首次配置、检查、更换或删除 zq-skills 平台 KeyB，或排查 API 凭据问题时使用。支持本地 REST 凭据和 MCP 连接鉴权。 |
| [zq-listing](skills/zq-listing/) | 0.6.0 | 付费（按平台实际费用结算） | 当用户要为 Amazon、Walmart、eBay、Ozon、Wildberries 或 TikTok 市场生成商品 Listing 文案，或对已有 Listing 独立评分和获得修改建议时使用。 |
| [zq-update](skills/zq-update/) | 1.1.4 | 免费 | 当用户检查或升级已安装的 zq-skills 客户端剧本时使用。读取本地版本标记，并与市场发布清单或用户提供的安装包版本比较。 |
| [zq-video-understanding](skills/zq-video-understanding/) | 2.2.9 | 付费（按平台实际费用结算） | 上传用户视频到平台，异步生成结构化视觉分析，并由平台脚本提取 6 至 10 张真实关键帧，说明模型选点或平台补选来源。 |

平台能力在云端执行；计费方式见上表和各技能说明，实际费用以任务回执和 SellerOS 积分账单为准。安装后先在 agent 对话里配置平台 KeyB（直接说"帮我配置 zq-skills API key"，`zq-config` 会引导完成）。

---
同步自 release `20261020-0529-a7ae687`（commit `a7ae687`，2026-10-20T05:29:09.994Z）。
