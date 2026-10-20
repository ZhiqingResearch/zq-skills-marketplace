---
name: zq-amazon-product-video
description: 当用户提供商品图片、名称和卖点，希望制作 Amazon 商品广告视频或对成片质检返修时使用。根据已确认的商品事实与创意生成视频，按可用通道进行补图、分析和可选质检。
---
<!-- zq-skills: zq-amazon-product-video v2.0.0 target=claude-code -->

## 平台连接

所有平台 REST 请求固定使用 `https://skills-platform-api.zhiqingresearch.com`，只需配置 KeyB。


# Amazon 商品广告视频

## 何时使用与开始概述

首次使用时简述本技能根据商品图、事实与创意制作广告视频，各模型任务按实际费用经
SellerOS 扣积分；分析、补图和质检各自独立计费，只在确有需要且获授权时执行。
纯剪辑、视频理解或 Listing 文案使用对应技能。

生成需要恰好 5 张不同且已确认的商品图、商品名称、可核实卖点、4–30 的整数秒时长
及 16:9/9:16/1:1 画幅；可选分辨率、语言和风格。视频固定有声。只补问缺项，
已确认的素材、方案和费用授权沿用，不重复要求用户确认。

## API key 配置与通道

REST 固定使用 `https://skills-platform-api.zhiqingresearch.com`，KeyB 从安全环境变量 `ZQ_API_KEY` 读取，
缺项再读 `~/.config/zq-skills/credentials`，按首个 `=` 分割，不 source/eval。
先带 `Authorization: Bearer <KeyB>` 调用 `GET /api/v1/skills` 验证鉴权；
200 的空 data 数组也有效，不证明余额或可扣费状态。缺凭据引导 zq-config，
不索取 KeyA，不回显 KeyB，不写入日志、命令参数或交付物。

MCP 当前提供 `create_upload`、`generate_image`、`get_image_generation`、
`generate_video`、`get_video_generation`；商品分析与质检暂不通过 MCP 提供。
仅有 MCP 时，根据用户提供或查询得到的商品事实编写创意与 prompt，确认后直接生成，
不要求用户自行提供提示词全文，不等待不可用的分析／质检工具。
已有同用户已完成且费用已确认的分析或质检结果，也可通过 analysis_id / qc_id 引用；
不可用时说明限制，不擅自切换通道或重新付费分析。

REST 的分析和质检是可选步骤，需可用凭据、对应服务和用户授权。先说明通道限制与
费用，再决定是否执行；不能为补齐流程而默认创建额外任务。

KeyB 仅放 MCP 连接头。MCP 创建参数是 REST 业务字段加 idempotency_key，可加
wait_seconds（0–20 秒）；create_upload 仅传 filename/content_type/size_bytes。
MCP 返回顶层 generation_id、status、result、issues；REST 以各端点实际包装为准。
processing 表示尚未终态，failed/cancelled 停止；调用错误读取 error_code。
即使创建或重放回执显示终态，也用原 ID 查询详情再判断产物，不把 result=null 当作需要重建。

## 物料与开始前提问

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

图片支持 jpg/jpeg/png/webp，每张 1 字节至 20 MiB。HTTPS 商品图可直接引用，
本地图片先按下节上传。生成前必须确认恰好 5 张不同参考图；不足时先补充，
补图需独立授权，不能复制同一图片凑数。允许先提供 1–20 张图进行可选 REST 分析。

展示商品事实、冲突和创意要点，确认镜头、风格、规格、人物／手部限制与费用。
未知事实保持缺失；参考图中的营销文案不直接视为已证实的宣称。私有平台提示词
留在服务端，不索取、不展示。客户端自写 prompt 仅使用已确认的事实与创意。

## 上传与可选分析

逐张读取真实文件名、MIME 和大小，再登记：

```http
POST https://skills-platform-api.zhiqingresearch.com/v1/files
Authorization: Bearer <KeyB>
Content-Type: application/json

{"filename":"front.jpg","content_type":"image/jpeg","size_bytes":845210}
```

上传登记成功只代表取得 file_ref。按 upload.method=POST 向 upload.url 发送 multipart/form-data：upload.fields 原样逐个添加，最后添加 file 原始字节；由表单库生成 Content-Type/boundary。遵守 upload.max_size_bytes、MIME 和 expires_at，存储返回 2xx 后才使用 file_ref。对象存储请求不携带 KeyB；不使用 PUT、raw POST 或把 fields 当 headers。登记结果未知时停止并保留回执，不自动重新登记。

需要补图时，用独立幂等键提交已批准的修改要求；不猜测私有补图提示词：

<!-- zq-request: POST /v1/image/generation -->

```json
{
  "file_refs": ["https://example.com/product.png"],
  "prompt": "保持商品结构，使用纯白背景",
  "output_format": "png"
}
```

用 GET `/v1/image/generation/{generationId}` 查询。图片格式与 retry_of 是否可用以
当前服务和结果为准；不能因能力不可用而循环创建新任务。

仅在 REST 分析可用且用户要求时提交：

<!-- zq-request: POST /v1/product-video/analysis -->

```json
{
  "file_refs": ["file_00000000000000000000000000000001"],
  "product_name": "Travel Mug",
  "selling_points": "用户确认：500 ml 容量",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "need_audio": true
}
```

其他可选字段为 category、attributes、target_market、style_pref、allow_human、
must_show、must_hide、resolution、output_lang。GET `/v1/product-video/analysis/{analysisId}`
查询 asset_coverage、product_facts 和 video_prompt 创意摘要，不读取私有提示词全文。

## 执行流程：视频生成

每次新任务创建独立的 8–128 字符 Idempotency-Key，保存原完整请求、键和返回 ID。
同任务恢复使用原键原参数；只有用户明确要求新任务或批准返修才换新键。
REST 幂等键放请求头，MCP 放 idempotency_key。

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
  "prompt": "按已确认的商品图制作 10 秒广告，保持杯身与杯盖结构一致，突出已核实的 500 ml 容量，无人物或手部。",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "resolution": "720p"
}
```

prompt / analysis_id / qc_id 必须且只能提供一项。已有分析或质检引用时替换 prompt，
不同时发送。支持 480p/720p/1080p，实际支持及降级以 issues 为准。

视频生成固定有声，REST 与 MCP 均不发送 with_audio。若通过 REST 先做商品分析，need_audio 仅表达创意意图（API 缺省 false），按本次有声方案显式传 true，不能用它切换生成音轨。用户要求静音时先说明限制，可按其授权在本地制作静音副本并保留平台原版，不提交已移除的生成参数。质检的 with_audio 仍是可选音频检测基准，与生成参数不同。

HTTP 202 返回 generation_id，用 GET `/v1/video/generation/{generationId}` 或
get_video_generation 查询。费用以 billed 的实际积分回执为准；pending/blocked
不交付结果，稍后查询原任务。缺少费用回执不等于免费，不能用 usage 或 token 推算账单。

## 可选质检与返修（REST）

成片先展示。未要求质检时直接交付并标明“未经平台质检”；MCP 未提供质检工具，
不能在仅有 MCP 时承诺平台质检。用户要求且 REST 服务可用时，才创建独立付费质检任务。

技术检查需下载真实成片并读元数据，不超过 64 MiB 时按 POST 表单重新上传，
video_file_ref 使用这次登记的 file_ref。保留真实扩展名/MIME和原版；无法完成时，
可用原 HTTPS 视频做一致性评估，但说明技术检测不可判定。

平台 file_ref 指向的 MP4 必须为 faststart（moov 在 mdat 前）。上传前检查；需要调整时创建无损转封装副本：ffmpeg -i 输入.mp4 -c copy -movflags +faststart 输出.mp4。读取副本的真实大小后登记上传；422 返回布局指引时按指引处理并重新上传，不覆盖已经绑定的上传对象。此预检不扩展到外部 HTTPS 视频或其他容器。

<!-- zq-request: POST /v1/product-video/qc -->

```json
{
  "video_file_ref": "file_00000000000000000000000000000006",
  "reference_file_refs": ["file_00000000000000000000000000000001"],
  "product_facts": {"name":"Travel Mug","capacity":"500 ml"},
  "generation_id": "vid_000000000000000000000000",
  "with_audio": true
}
```

质检可带 storyboard 及 duration_seconds/aspect_ratio/resolution 基准；
generation_id 与调用方自写 video_prompt 二选一。GET `/v1/product-video/qc/{qcId}`
查询报告。technical_facts_unavailable 不证明画质差，不能据此重新生成。

质检返修仅交付公开问题、add_constraints 和可选分镜。平台提示词留在服务端；批准返修后用 qc_id 引用修订内容，新生成请求的 prompt / analysis_id / qc_id 必须三选一，不索取或展示私有提示词。新任务保留用户确认的图片与规格，并使用独立幂等键。

最多 3 轮用户批准的质量返修；生成失败、超时或中断不视为质量不通过，不自动重建。

## 查询、下载与出错处理

按 next_action.retry_after_seconds 自动续查；没有建议时分析／质检至少 5 秒、补图
至少 3 秒、视频至少 10 秒。视频观察期限约 12 分钟，其余约 7 分钟；到期仍在处理时
保存 ID 交代后续查询方式，不凭等待超时推定失败或无成本。用户停止时不再提交任务，
停止本地等待不等于取消上游任务。

NEW 202 是受理提交时的初始回执，不代表响应送达时的当前执行或费用状态；随后用原任务 ID 查询，重放也读取原任务，不换键新建。图片或视频查询出现 phase=recovering 时，说明“上游调用异常，正在确认原任务结果，请勿重复提交”，不能称为正常生成中。创建响应即使为恢复型 503，只要带已确认任务 ID，仍查询原任务。failed/cancelled 是终态，停止轮询；失败不证明未收费或已退款，费用缺失也不表示免费，usage 不作为积分账单。429 可能是时间窗口限流或任务容量限制。优先按 Retry-After 或 MCP next_action.retry_after_seconds 等待，不把等待建议当成完成时间。上传登记当前每用户 30 次/小时，任务创建共享每分钟 60 次 NEW 准备；原键有效重放不消耗新任务额度。任务恢复复用原键原完整参数，不换 Key 或幂等键绕过限流。明确的上传 429 未签发 file_ref，可等待后重试；网络结果未知时不自动重新登记。

completed 后按返回的 download_url 与 expires_at 下载，不承诺固定链接寿命。
下载失败保留原 ID；GET 网络或 5xx 间隔重试最多 2 次，仍失败停止并交代记录。
遇到 billing_pending 稍后重查原任务；billing_blocked 联系项目方处理原账单。
401/404 核对凭据与所属任务；409 核对原键原参数；422 按字段提示修正。
POST 结果未知且无 ID 时保存原键原请求，仅用户明确恢复才原样重放。
billing_unavailable / skill_unavailable 停止并联系项目方，不更改计费属性绕过。

## 交付

交付平台生成的视频、已确认素材与事实表、质检报告和返修记录（提示词由平台服务端持有，不随交付物出现）。
用户未选择质检时标明“未经平台质检”，不编造质检报告。
标明 AI 生成素材，转述 `issues` 和未解决问题，不将质检结果描述为平台上架保证。

保留分析、生成、质检任务 ID 便于恢复查询。下载链接有效期以服务返回为准；
费用字段缺失不代表零成本，不用历史积分单价或 token 推算账单。


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
  "prompt": "按已确认的商品图制作 10 秒广告，保持杯身与杯盖结构一致，突出已核实的 500 ml 容量，无人物或手部。",
  "duration_seconds": 10,
  "aspect_ratio": "16:9",
  "resolution": "720p",
  "idempotency_key": "example-request-0001"
}
```
