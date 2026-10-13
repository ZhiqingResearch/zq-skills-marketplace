---
name: zq-config
version: 1.0.6
description: 当用户首次配置、检查、更换或删除 zq-skills 平台 KeyB，或排查 API 凭据问题时使用。支持本地 REST 凭据和 MCP 连接鉴权。
---
<!-- zq-skills: zq-config v1.0.6 target=free -->

## 平台连接

所有平台 REST 请求固定使用 `https://skills-platform-api.zhiqingresearch.com`，只需配置 KeyB。


# zq-config — API key 配置助手

免费配置助手；检查凭据只做只读发现，不创建模型任务。支持 REST 与 MCP 连接鉴权。

## 凭据与地址

- 使用接入项目为当前用户签发的 KeyB（通常 `zk-` 开头，兼容旧 `sk_`）；
  不按前缀自行判定有效性，以服务端验证为准。KeyA 仅由项目方管理。
- REST：读取安全注入的 `ZQ_API_KEY` 环境变量，缺项再读
  `~/.config/zq-skills/credentials`。逐行按第一个 `=` 分割；不要 source/eval 文件。
- 所有 REST 请求固定使用生产地址 `https://skills-platform-api.zhiqingresearch.com`。
- MCP：KeyB 由客户端安全的连接设置放在 `/mcp` 的 `Authorization: Bearer <KeyB>`；
  平台 API 地址与 MCP 服务地址可能不同。不得把 KeyB 填进模型工具参数。

凭据文件示意（实际值由用户安全输入，不把真实 KeyB 写进对话、代码或命令参数）：

```text
ZQ_API_KEY=<用户 KeyB>
```

## set / rotate

1. 缺少 KeyB 时引导在本地受控输入或客户端凭据设置中填写，
   不要求粘贴到公开对话；无需提供 API 地址。
2. 创建 `~/.config/zq-skills`，目录权限 700，凭据文件权限 600。
   只保存或更新 KeyB；以临时文件加原子替换写入，
   不输出完整内容，保留其他无关字段。
3. 执行 status。轮换由项目方签发新 KeyB；同手机号重新签发会使旧 KeyB 失效。
   本助手只替换本地/连接凭据，不自行持有 KeyA 或调用签发接口。

## status

REST 携带 `Authorization: Bearer <KeyB>` 调用：

```http
GET https://skills-platform-api.zhiqingresearch.com/api/v1/skills
```

HTTP 200 的 `data` 是技能数组；空数组也表示本次鉴权通过，不代表没有权限。
只展示脱敏 key、地址、连接结果。可见技能不保证模型任务已启用。
MCP 使用 `list_skills` 做同样检查。不要同时尝试另一通道来绕过权限或重复执行。

能力发现不返回积分余额；不能据发现结果声称有积分、可扣费或免费使用模型。
费用以具体任务的 `billed` 回执和 SellerOS 积分账单为准。业务调用返回
`billing_unavailable` 时联系项目方处理；配置助手不修改服务端计费属性。

## remove

删除只作用于用户指定的凭据来源：本地 credentials 文件、当前会话的 ZQ_API_KEY 或 MCP 连接凭据分别处理。删除文件后仍可能有其他活动来源；检查来源是否仍存在，只报告脱敏状态，不能声称必然无法调用。删除字段时保留其他字段；本地删除不等于服务端吊销，吊销由项目方处理。

## 无本地凭据文件的环境

优先使用客户端的安全连接/会话密钥设置。临时环境变量仅用于当前会话；
不要把真实 KeyB 写入项目、日志或交付物，也不宣称对话记录会自动删除。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 | 检查 KeyB、项目和用户状态；必要时由项目方重新签发 |
| 404 | 检查 API origin 和部署版本，不猜测余额或安装接口 |
| 5xx / 网络 | 只读 status 可间隔重试最多两次；仍失败说明连接尚未验证 |
| 文件权限异常 | 修正为目录 700、文件 600；不输出文件内容 |
