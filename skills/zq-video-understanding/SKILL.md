---
name: zq-video-understanding
description: 对用户提供的视频做多模态理解（画面、对白、声音联合分析），交付结构化分析（摘要、关键词、人物、物体、场景时间轴）、按时间戳从原视频提取的代表性截图（ZIP 打包）与本次用量费用摘要。当用户给出视频文件、要求理解/分析/总结视频内容或提取关键画面时使用。
---
<!-- zq-skills: zq-video-understanding v0.2.1 target=claude-code -->

# 视频理解与关键截图提取

把用户提供的视频交给平台做多模态理解，按平台返回的时间戳从**原视频**
提取真实帧（非生成式图片），打包交付。本剧本只负责收集物料、转发
问题、跟随平台指令；理解提示词、选点策略、模型接入均在平台侧。

## 何时使用 / 何时不使用

- 使用：用户提供视频文件（≤150MB），想要内容理解/总结、结构化分析
  （人物/物体/场景时间轴）、或提取 6–10 张代表性截图；
- 不使用：只要本地剪辑/转码/压缩；图片（非视频）的理解；实时流分析。

## 首次使用引导：API key 配置（安装后第一次运行时必须先做）

本 skill 调用平台能力并按次计费，**key 未配置前不得开始任何流程步骤、
不得发起任何计费动作**。首次被加载或被调用时，先做以下检查：

1. 读取 `~/.config/zq-skills/credentials` 中的 `ZQ_API_KEY`
   （沙箱环境读不到本地文件，见第 4 条）。
2. 未配置 → **先向用户原样输出下面的引导文案**，等用户完成配置并
   明确继续后再开始流程：

   > 【首次使用提示】本 skill 调用 zq-skills 平台能力，按次消耗积分，
   > 使用前需配置 API key（约 1 分钟）：
   > 1. 到 Skill 市场账号页领取 API key（`sk-` 或 `sk_` 开头）；
   > 2. 桌面端：安装 `zq-config` skill 后对我说"配置 key"；或手动写入
   >    `~/.config/zq-skills/credentials`（权限 600，一行
   >    `ZQ_API_KEY=sk-...`）；
   > 3. 网页/沙箱端：直接把 key 发给我，我在本次会话中使用（不会回显）。
   > 配置完成后对我说"继续"。未配置前我不会开始流程，也不会产生扣费。

3. 已配置 → 用 `GET https://skills-platform-api-dev.zhiqingresearch.com/v1/account/balance` 验证
   （200 即有效，顺带向用户展示余额），然后进入流程。
4. 沙箱环境（网页 agent 会话内，读不到本地文件）：key 由用户在会话中
   提供一次，保存在会话内存中传给脚本/请求，禁止回显到输出。
5. 任何情况下不得把 key 完整展示在对话中、写进代码或仓库。
6. 更完整的配置/校验/轮换/删除，引导用户安装 `zq-config` skill。

## 你需要向用户收集的物料

| 物料 | 说明 | 必须 |
| --- | --- | --- |
| 视频文件 | 常见格式（mp4/mov/webm），≤150MB；超限先引导压缩 | 是 |
| 分析偏好 | 供应商、截图数量、输出语言、关注点（见 intake） | 否（有默认） |

## 开始前：向用户提问（intake）

按 `intake/questions.yaml` 初始题集问齐偏好（供应商 / 截图数量 /
输出语言 / 关注点）——直连模式下没有服务端动态追问，以本题集为准，
用户跳过的按默认值提交。转述问题保持原意，不要替用户作答。

## 执行流程

两条铁律：

1. 每次"提交分析"生成一次 `Idempotency-Key`（uuid）：同一分析的
   登记与重试**复用同一个 key**（重放返回首次结果，不会重复扣费）；
   开始新的分析才换新 key；
2. 严格按下列序列调用，不自创步骤、不编造参数。

调用序列（均带 `Authorization: Bearer <ZQ_API_KEY>`）：

```
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/files                          上传登记（返回 file_ref + 预签名 URL）
PUT  {预签名 URL}                                        上传视频原文件（一次性地址，限时有效）
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/video/analysis                  提交分析（异步 202 + analysis_id，此处计费）
GET  https://skills-platform-api-dev.zhiqingresearch.com/v1/video/analysis/{analysis_id}    轮询直到 completed / failed
```

执行要点（本 skill 特有）：

- `/v1/files` 登记体：`{skill_id: zq-video-understanding, filename,
  content_type, size_bytes}`——平台按 file_policy 校验（mp4/mov/webm、
  ≤150MB），422 时原样转述字段提示并引导压缩或换格式；
- 分析提交体：`{file_ref, provider, shot_count, analysis_lang,
  extra_focus}`，偏好取自 intake 答案（未答按默认：auto / 8 / 中文 / 空）；
- 轮询间隔 5–10 秒；超过约 10 分钟仍未完成：告知用户稍后凭
  analysis_id 查询，不要空转；
- 用户要求中止：说明已受理的分析无法中止（会继续执行并计费）；
  尚未提交的（还在收集物料/问答阶段）直接停止即可，不产生费用；
- **抽帧执行模式自检（轮询到 completed 后）**：拿到结果的 shots
  时间戳后——若你能运行本地脚本且本 skill 附带
  `client/scripts/extract_frames.py`，在执行环境安装依赖
  （`client/scripts/requirements.txt`）后对**用户原始视频**按时间戳
  抽帧（文件路径由调用方传入，凭据经环境变量）；若不能（网页沙箱
  无法装依赖），使用结果中平台生成的截图下载链接，无需本地抽帧。

## 交付

交付说明与话术见 `deliverables.md`（渲染安装包时会并入本文）。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 | key 无效或过期 → 回到"首次使用引导"重新配置 |
| 402 | 积分不足 → 指引用户到市场充值后重试 |
| 404 | analysis_id 不存在或已过期 → 确认 id；已过期的重新提交分析 |
| 409 | 幂等重放 → 沿用首次响应继续，同一分析不要换 key 重提 |
| 422 | 输入不合法（如格式/大小不符）→ 原样转述字段级提示，不解释规则来源 |
| 5xx / 网络 | 间隔重试 ≤2 次（写请求有幂等键保护），仍失败告知稍后再试 |

除错误提示外，不要把原始 JSON 响应整段丢给用户，转述要点即可。
