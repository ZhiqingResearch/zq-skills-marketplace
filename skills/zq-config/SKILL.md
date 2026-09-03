---
name: zq-config
version: 0.1.2
description: 配置、校验、轮换或删除 zq-skills 平台 API key（写入本地凭据文件、验证有效性、查询积分余额），并提供沙箱环境的一次性 key 使用指引。当用户要设置、检查、更换 zq-skills API key，查看积分余额，或排查 key 问题时使用。
---
<!-- zq-skills: zq-config v0.1.2 target=free -->

# zq-config — API key 配置助手

管理 zq-skills 平台的 API key 与本地凭据。所有付费 skill 运行前都依赖
这份凭据，本 skill 是标准配置入口；免费 skill，本地执行，不产生计费。

## 何时使用 / 何时不使用

- 使用：首次配置 key、检查 key 是否有效、查看积分余额、更换或吊销
  旧 key、删除本地凭据、了解沙箱环境怎么用 key；
- 不使用：与 key 无关的问题（如 skill 执行失败的原因排查、模板处理）。

## 本地凭据

- 位置：`~/.config/zq-skills/credentials`（权限 600，属主本人）；
- 格式：

```
ZQ_API_KEY=sk-xxxxxxxxxxxxxxxx
ZQ_API_BASE=https://skills-platform-api-dev.zhiqingresearch.com
```

- `ZQ_API_BASE` 可选；默认地址以 Skill 市场安装页展示的官方地址为准；
- 本 skill 与所有付费 skill 都从这里读取；文件不存在时进入"配置流程"。

## 功能

### 1. status（检查 / 查余额）

读取凭据文件 → 展示脱敏 key（如 `sk-****abcd`）→ 验证：

```sh
curl -s -w "\nHTTP %{http_code}\n" \
  -H "Authorization: Bearer $(grep '^ZQ_API_KEY=' ~/.config/zq-skills/credentials | cut -d= -f2)" \
  "$(grep '^ZQ_API_BASE=' ~/.config/zq-skills/credentials | cut -d= -f2 || echo https://skills-platform-api-dev.zhiqingresearch.com)/v1/account/balance"
```

200 → 展示余额；401 → 提示进入 rotate；网络错误 → 检查
`ZQ_API_BASE` 或稍后再试。

### 2. set（配置 / 首次设置）

1. 引导用户到 **Skill 市场账号页**领取 API key（`sk-` 或 `sk_` 开头）；
2. 写入凭据文件：

```sh
mkdir -p ~/.config/zq-skills
printf 'ZQ_API_KEY=%s\n' "<key>" > ~/.config/zq-skills/credentials
chmod 600 ~/.config/zq-skills/credentials
```

3. 立即执行 status 验证；通过后告知用户付费 skill 已可用。

### 3. rotate（轮换）

1. 引导用户在市场账号页**生成新 key 并吊销旧 key**；
2. 按 set 流程覆盖写入 → status 验证；
3. 提醒：怀疑 key 泄露（例如曾在网页会话中粘贴过）时应立即轮换。

### 4. remove（删除）

向用户确认后删除 `~/.config/zq-skills/credentials`，并提醒已装的
付费 skill 在重新配置前无法使用。

## 沙箱环境（网页 agent）指引

- 沙箱读不到本地文件：key 由用户在会话中提供**一次**，保存在会话
  内存/环境变量中供本次任务使用，会话结束即失效；
- 粘贴进会话的 key 会留在第三方对话记录里——重要任务完成后建议到
  市场账号页轮换；
- 任何情况下不得把完整 key 回显到对话输出、写入代码或仓库。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 | key 无效或已吊销 → 走 rotate 流程 |
| 403 | key 被禁用 → 指引联系市场客服 |
| 网络错误 | 检查 ZQ_API_BASE 是否写错；稍后重试 |
| 文件权限异常 | 重新 chmod 600；仍异常则 remove 后重走 set |
