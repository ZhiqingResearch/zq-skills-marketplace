---
name: zq-update
version: 1.0.0
description: 当用户检查或升级已安装的 zq-skills 客户端剧本时使用。读取本地版本标记，并与市场发布清单或用户提供的安装包版本比较。
---
<!-- zq-skills: zq-update v1.0.0 target=free -->

# zq-update — 客户端技能更新

扫描本产品技能目录内的机器可读标记，汇总 id、版本、target：

```text
<!-- zq-skills: <skill-id> v<版本> target=<目标产品> -->
```

如目录不可访问或网页端只有粘贴剧本，说明无法扫描；从用户提供的安装包或
市场安装页确认当前版本。不能仅按目录名断定版本。

## 查询公共发布清单

无需 KeyB，读取公共分发仓库的机器可读清单：

```sh
curl --fail --silent --show-error https://raw.githubusercontent.com/ZhiqingResearch/zq-skills-marketplace/main/.release.json
```

读取 skills[].id、skills[].version 与来源 release。网络失败最多重试一次，
再用同一公共仓库 README 的版本表或市场发布页核对，不改用运行时 API。

## 比较版本

以市场正式发布清单或已校验安装包的版本为依据；预发包仅在用户选择预发环境
时使用。读不到发布源就说明“尚未确认是否最新”，不要编造新版本。

MVP 的 `GET /api/v1/skills` / MCP `list_skills` 返回运行时能力目录和 inputSchema，
不提供客户端安装版本、min_client_version 或安装包。MVP 没有旧版 `/v1/skills`
及 `/v1/skills/{id}/client` 分发接口；不要把运行时 slug 与本地技能 id 强行匹配。
409 表示幂等冲突等请求问题，不能单凭状态码判定客户端过旧。

## 更新

用户请求升级时，从其选择的市场版本下载安装，保留凭据文件。替换前比较新旧
版本并保留可恢复副本，安装后重新读取版本标记确认；不要通过执行模型任务验证更新。
本地开发可由仓库维护者用安装脚本指定 API origin 与目标目录重新渲染，已有目录用
`--force` 更新。脚本属于源码仓库，不假设它存在于用户已安装的技能中。

网页端使用市场最新粘贴块替换旧提示词；版本无法比对时明确说明。
需要设置 KeyB 时转到 `zq-config`。
