---
name: zq-listing
description: 当用户要为 Amazon、Walmart、eBay、Ozon、Wildberries 或 TikTok 市场生成商品 Listing 文案，或对已有 Listing 独立评分和获得修改建议时使用。
---
<!-- zq-skills: zq-listing v0.4.0 target=claude-code -->

# 多平台 Listing 生成与评分

## 数据展示红线（最高优先级，覆盖本文件其他展示要求）

禁止向用户展示真实模型费用：平台回执中的 `billed.charges[].costCny`、任何
CNY/人民币金额、上游单价、按 token 用量折算的成本，以及由此推算的加价或
利润，都属于平台内部信息，不得出现在面向用户的回复、交付文件或日志中。
用户可见的计费信息只有积分口径（`creditsCharged`/"已扣 N 积分"）与
`billed.state`；被问及费用明细时回答"以 SellerOS 积分账单为准"。

## 何时使用 / 何时不使用

生成指定市场商品文案，或独立评估用户现有 Listing。生成不会自动评分；只要求
评分时直接评估现有稿，包括缺字段的草稿。用户要求生成并评估时按两项任务执行。
不自动发布商品、不管理店铺，不把平台评分说成上架或合规保证。

## API key 配置

使用项目为当前用户签发的 KeyB（通常 `zk-`，兼容旧 `sk_`）。REST 优先从安全
环境变量 `ZQ_API_KEY` / `ZQ_API_BASE` 读取，缺项再读
`~/.config/zq-skills/credentials`；按首个 `=` 分割，不 source/eval。
API origin 缺省用 `http://skills-platform-api-uat.zhiqingresearch.com`，不另加 `/api` 或 `/v1` 后缀。
带 `Authorization: Bearer <KeyB>` 调用 `GET /api/v1/skills` 可验证鉴权；
200 的空 `data` 数组也有效。缺凭据引导 `zq-config`，不索取 KeyA，不回显 KeyB。

如已连接平台 MCP，使用 `generate_listing` / `get_listing_generation`、
`score_listing` / `get_listing_score`。KeyB 仅在连接头中设置；MCP 创建参数是
`{input:<下述REST body>,idempotency_key:<键>,wait_seconds:<可选>}`。
查询分别传 `generation_id` 或 `score_id`，可设置 `wait_seconds`（0–20 秒）。
MCP 响应使用顶层 generation_id / score_id、status、result、issues，没有 REST 的
`data` 包装；processing 表示 queued/processing，failed/cancelled 是终态失败；调用错误读取 error_code，unknown 表示受理未知。
下文 `data.*` 字段专用于 REST，MCP 从顶层 result 交付。
REST 与 MCP 二选一，不切通道重复创建。

MCP 创建工具重放已终态任务时，可能只返回状态回执、`result=null` 和查询
`next_action`。这不表示没有产物；按原任务 ID 调用对应查询工具取得结果或失败
原因，不重新创建。只有查询详情后才能判断交付内容。

## 物料与开始前提问

先确定动作、目标市场及缺失的必要信息，已提供的不重复询问。

```yaml
version: 1
questions:
  - id: action
    ask: 需要生成 Listing、评估已有文案，还是生成后评分？
    type: choice
    choices: [生成, 评分, 生成后评分]
    required: true
  - id: market
    ask: 目标平台与国家市场是什么？
    type: choice
    choices: [amazon_us, amazon_uk, amazon_de, amazon_jp, walmart_us, ebay_us, ozon_ru, wildberries_ru, tiktok_us, tiktok_mx]
    required: true
  - id: product
    ask: 生成时请提供商品名称、类目及已确认的事实或属性；评分时可选提供独立事实资料。
    type: text
    required: false
  - id: listing
    ask: 若需评分，请提供现有 Listing 的字段或原稿。
    type: text
    required: false
  - id: feedback_language
    ask: 反馈使用中文还是英文？
    type: choice
    choices: [zh-CN, en-US]
    default: zh-CN
    required: false
```

| 市场值 | 平台 | 买家文案语言 |
| --- | --- | --- |
| amazon_us / amazon_uk | Amazon | en-US / en-GB |
| amazon_de / amazon_jp | Amazon | de-DE / ja-JP |
| walmart_us / ebay_us | Walmart / eBay | en-US |
| ozon_ru / wildberries_ru | Ozon / Wildberries | ru-RU |
| tiktok_us / tiktok_mx | TikTok | en-US / es-MX |

`feedbackLanguage` 只决定反馈语言（zh-CN 默认，或 en-US），不覆盖市场文案语言。
不要猜未支持的市场。核心事实或市场缺失时先补齐；参考稿不是商品事实证据。

## 执行流程

每项创建生成一个 8–128 字符稳定 `Idempotency-Key`（UUID 可用）。保存原键、原
完整 JSON 和任务 ID。省略字段与显式填入默认值不等价，恢复时不要补默认值或
重写输入。两项 POST body 都不超过 64 KiB；POST/GET 均不带查询参数。

本版对应候选 MVP。收费模式以平台响应为准：默认（disabled）拒绝模型创建；
`operator-funded` 项目允许时执行、运营方承担成本；`selleros` 模式按网关实际
CNY 费用逐笔经 SellerOS 扣积分（受理 `billed={state:"pending"}`，扣费确认前
查询返回 503 `billing_pending` / `billing_blocked` 不交付产物——稍后重查原
任务即可，不换键重新生成；终态查询透传 `billed` 回执，展示仅限积分口径——见顶部数据展示红线）。不据此承诺固定
积分或零成本。无余额查询接口。

生成查询可能降级交付：模型输出未通过严格校验时 status=completed、result
按原样交付，issues 逐项列出未通过规则（severity=error 须按 suggestion 修正
后再发布，如压缩超限标题、改写禁用词——禁用词匹配不区分否定句）。每个任务
恰好一次模型调用、一笔扣费，平台不会自动修复重试。是否重新生成由用户决定：
先展示 issues 与修正建议并说明重新生成会再次扣积分，用户同意后才用新幂等键
发起；未经确认不自动重发。只有 issues 为空的 completed 结果可直接发布。

### 生成

```http
POST http://skills-platform-api-uat.zhiqingresearch.com/api/v1/listings/generations
Authorization: Bearer <KeyB>
Idempotency-Key: <稳定键>
Content-Type: application/json

{
  "market": "amazon_us",
  "product": {
    "name": "Travel Mug",
    "category": "Drinkware",
    "facts": ["Capacity 500 ml", "Stainless steel body"]
  }
}
```

替换示例为用户事实。`product.name/category` 必填，`facts` 非空字符串数组或
`attributes` 非空 `{name,value}` 数组至少提供一项；可选 brand、variations
`{theme,values}`。可选 `keywords` 为 `{term,searchVolume?,period?,source?}` 数组；
提供 searchVolume 必须提供 period（week/month），缺搜索数据不要编造。
还可传 `referenceContent`、`constraints:{requiredTerms?,forbiddenTerms?}`。

选项随市场变化，不使用通用选项对象填所有平台：

- Amazon 默认 standard，可选 `options:{mode:"standard",includeQA,includeLongTitle}`。
  custom 仅用于电脑类商品，需提供 `mode:"custom"`、ownBrand、shopName、oemModel、
  非空字符串数组 ram/rom，以及商品事实中的保修信息；缺事实先问用户，不自行补造。
- Ozon / Wildberries 可选 `options:{includeTitleVariants:true}`。
- Walmart / eBay / TikTok 不发送 options。

HTTP 202 为 `{code,message,data,timestamp}`，任务 ID 在 `data.id`（`lgen_` 前缀），
另有 `data.status/replayed`。GET `/api/v1/listings/generations/{id}` 查询。
完成后读取 `data.result.listing`、`warnings`、`rulesVersion`。

### 独立评分

```http
POST http://skills-platform-api-uat.zhiqingresearch.com/api/v1/listings/scores
Authorization: Bearer <KeyB>
Idempotency-Key: <评分自己的稳定键>
Content-Type: application/json

{
  "market": "amazon_us",
  "listing": {"platform": "amazon", "title": "Travel Mug 500 ml"}
}
```

`listing.platform` 与 market 对应，至少含一项实质文案内容。缺标题、要点等可以
进入评分，不要擅自补全差稿后再评分。可选 `productFacts` 与生成 product 同结构，
由用户独立提供；不能把生成稿或待评稿本身当作事实证据。

已有生成结果可直接将其 `listing` 对象用于评分。对于用户原稿，保留字段结构；
常用文本字段如下，额外复杂字段仅在确知其类型时提交：

| platform | 常用字段 |
| --- | --- |
| amazon | title、description 字符串；bullets、itemHighlights、searchTerms 字符串数组 |
| walmart | title、description；keyFeatures 数组 |
| ebay | title、description；itemSpecifics 为 name/value 数组 |
| ozon | title、description；highlights、hashtags 数组 |
| wildberries | title、description；highlights、searchClusters、sizes 数组 |
| tiktok | title、shortDescription；productHighlights、searchTerms 数组 |

评分只在 Amazon 接受 `options.mode`（standard/custom）和 custom 的 ownBrand；
若 listing 也含 mode/ownBrand，两处必须一致。其他平台评分不发 options。
HTTP 202 的 `data.id` 为 `lscore_` 前缀；GET `/api/v1/listings/scores/{id}` 查询。

### 查询

每隔至少 5 秒查询，`data.status=queued/processing` 等待，completed 才读 result；
failed 读取 `data.error` 并停止。未完成时 result 为 null。
约 10 分钟未结束则保存 ID，告知可继续查询，不新建任务。REST 不加 wait 参数。
MVP 默认任务期限 15 分钟；超时、中断或客户端停止等待不证明上游没执行。

## 交付

生成时交付市场对应的 Listing 字段、warnings、rulesVersion 和任务 ID，不省略
缺少事实或合规风险提示。评分时交付 total、rating、dimensions、issues、suggestions、
assessmentCoverage 与 rubricVersion；保留未评估项和覆盖率，不能把未评估视为通过。

用户只要其中一项时不要自动增加另一项模型任务。反馈使用用户选择的语言，
商品文案保留目标市场语言。不复制平台内部评分规则、权重或提示词。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 / 404 | 核对 KeyB、origin、任务 ID 和所属用户，不通过换用户查询 |
| 409 idempotency_conflict | 核对原键原完整 body，不自动换键 |
| 413 / 422 | 按 error.fieldErrors 的 path/code 修正体积或字段；不解释私有规则 |
| 429 | 等待容量释放后原键原参数有限重试，不并行换键 |
| billing_unavailable / skill_unavailable | 联系项目方，停止创建，不修改项目计费属性绕过 |
| billing_pending / billing_blocked（查询时，selleros 模式） | 扣费未确认或被拒，不交付产物：稍后重查原任务；被拒时联系项目方处理原账单，不换键重新生成 |
| admission_timeout / submission_unknown / POST 网络或 5xx | 有 ID 查询原任务；无 ID 保存原键原完整请求，用户明确要求恢复后才原样重放 |
| TASK_TIMEOUT / TASK_INTERRUPTED / TASK_FAILED，以及生成输出完全不可解析时的 MODEL_OUTPUT_INVALID | 停止；交付任务 ID 和可行动错误，不自动新建或声称无成本；生成校验失败优先以 completed+issues 降级交付，不走此行 |
| completed 但 issues 含 severity=error（生成降级交付） | 非失败：交付 result 内容并逐条转述 issues 与 suggestion；是否重新生成（会再次扣积分、用新幂等键）必须先征得用户同意 |
| delivery_unavailable / GET 网络或 5xx | 间隔重试最多 2 次，仍失败保留原任务 ID，不能重新生成替代查询 |
