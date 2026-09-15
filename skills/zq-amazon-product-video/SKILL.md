---
name: zq-amazon-product-video
description: 当用户提供商品图片、名称和卖点，希望制作 Amazon 商品广告视频或对成片质检返修时使用。通过平台分析、补图、生成和质检能力完成。
---
<!-- zq-skills: zq-amazon-product-video v1.3.0 target=claude-code -->

# Amazon 商品广告视频


## 何时使用 / 何时不使用

用于商品广告视频生成和基于质检结果的返修；纯剪辑、视频理解或 Listing 文案
分别使用对应工具。本剧本对齐 MCP / Skills API MVP，服务是否已部署以项目方为准。

## API key 配置

REST 使用当前用户 KeyB（通常 `zk-`，兼容旧 `sk_`）。优先读取安全注入的
`ZQ_API_KEY` / `ZQ_API_BASE`，缺项再读取 `~/.config/zq-skills/credentials`，
按首个 `=` 分割，不 source/eval。API origin 缺省用 `http://skills-platform-api-uat.zhiqingresearch.com`。
先带 `Authorization: Bearer <KeyB>` 调用 `GET /api/v1/skills`，200 的 `data`
为技能数组；空数组也表示鉴权通过。未配置时引导 `zq-config`，不索取 KeyA，
不回显 KeyB、不写入仓库或命令参数。MVP 没有余额查询接口。

如已连接平台 MCP，可使用 `create_upload`、`analyze_product_video`、
`get_product_video_analysis`、`generate_image`、`get_image_generation`、
`generate_video`、`get_video_generation`、`qc_product_video`、`get_product_video_qc`。
KeyB 只在 MCP 连接头中传入。模型任务创建工具的业务字段与 REST body 相同，另加
`idempotency_key`，可加 `wait_seconds`（0–20 秒）；REST 请求不发送这两个 body 字段。
`create_upload` 仅收 filename/content_type/size_bytes，不带这两个任务参数。
MCP 模型任务返回顶层 analysis_id / generation_id / qc_id、status、result、issues；
这些 ID 对应 REST 的任务 ID。MCP processing 表示仍在进行，failed/cancelled 是终态失败；调用错误读取 error_code，unknown 表示受理未知。
选一个通道执行，不能因等待超时切通道重新创建。

MCP 创建工具重放已终态任务时，可能只返回状态回执、`result=null` 和查询
`next_action`。这不表示没有产物；按原任务 ID 调用对应查询工具取得结果或失败
原因，不重新创建。只有查询详情后才能判断交付内容。

## 你需要向用户收集的物料

| 物料 | 要求 |
| --- | --- |
| 商品图片 | jpg/jpeg/png/webp，每张 1 字节至 20 MiB；输入分析接受 1–20 张 |
| 商品名称、卖点 | 非空字符串；多条卖点合并成字符串 |
| 类目、属性 | 可选字符串；未知就省略，不编造 |
| 视频规格 | 时长 10/15/20/30 秒；画幅 16:9/9:16/1:1；其余见题集 |

## 开始前提问

只补问缺失的必要信息；已提供的信息直接使用，选项无偏好时采用题集默认值。

```yaml
# zq-amazon-product-video —— intake 初始题集
# 直连模式下本题集即最终题集（无服务端动态追问）；
# 只放"提交能力任务前就能确定"的问题，用户跳过的按默认值提交。
version: 1
questions:
  - id: product_name
    ask: 商品名称是什么？（用于识别商品与包装身份）
    type: text
    required: true
  - id: selling_points
    ask: 商品的核心卖点有哪些？（将进入视频画面表达，可多条）
    type: text
    required: true
  - id: category
    ask: 商品属于哪个类目？
    type: choice
    choices: [自动识别, 美妆, 电脑/电子, 小家电, 家居, 服装, 食品]
    required: false
    default: 自动识别
  - id: attributes
    ask: 有没有商品属性资料（尺寸/材质/颜色/包装内容等）？可直接粘贴
    type: text
    required: false
  - id: duration
    ask: 目标视频时长是多少秒？
    type: choice
    choices: [10, 15, 20, 30]
    required: true
    default: 10
  - id: aspect_ratio
    ask: 画幅比例用哪种？（Amazon 商品页通常 16:9）
    type: choice
    choices: [16:9, 9:16, 1:1]
    required: true
    default: 16:9
  - id: resolution
    ask: 分辨率要求？
    type: choice
    choices: [480p, 720p, 1080p]
    required: false
    default: 720p
  - id: output_lang
    ask: 画面文字与交付说明使用什么语言？
    type: choice
    choices: [中文, English]
    required: false
    default: 中文
  - id: allow_human
    ask: 是否允许出现人物、手部或真实使用动作？
    type: boolean
    required: false
    default: false
  - id: need_audio
    ask: 是否需要生成音频？
    type: boolean
    required: false
    default: false
  - id: style_pref
    ask: 对视觉风格有偏好吗？（如清爽产品摄影、暖调生活场景；没有则由平台按类目选择）
    type: text
    required: false
  - id: must_show
    ask: 有没有必须展示的内容？
    type: text
    required: false
  - id: must_hide
    ask: 有没有禁止展示的内容？（如竞品、价格、特定人群）
    type: text
    required: false
```

## 执行流程

为每个能力创建请求生成独立的 8–128 字符 `Idempotency-Key`，保存该键、原完整
body 和返回 ID。同一请求恢复使用原键原参数；已批准的新返修是新任务，使用新键。
上传登记不保证幂等，不与分析/生成共用“全流程键”。

MVP 默认拒绝创建模型任务（`billing_unavailable`）。收费模式以平台响应为准：
`operator-funded` 项目允许时执行，响应 `billed={state:"not_charged"}`，运营方
承担模型成本；`selleros` 模式按网关实际 CNY 费用逐笔经 SellerOS 扣积分，受理
`billed={state:"pending"}`，扣费确认前查询返回 503 `billing_pending` /
`billing_blocked` 不交付产物——稍后重查原任务即可，不换键重新生成。终态查询
透传 `billed` 回执，只报积分口径；费用明细以 SellerOS 积分账单为准。不得承诺每步固定积分、余额冻结或"免费无成本"。
用户已要求制作视频且材料与下列确认完成后继续执行。

### 1. 上传

逐张读取真实文件元数据，不估算大小：

```http
POST http://skills-platform-api-uat.zhiqingresearch.com/v1/files
Authorization: Bearer <KeyB>
Content-Type: application/json

{"filename":"front.jpg","content_type":"image/jpeg","size_bytes":845210}
```

HTTP 201 返回 `file_ref`、`upload.method`、`upload.url`、`upload.headers` 和有效期。
按返回头向 `upload.url` PUT 原文件，在有效期内完成；PUT 不带 KeyB，不改签名参数，
不上传 multipart，也不另调 complete。不要提交 `skill_id`。

### 2. 输入分析

`POST http://skills-platform-api-uat.zhiqingresearch.com/v1/product-video/analysis` 提交：

- `file_refs`：1–20 项，已上传 file_ref 或 HTTPS 图片 URL；
- `product_name`、`selling_points`、`duration_seconds`、`aspect_ratio` 为必填；
- 可选 `category`、`attributes`、`target_market`、`style_pref`、`allow_human`、
  `must_show`、`must_hide`、`resolution`、`output_lang`、`need_audio`。

题集 `duration` 映射到 `duration_seconds` 数字；布尔值用 JSON boolean。
HTTP 202 返回 `analysis_id`，以
`GET http://skills-platform-api-uat.zhiqingresearch.com/v1/product-video/analysis/{analysisId}` 查询，路径填该 ID。
结果使用 `result.asset_coverage`、`result.product_facts` 与创意要点 `result.video_prompt`（类目/策略摘要/核心卖点/追加约束；平台提示词全文由服务端持有，响应不返回）。

### 3. 素材、事实与创意要点确认

1. 展示真实图及平台补充图，标明 AI 生成，逐张确认。生成视频需要**恰好 5 张不同的
   已确认参考图**；从素材中选择 5 张，不复制同一引用凑数。少于 5 张时请补充或
   使用平台补图结果。拒绝补充图时以用户原始意见作为 `feedback`，保留平台给出的
   提示词与引用；不得猜测内部补图提示词或不存在的 `retry_of`。
2. 展示商品事实、冲突和高风险宣称，使用用户确认的事实表。
3. 展示创意要点（类目、策略摘要、核心卖点、追加约束）并获用户确认；不展示、不索要提示词文本，平台提示词由服务端持有。
   可选补图端点为 `POST /v1/image/generation`（1–10 项 `file_refs`、非空 `prompt`，
   可选 `output_format`、`retry_of`、`feedback`），查询 `/v1/image/generation/{generationId}`。

### 4. 生成与质检

`POST http://skills-platform-api-uat.zhiqingresearch.com/v1/video/generation` body 为：

```json
{
  "file_refs": ["<确认图1>", "<确认图2>", "<确认图3>", "<确认图4>", "<确认图5>"],
  "analysis_id": "<分析任务 ID，服务端使用平台持有的提示词>",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "resolution": "720p",
  "with_audio": false
}
```

允许分辨率 480p/720p/1080p，实际支持及降级以结果 `issues` 为准。
HTTP 202 返回 `generation_id`；GET `/v1/video/generation/{generationId}` 查询。
成片生成后先展示视频。质检为可选能力：用户已要求质检则继续；否则请用户选择。
用户不选择质检时直接交付，并标明“未经平台质检”。以下重上传、质检和返修
只在用户选择质检后执行，不能默认创建额外模型任务。

先检查 `result.video_file_ref` 的类型：当前网关通常返回 HTTPS 视频 URL。
若要完成技术质检，下载该成片并读取准确文件元数据；不超过 **64 MiB** 时通过
`POST /v1/files` + PUT 重新上传，用本次登记返回的 `file_ref` 作为质检视频引用。
保留真实扩展名/MIME，不重编码或修改文件。64 MiB 是技术检查上限，低于通用
视频上传的 150 MiB 上限。下载或重上传无法完成、或文件超限时，可以用原 HTTPS
URL 做一致性评估，但必须注明技术检测不可判定。

提交 `POST /v1/product-video/qc`：
`video_file_ref`、1–30 项 `reference_file_refs`、已确认 `product_facts` 对象、
生成所用提示词（传 `generation_id`，服务端按该次生成实际使用的提示词作为基准）；可带 `storyboard` 对象及规格基准
`duration_seconds/aspect_ratio/resolution/with_audio`。
HTTP 202 返回 `qc_id`；GET `/v1/product-video/qc/{qcId}` 查询。

展示成片与平台质检结论。`technical_facts_unavailable` 表示缺少可检测的本地
视频事实；它不证明成片质量差，不能据此重生成。先补上上述成片上传，或交付时
明确技术未评估。实际画面质量未通过时，转述问题及
`result.revision_suggestions.revised_prompt` 字符串（可选分镜在同级 storyboard_draft），
用户批准后创建新的生成和质检任务。最多 3 轮返修，仍失败则停止并说明未解决项。
任务执行失败、超时或中断不等同于质量未通过，不触发自动返修。

### 5. 查询与下载

分析/质检 ≥5 秒、补图 ≥3 秒、视频 ≥10 秒查询一次。`queued/processing` 等待，
`completed` 读取结果，`failed/cancelled` 停止。约 10 分钟无结果时保存任务 ID，
交代可继续查询；MVP 默认任务期限 15 分钟，超时并不撤销已发给上游的调用。
用户要求停止时停止新提交；没有取消接口。

链接按结果实际提供的 URL 和有效期使用，不承诺固定 24 小时或可刷新。下载到
工作区后交付文件；下载失败保留 ID，查询原任务，不自动重新生成。

## 交付

交付平台生成的视频、已确认素材与事实表、质检报告和返修记录（提示词由平台服务端持有，不随交付物出现）。
用户未选择质检时标明“未经平台质检”，不编造质检报告。
标明 AI 生成素材，转述 `issues` 和未解决问题，不将质检结果描述为平台上架保证。

保留分析、生成、质检任务 ID 便于恢复查询。下载链接有效期以服务返回为准；
费用字段缺失不代表零成本，不用历史积分单价或 token 推算账单。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 | 重新检查 KeyB、API origin 和项目状态 |
| 404 | 核对任务/素材 ID 与所属用户，不据此新建任务 |
| 409 | 核对原幂等键及原完整 body，不自动换键 |
| 422 | 按字段提示修正类型、数量、上传状态和参数 |
| 429 | 总并发或 KeyB 达上限；等待后以原键原参数有限重试 |
| billing_unavailable / skill_unavailable | 停止并联系项目方，不充值或更改项目属性绕过 |
| billing_pending / billing_blocked（查询时，selleros 模式） | 扣费未确认或被拒，不交付产物：稍后重查原任务；被拒时联系项目方处理原账单，不换键重新生成 |
| admission_timeout / submission_unknown / POST 5xx 或网络异常 | 受理结果未知；有 ID 先查询，无 ID 保存原键与完整 body，仅用户明确恢复时原样重放 |
| task_timeout / task_interrupted | 查询原任务，禁止自动换键重跑；不能声称上游未执行或无费用 |
| GET 5xx / 网络异常 | 间隔重试最多 2 次，仍失败保留 ID |
