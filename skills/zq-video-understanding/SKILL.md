---
name: zq-video-understanding
description: 上传用户视频到平台，异步生成结构化视觉分析，并由平台脚本按模型选出的时间点提取 6 至 10 张真实关键帧。
---
<!-- zq-skills: zq-video-understanding v1.1.3 target=claude-code -->

# 视频理解与平台关键帧提取

把用户提供的视频通过预签名地址直传平台对象存储，启动异步分析，轮询结果，
再下载平台脚本从原视频提取的关键帧。不要在客户端自行猜测时间点，也不要把
平台内部提示词、模型路由或脚本内容复述给用户。

## 何时使用 / 何时不使用

- 使用：用户希望理解、总结一个 `.mp4`、`.mov` 或 `.webm` 视频，或希望
  获取 6–10 张有代表性的真实画面。
- 不使用：实时视频流、超过 150MB 的视频、纯剪辑/转码任务，以及要求完整
  对白转写或声音分析的任务。当前 Demo 使用抽样画面做视觉分析，不处理音轨。

## API key 配置

所有平台请求都使用项目为该用户签发的 KeyB：
`Authorization: Bearer <ZQ_API_KEY>`。KeyB 以 `sk-` 或 `sk_` 开头，只允许操作创建
它的同一项目与用户数据。

1. 优先从 `~/.config/zq-skills/credentials` 读取 `ZQ_API_KEY`；也可使用当前
   会话安全注入的环境变量。
2. 若未配置，提示用户向接入项目领取 KeyB，或使用 `zq-config` 完成配置；
   不得要求用户把 KeyA 或对象存储密钥交给你。
3. 不回显完整 KeyB，不把它写入仓库、日志、命令参数或交付文件。
4. KeyB 有效性由首次业务请求验证，返回 401 时再引导用户更新 KeyB。

## 你需要向用户收集的物料

| 物料 | 说明 | 必须 |
| --- | --- | --- |
| 视频文件 | `.mp4` / `.mov` / `.webm`，1 字节至 150MB | 是 |
| 截图数量 | 6–10，默认 8 | 否 |
| 结果语言 | `中文` 或 `English`，默认中文 | 否 |
| 视觉关注点 | 例如“重点看商品结构变化”；最长 1000 字符 | 否 |

本地读取文件名、字节数和扩展名，并按以下映射确定 `content_type`：
`.mp4 → video/mp4`、`.mov → video/quicktime`、`.webm → video/webm`。

## 开始前提问

只询问用户尚未给出的可选项。用户没有偏好时直接采用 `shot_count=8`、
`analysis_lang=中文`，不要把平台内部供应商选择暴露成必答问题。开始上传前
明确告知：原视频在分析结束后删除；输出关键帧可能按部署配置经 CDN 提供。

## 执行流程

API 根地址使用 `https://skills-platform-api-dev.zhiqingresearch.com`。平台请求带 KeyB；预签名 PUT 请求绝不能
携带 KeyB。为视频分析生成一个 8–128 字符的稳定 `Idempotency-Key`，网络
重试时必须复用，避免重复任务。

### 1. 登记并直传视频

```http
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/files
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

### 2. 创建异步分析

```http
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/video/analysis
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

HTTP 202 返回 `analysis_id`、`status` 和 `billed`。Demo 中
`billed.state=quoted` 只表示报价展示，不能向用户声称积分已经冻结或扣除。

### 3. 轮询并获取结果

每隔至少 5 秒查询一次：

```http
GET https://skills-platform-api-dev.zhiqingresearch.com/v1/video/analysis/{analysis_id}
Authorization: Bearer <ZQ_API_KEY>
```

- `queued` / `processing`：继续等待；
- `completed`：读取 `result` 和 `issues`；
- `failed` / `cancelled`：停止轮询并转述 `issues`；
- 超过约 10 分钟仍未终态：保存 `analysis_id`，提示用户稍后继续查询，
  不要高频空转。

`result.shots[].time` 是模型选择的原视频秒数，`frame_url` 是平台已按该时间点
执行抽帧脚本得到的 JPEG 入口，一律返回以 `/` 开头的**相对路径**（如
`/v1/video/analysis/{analysis_id}/frames/frame-01.jpg`），不是完整 URL。
不要再次在本地抽帧。

### 4. 下载关键帧

`frame_url` 是相对路径，不能直接交给 `curl`/`wget` 等下载工具（缺少主机名，
必然失败）。必须先拼接平台根地址得到完整 URL 再请求：

```text
完整地址 = https://skills-platform-api-dev.zhiqingresearch.com + frame_url
示例     = https://skills-platform-api-dev.zhiqingresearch.com/v1/video/analysis/analysis_abc123/frames/frame-01.jpg
```

拼接规则：`frame_url` 自带开头的 `/`；`API_BASE_URL` 末尾若带有 `/`，先去掉
再拼接，避免路径出现 `//`。对每个完整地址发带 KeyB 的 GET。平台校验归属后
返回 HTTP 302，跳转到 CDN 或短期签名地址；跟随跳转保存 JPEG 文件。跟随
跨域跳转时不要把 KeyB 转发给目标存储域名。

## 交付

向用户交付：结构化摘要、关键词、人物、物体、场景时间轴、按顺序命名的
关键帧，以及 `usage` 中实际返回的模型、token 和费用信息。`issues` 必须一并
转述；尤其不能隐瞒 `sampled_visual_analysis`，它表示当前 Demo 只基于候选帧
做视觉分析，音轨或采样间瞬时画面可能未被覆盖。

## 出错处理

| 状态 | 处理 |
| --- | --- |
| 401 | KeyB 缺失、失效或不属于当前项目用户；引导重新领取/配置 |
| 409 | 同一 `Idempotency-Key` 被用于不同参数；复用原参数或换新的业务键 |
| 422 | 按 `field_errors` 修正文件类型、大小、上传状态或分析参数 |
| 429 | 当前 KeyB 并发任务已达上限；等待已有任务终态后重试 |
| 503 / 网络错误 | 间隔重试最多 2 次；分析 POST 必须复用同一幂等键 |
| 404 | `analysis_id` / 关键帧不存在、过期或不属于当前 KeyB |

错误内容只转述用户可行动的信息，不暴露对象键、签名 URL、内部脚本错误、
模型供应商密钥或服务端堆栈。
