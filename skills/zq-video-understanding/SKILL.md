---
name: zq-video-understanding
description: 上传用户视频到平台，异步生成结构化视觉分析，并由平台脚本按模型选出的时间点提取 6 至 10 张真实关键帧。
---
<!-- zq-skills: zq-video-understanding v2.1.0 target=claude-code -->

# 视频理解与平台关键帧提取

把用户提供的视频通过预签名地址直传平台对象存储，启动异步分析，轮询结果，
再下载平台脚本从原视频提取的关键帧。不要在客户端自行猜测时间点，也不要把
平台内部提示词、模型路由或脚本内容复述给用户。

## 何时使用 / 何时不使用

- 使用：用户希望理解、总结一个 `.mp4`、`.mov` 或 `.webm` 视频，或希望
  获取 6–10 张有代表性的真实画面。
- 不使用：实时视频流、超过 150 MiB 的视频、纯剪辑/转码任务，以及要求完整
  对白转写或声音分析的任务。当前 Demo 使用抽样画面做视觉分析，不处理音轨。

## API key 配置

所有平台请求都使用项目为该用户签发的 KeyB：
`Authorization: Bearer <ZQ_API_KEY>`。KeyB 通常以 `zk-` 开头，兼容历史 `sk_`，只允许操作创建
它的同一项目与用户数据。

1. 优先读取当前会话安全注入的 `ZQ_API_KEY` 环境变量，缺项再从
   `~/.config/zq-skills/credentials` 读取。
2. 若未配置，提示用户向接入项目领取 KeyB，或使用 `zq-config` 完成配置；
   不得要求用户把 KeyA 或对象存储密钥交给你。
3. 不回显完整 KeyB，不把它写入仓库、日志、命令参数或交付文件。
4. REST 先用 `GET http://skills-platform-api-uat.zhiqingresearch.com/api/v1/skills` 验证，HTTP 200 的 `data` 是能力数组；
   即使为空也表示此次鉴权成功。MVP 没有余额查询端点。
5. API 地址优先取环境变量 `ZQ_API_BASE`，再取凭据文件；最后使用安装包中地址。
   地址为 origin，不加 `/api` 或 `/v1` 后缀；凭据按首个 `=` 分割，不 source/eval。

MCP 创建工具重放已终态任务时，可能只返回状态回执、`result=null` 和查询
`next_action`。这不表示没有产物；按原任务 ID 调用对应查询工具取得结果或失败
原因，不重新创建。只有查询详情后才能判断交付内容。

## 你需要向用户收集的物料

| 物料 | 说明 | 必须 |
| --- | --- | --- |
| 视频文件 | `.mp4` / `.mov` / `.webm`，1 字节至 150 MiB | 是 |
| 截图数量 | 6–10，默认 8 | 否 |
| 结果语言 | `中文` 或 `English`，默认中文 | 否 |
| 视觉关注点 | 例如“重点看商品结构变化”；最长 1000 字符 | 否 |

本地读取文件名、字节数和扩展名，并按以下映射确定 `content_type`：
`.mp4 → video/mp4`、`.mov → video/quicktime`、`.webm → video/webm`。

## 开始前提问

只询问用户尚未给出的可选项。用户没有偏好时直接采用 `shot_count=8`、
`analysis_lang=中文`，不要把平台内部供应商选择暴露成必答问题。开始上传前
明确告知：原视频正常处理结束会清理，异常退出可能留存；关键帧使用私有存储与短期签名链接。

## 执行流程

API 根地址使用 `http://skills-platform-api-uat.zhiqingresearch.com`。平台请求带 KeyB；预签名 PUT 请求绝不能
携带 KeyB。为视频分析生成一个 8–128 字符的稳定 `Idempotency-Key`，网络
恢复时必须复用原键和原完整请求，不能自动重发未知结果的 POST。

若已连接该平台 MCP，可使用 `create_upload` → `analyze_video` →
`get_video_analysis` → `get_video_frame`；模型任务创建工具 `analyze_video` 增加 `idempotency_key`，
可设置 `wait_seconds`（0–20 秒），KeyB 只在连接头中传入。PUT 仍由客户端执行；
`create_upload` 仅收 filename/content_type/size_bytes，不加任务参数。
MCP 分析工具返回顶层 analysis_id、status、result、issues，processing 表示仍在进行，
failed/cancelled 是终态失败；调用错误读取 error_code，unknown 表示受理未知；`get_video_frame` 返回可下载的签名 URL，不额外带 KeyB。
REST 与 MCP 二选一，不要切通道重复提交。MCP 参数的等待时间不传给 REST 查询。

### 1. 登记并直传视频

```http
POST http://skills-platform-api-uat.zhiqingresearch.com/v1/files
Authorization: Bearer <ZQ_API_KEY>
Content-Type: application/json

{
  "filename": "demo.mp4",
  "content_type": "video/mp4",
  "size_bytes": 10485760
}
```

成功返回 HTTP 201，包含 `file_ref`、`upload.url`、`upload.headers`、
`upload.expires_at` 和 `expires_at`。随后在 15 分钟内执行：

```http
PUT {upload.url}
Content-Type: <upload.headers 中的 Content-Type>

<原始视频二进制>
```

PUT 通常返回 200 或 204。必须原样使用平台返回的 URL 与请求头，不修改查询
参数，不向 Spaces/CDN 地址附加 `Authorization`。

上传完成后保持对象不变。平台在分析受理时绑定 ETag；不要向同一预签名 URL
覆盖或重传新内容。更换视频需重新登记上传，获得新的 file_ref。存储缺少 ETag
可返回 503 service_unavailable；下载版本、大小或 Content-Type 不符，超过 150 MiB
或整体下载超过 30 秒会使任务失败，先检查上传/存储，不能自动重跑模型任务。

### 2. 创建异步分析

```http
POST http://skills-platform-api-uat.zhiqingresearch.com/v1/video/analysis
Authorization: Bearer <ZQ_API_KEY>
Idempotency-Key: <本次分析稳定键>
Content-Type: application/json

{
  "file_ref": "file_...",
  "provider": "auto",
  "shot_count": 8,
  "analysis_lang": "中文",
  "extra_focus": "可省略"
}
```

HTTP 202 返回 `analysis_id`、`status` 和 `billed`。收费模式以平台响应为准：
`billed={state:"not_charged"}` 表示运营方承担成本、未扣费；`selleros` 模式
按网关实际 CNY 费用逐笔经 SellerOS 扣积分（受理 `state:"pending"`）。
服务端默认禁用模型创建，返回 `billing_unavailable` 时停止并联系项目方，
不能要求充值或修改计费属性绕过；扣费确认前查询与关键帧下载可能返回
503 `billing_pending` / `billing_blocked`——稍后重查原任务即可，不换键重新生成。

### 3. 轮询并获取结果

每隔至少 5 秒查询一次：

```http
GET http://skills-platform-api-uat.zhiqingresearch.com/v1/video/analysis/{analysis_id}
Authorization: Bearer <ZQ_API_KEY>
```

- `queued` / `processing`：继续等待；
- `completed`：读取 `result` 和 `issues`；
- `failed` / `cancelled`：停止轮询并转述 `issues`；
- 超过约 10 分钟仍未终态：保存 `analysis_id`，提示用户稍后继续查询，
  不要高频空转。

`result.shots[].time` 是模型选择的原视频秒数，`frame_url` 是平台已按该时间点
执行抽帧脚本得到的 JPEG 入口，一律返回以 `/` 开头的**相对路径**（如
`/v1/video/analysis/{analysis_id}/frames/frame-01-0000-01-0.jpg`），不是完整 URL。
不要再次在本地抽帧。

### 4. 下载关键帧

`frame_url` 是相对路径，不能直接交给 `curl`/`wget` 等下载工具（缺少主机名，
必然失败）。必须先拼接平台根地址得到完整 URL 再请求：

```text
完整地址 = http://skills-platform-api-uat.zhiqingresearch.com + frame_url
示例     = http://skills-platform-api-uat.zhiqingresearch.com/v1/video/analysis/analysis_abc123/frames/frame-01-0000-01-0.jpg
```

拼接规则：`frame_url` 自带开头的 `/`；`API_BASE_URL` 末尾若带有 `/`，先去掉
再拼接，避免路径出现 `//`。对每个完整地址发带 KeyB 的 GET。平台校验归属后
返回 HTTP 302，跳转到私有存储的短期签名地址；跟随跳转保存 JPEG 文件。跟随
跨域跳转时不要把 KeyB 转发给目标存储域名。

## 交付

向用户交付：结构化摘要、关键词、人物、物体、场景时间轴、按顺序命名的
关键帧，以及 `result.usage` 实际返回的 model、input_tokens、output_tokens 等用量。货币费用字段可能省略，不能把缺失金额当作零费用，
也不能根据 token 或历史报价自行计算实际账单。`issues` 必须一并
转述；尤其不能隐瞒 `sampled_visual_analysis`，它表示当前 Demo 只基于候选帧
做视觉分析，音轨或采样间瞬时画面可能未被覆盖。

## 出错处理

| 状态 | 处理 |
| --- | --- |
| 401 | KeyB 缺失、失效或不属于当前项目用户；引导重新领取/配置 |
| 409 | 幂等冲突；核对原键与原完整请求，不自动换键新建 |
| 422 | 按 `field_errors` 修正文件类型、大小、上传状态或分析参数 |
| 429 | 总并发或当前 KeyB 达上限；等待后原键原参数有限重试，不并行换键 |
| 503 billing_unavailable | 模型任务未启用或项目要求扣费；停止并联系项目方 |
| 503 billing_pending / billing_blocked（查询或关键帧下载时，selleros 模式） | 扣费未确认或被拒，不交付产物：稍后重查原任务/重取关键帧；被拒时联系项目方处理原账单，不换键重新生成 |
| admission_timeout / submission_unknown / POST 网络或 5xx | 受理结果未知；有 analysis_id 则查询，没有则保存原键和原完整参数，用户明确要求恢复后才原样重放 |
| task_timeout / task_interrupted | 停止自动执行；查询原任务，失败不证明上游未执行或无成本，不能自动换键重跑 |
| GET 网络或 5xx | 间隔重试最多 2 次，仍失败保留任务 ID |
| 404 | `analysis_id` / 关键帧不存在、过期或不属于当前 KeyB |

错误内容只转述用户可行动的信息，不暴露对象键、签名 URL、内部脚本错误、
模型供应商密钥或服务端堆栈。
