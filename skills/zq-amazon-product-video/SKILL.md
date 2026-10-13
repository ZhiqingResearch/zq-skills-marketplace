---
name: zq-amazon-product-video
description: 当用户提供商品图片、名称和卖点，希望制作 Amazon 商品广告视频或对成片质检返修时使用。通过平台分析、补图、生成和质检能力完成。
---
<!-- zq-skills: zq-amazon-product-video v1.3.9 target=claude-code -->

## 平台连接

所有平台 REST 请求固定使用 `https://skills-platform-api.zhiqingresearch.com`，只需配置 KeyB。


# Amazon 商品广告视频


## 何时使用与开始概述（本技能每次会话首次被调用时）

先用两三句话向用户概述本技能能做什么与计费方式，再列出本次操作需要用户
提供的信息清单（标明必填/可选；用户已给出的不重复询问）。清单齐备后才
开始执行；用户问"这个技能是干嘛的"时也按此概述回答。

概述示例：本技能用商品图和卖点制作 Amazon 商品广告视频——先平台分析素材
与事实，再生成视频，可做质检返修；每步按次经 SellerOS 扣积分。本次需要：
① 1–20 张商品图（必填，生成前确认 5 张不同图片）；② 商品名称（必填）；③ 核心卖点（必填）；
④ 时长 4–30 的整数秒与画面比例 16:9/9:16/1:1（必填）；可选：分辨率、风格偏
好、目标市场。默认生成有声音的视频，需要静音时说明即可。


用于商品广告视频生成和基于质检结果的返修；纯剪辑、视频理解或 Listing 文案
分别使用对应工具。本技能通过平台 REST 或已连接的 MCP 调用。

## API key 配置

REST 使用当前用户 KeyB（通常 `zk-`，兼容旧 `sk_`）。优先读取安全注入的
`ZQ_API_KEY`，缺项再读取 `~/.config/zq-skills/credentials`，
按首个 `=` 分割，不 source/eval。所有 REST 请求固定使用生产地址 `https://skills-platform-api.zhiqingresearch.com`。
先带 `Authorization: Bearer <KeyB>` 调用 `GET /api/v1/skills`，200 的 `data`
为技能数组；空数组也表示鉴权通过。未配置时引导 `zq-config`，不索取 KeyA，
不回显 KeyB、不写入仓库或命令参数。能力发现结果不表示积分余额或可扣费状态。

先确认可用通道。当前 MCP 提供 `create_upload`、`generate_image`、`get_image_generation`、
`generate_video`、`get_video_generation`；商品资料分析及质检暂不通过 MCP 提供。
完整分析/质检流程需要可用 REST 凭据与对应服务；仅有 MCP 时先说明限制，
可用用户自写创意提示词或已有分析/质检 ID 生成视频，不承诺自动完成全流程。
缺少完整流程所需能力时，在创建任何付费任务前说明缺项，不切通道重建原任务。
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
| 视频规格 | 时长 4–30 的整数秒；画幅 16:9/9:16/1:1；其余见题集 |

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
    ask: 目标视频时长是多少秒？（4 到 30 之间的整数秒）
    type: number
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
    ask: 是否生成有声音的视频？（默认有声音；需要静音视频选否）
    type: boolean
    required: false
    default: true
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

每项模型任务按平台实际费用经 SellerOS 扣积分，费用状态以原任务 `billed`
回执为准。`billed={state:"pending"}` 表示尚待确认，扣费确认前查询返回 503 `billing_pending` /
`billing_blocked` 不交付产物——稍后重查原任务即可，不换键重新生成。终态查询
透传 `billed` 回执，只报积分口径；费用明细以 SellerOS 积分账单为准。不得承诺每步固定积分、余额冻结或"免费无成本"。
用户已要求制作视频且材料与下列确认完成后继续执行。

### 1. 上传

逐张读取真实文件元数据，不估算大小：

```http
POST https://skills-platform-api.zhiqingresearch.com/v1/files
Authorization: Bearer <KeyB>
Content-Type: application/json

{"filename":"front.jpg","content_type":"image/jpeg","size_bytes":845210}
```

上传登记成功只代表取得 file_ref。按 upload.method=POST 向 upload.url 发送 multipart/form-data：upload.fields 原样逐个添加，最后添加 file 原始字节；由表单库生成 Content-Type/boundary。遵守 upload.max_size_bytes、MIME 和 expires_at，存储返回 2xx 后才使用 file_ref。对象存储请求不携带 KeyB；不使用 PUT、raw POST 或把 fields 当 headers。登记结果未知时停止并保留回执，不自动重新登记。

### 2. 输入分析

`POST https://skills-platform-api.zhiqingresearch.com/v1/product-video/analysis` 提交：

- `file_refs`：1–20 项，已上传 file_ref 或 HTTPS 图片 URL；
- `product_name`、`selling_points`、`duration_seconds`、`aspect_ratio` 为必填；
- 可选 `category`、`attributes`、`target_market`、`style_pref`、`allow_human`、
  `must_show`、`must_hide`、`resolution`、`output_lang`、`need_audio`（本技能按用户选择显式传布尔值）。

<!-- zq-request: POST /v1/product-video/analysis -->

```json
{
  "file_refs": [
    "file_00000000000000000000000000000001"
  ],
  "product_name": "Travel Mug",
  "selling_points": "用户确认：500 ml 容量",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "need_audio": true
}
```

题集 `duration` 映射到 `duration_seconds` 数字；布尔值用 JSON boolean。
HTTP 202 返回 `analysis_id`，以
`GET https://skills-platform-api.zhiqingresearch.com/v1/product-video/analysis/{analysisId}` 查询，路径填该 ID。
结果使用 `result.asset_coverage`、`result.product_facts` 与创意要点 `result.video_prompt`（类目/策略摘要/核心卖点/追加约束；平台提示词全文由服务端持有，响应不返回）。

### 3. 素材、事实与创意要点确认

1. 展示真实图及平台补充图，标明 AI 生成，逐张确认。生成视频需要**恰好 5 张不同的
   已确认参考图**；从素材中选择 5 张，不复制同一引用凑数。少于 5 张时请补充或
   使用平台补图结果。拒绝补充图时以用户原始意见作为 `feedback`，保留平台给出的
   任务 ID 与引用；不得猜测内部补图提示词或不存在的 `retry_of`。
2. 展示商品事实、冲突和高风险宣称，使用用户确认的事实表。
3. 展示创意要点（类目、策略摘要、核心卖点、追加约束）并获用户确认；不展示、不索要提示词文本，平台提示词由服务端持有。
   可选补图端点为 `POST /v1/image/generation`（1–10 项 `file_refs`、非空 `prompt`，
   可选 `output_format`、`retry_of`、`feedback`），查询 `/v1/image/generation/{generationId}`。

补图格式示例（prompt 为用户批准的修改要求，参考图遵守接口支持的静态图片格式）：

<!-- zq-request: POST /v1/image/generation -->

```json
{
  "file_refs": [
    "https://example.com/product.png"
  ],
  "prompt": "保持商品结构，使用纯白背景",
  "output_format": "png"
}
```

### 4. 生成与质检

`POST https://skills-platform-api.zhiqingresearch.com/v1/video/generation` body 为：

<!-- zq-request: POST /v1/video/generation -->

```json
{
  "file_refs": [
    "file_00000000000000000000000000000001",
    "file_00000000000000000000000000000002",
    "file_00000000000000000000000000000003",
    "file_00000000000000000000000000000004",
    "file_00000000000000000000000000000005"
  ],
  "analysis_id": "pva_000000000000000000000000",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "resolution": "720p",
  "with_audio": true
}
```

允许分辨率 480p/720p/1080p，实际支持及降级以结果 `issues` 为准。时长须为
4–30 的整数秒（旧档位 10/15/20/30 仍合法，出界或非整数会被 422 拒绝）。
API 的 need_audio 与 with_audio 缺省均为 false。本技能默认有声时必须显式提交 need_audio=true 和 with_audio=true；用户选择静音时两处显式设为 false，不能依赖 API 缺省值。

HTTP 202 返回 `generation_id`；GET `/v1/video/generation/{generationId}` 查询。
成片生成后先展示视频。质检为可选能力：用户已要求质检则继续；否则请用户选择。
用户不选择质检时直接交付，并标明“未经平台质检”。以下重上传、质检和返修
只在用户选择质检后执行，不能默认创建额外模型任务。

先检查 `result.video_file_ref` 的类型：当前网关通常返回 HTTPS 视频 URL。
若要完成技术质检，下载该成片并读取准确文件元数据；不超过 **64 MiB** 时通过
`POST /v1/files` 登记后按 POST 表单重新上传，用本次登记返回的 `file_ref` 作为质检视频引用。
保留真实扩展名/MIME；如需调整 MP4 布局，只创建无损转封装副本。64 MiB 是技术检查上限，低于通用
视频上传的 150 MiB 上限。下载或重上传无法完成、或文件超限时，可以用原 HTTPS
URL 做一致性评估，但必须注明技术检测不可判定。

平台 file_ref 指向的 MP4 必须为 faststart（moov 在 mdat 前）。上传前检查；需要调整时创建无损转封装副本：ffmpeg -i 输入.mp4 -c copy -movflags +faststart 输出.mp4。读取副本的真实大小后登记上传；422 返回布局指引时按指引处理并重新上传，不覆盖已经绑定的上传对象。此预检不扩展到外部 HTTPS 视频或其他容器。

提交 `POST /v1/product-video/qc`：
`video_file_ref`、1–30 项 `reference_file_refs`、已确认 `product_facts` 对象、
生成所用提示词（传 `generation_id`，服务端按该次生成实际使用的提示词作为基准）；可带 `storyboard` 对象及规格基准
`duration_seconds/aspect_ratio/resolution/with_audio`。
<!-- zq-request: POST /v1/product-video/qc -->

```json
{
  "video_file_ref": "file_00000000000000000000000000000006",
  "reference_file_refs": [
    "file_00000000000000000000000000000001",
    "file_00000000000000000000000000000002",
    "file_00000000000000000000000000000003",
    "file_00000000000000000000000000000004",
    "file_00000000000000000000000000000005"
  ],
  "product_facts": {
    "name": "Travel Mug",
    "capacity": "500 ml"
  },
  "generation_id": "vid_000000000000000000000000",
  "with_audio": true
}
```

HTTP 202 返回 `qc_id`；GET `/v1/product-video/qc/{qcId}` 查询。

展示成片与平台质检结论。`technical_facts_unavailable` 表示缺少可检测的本地
视频事实；它不证明成片质量差，不能据此重生成。先补上上述成片上传，或交付时
明确技术未评估。实际画面质量未通过时，转述问题及
公开的修改要点 `result.revision_suggestions.add_constraints` 与可选分镜；
服务端不返回私有提示词全文。用户批准返修后，新生成请求以 `qc_id` 替换 `analysis_id`，
两者与用户自写 `prompt` 必须三选一，其余确认图、音频等规格显式保留。
生成后仅在用户授权质检时创建新质检任务。最多 3 轮返修，仍失败则停止并说明未解决项。
任务执行失败、超时或中断不等同于质量未通过，不触发自动返修。

### 5. 查询与下载

自动轮询至终态（分析/质检 ≥5 秒、补图 ≥3 秒、视频 ≥10 秒一次；不结束回合、不需用户催促，终态或约 3 分钟后汇报；视频生成可能需要数分钟）。`queued/processing` 由 Agent
自动按间隔续查直至终态，不需要用户再次发送查询指令；`completed` 读取结果，
`failed/cancelled` 停止。视频生成受理后执行期限约 12 分钟（排队 2 分钟 + 执行
10 分钟，2026-10-08 起按操作放宽，其余能力约 7 分钟），到期仍 `processing` 时
保存任务 ID 并向用户交代可继续查询；超期进入 15 分钟对账恢复窗口，超时并不
撤销已发给上游的调用。用户要求停止时停止新提交；没有取消接口。

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
| 429 | 按下方限流约定等待，保留原请求与任务 ID |
| billing_unavailable / skill_unavailable | 停止并联系项目方，不充值或更改项目属性绕过 |
| billing_pending / billing_blocked（查询时） | 扣费未确认或被拒，不交付产物：稍后重查原任务；被拒时联系项目方处理原账单，不换键重新生成 |
| admission_timeout / submission_unknown / POST 5xx 或网络异常 | 受理结果未知；有 ID 先查询，无 ID 保存原键与完整 body，仅用户明确恢复时原样重放 |
| task_timeout / task_interrupted | 查询原任务，禁止自动换键重跑；不能声称上游未执行或无费用 |
| GET 5xx / 网络异常 | 间隔重试最多 2 次，仍失败保留 ID |

NEW 202 是受理提交时的初始回执，不代表响应送达时的当前执行或费用状态；随后用原任务 ID 查询，重放也读取原任务，不换键新建。429 可能是时间窗口限流或任务容量限制。优先按 Retry-After 或 MCP next_action.retry_after_seconds 等待，不把等待建议当成完成时间。上传登记当前每用户 30 次/小时，任务创建共享每分钟 60 次 NEW 准备；原键有效重放不消耗新任务额度。任务恢复复用原键原完整参数，不换 Key 或幂等键绕过限流。明确的上传 429 未签发 file_ref，可等待后重试；网络结果未知时不自动重新登记。

质检返修仅交付公开问题、add_constraints 和可选分镜。平台提示词留在服务端；批准返修后用 qc_id 引用修订内容，新生成请求的 prompt / analysis_id / qc_id 必须三选一，不索取或展示私有提示词。新任务保留用户确认的图片与规格，并使用独立幂等键。


## MCP 请求示例

以下仅为格式示例；替换为本次真实物料与任务 ID，新业务使用独立稳定幂等键。

### MCP：create_upload

<!-- zq-tool: create_upload -->

```json
{
  "filename": "front.jpg",
  "content_type": "image/jpeg",
  "size_bytes": 845210
}
```

### MCP：generate_image

<!-- zq-tool: generate_image -->

```json
{
  "file_refs": [
    "https://example.com/product.png"
  ],
  "prompt": "保持商品结构，使用纯白背景",
  "output_format": "png",
  "idempotency_key": "example-request-0001"
}
```

### MCP：generate_video

<!-- zq-tool: generate_video -->

```json
{
  "file_refs": [
    "file_00000000000000000000000000000001",
    "file_00000000000000000000000000000002",
    "file_00000000000000000000000000000003",
    "file_00000000000000000000000000000004",
    "file_00000000000000000000000000000005"
  ],
  "analysis_id": "pva_000000000000000000000000",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "resolution": "720p",
  "with_audio": true,
  "idempotency_key": "example-request-0001"
}
```
