---
name: zq-amazon-product-video
description: 根据商品图片与资料生成与真实商品结构一致的 Amazon 商品广告视频：平台完成素材分析、缺失视角补图、多图参考视频生成与商品一致性质检，剧本在会话内组织素材、事实、提示词、成片四个确认门并渐进返修。当用户要为商品制作广告视频、商品页视频素材时使用。
---
<!-- zq-skills: zq-amazon-product-video v0.1.0 target=claude-code -->

# Amazon 商品广告视频生成

把用户提供的商品图片与资料交给平台的分析、补图、生成与质检能力，
产出与真实商品结构一致的商品广告视频。本剧本只负责收集物料、按
序列调用能力端点、在会话内组织四个确认门和交付；素材审核、商品
事实表、视频提示词与返修约束全部由平台生成，剧本不包含、也不得
自行组装任何提示词与内部规则。

## 何时使用 / 何时不使用

- 使用：用户为 Amazon（或类似电商）商品页制作广告视频——能提供商品
  图片（至少 1 张，建议 ≥5 张）与商品名称、卖点，并确定时长/画幅等
  规格；
- 不使用：只要本地剪辑/转码/压缩；对已有视频做理解分析或截图
  （用 zq-video-understanding）；无商品实拍素材的纯创意视频。

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
   提供一次，保存在会话内存中传给请求，禁止回显到输出。
5. 任何情况下不得把 key 完整展示在对话中、写进代码或仓库。
6. 更完整的配置/校验/轮换/删除，引导用户安装 `zq-config` skill。

## 你需要向用户收集的物料

| 物料 | 说明 | 必须 |
| --- | --- | --- |
| 商品图片 | jpg/jpeg/png/webp，单张 ≤10MB；至少 1 张，建议 ≥5 张且覆盖正面、两侧、背面/侧面、部件特写等关键视角（越全越好） | 是 |
| 商品名称 | 用于识别商品与包装身份 | 是 |
| 商品卖点 | 一条或多条，将进入画面表达 | 是 |
| 商品类目 / 属性 | 可选；缺省由平台识别 | 否 |
| 视频规格 | 时长（10/15/20/30 秒）、画幅（16:9/9:16/1:1）；分辨率、语言、音频见 intake 默认值 | 是 |

## 开始前：向用户提问（intake）

按 `intake/questions.yaml` 初始题集问齐信息——直连模式下没有服务端
动态追问，以本题集为准，用户跳过的按默认值提交。转述问题保持原意，
不要替用户作答。

## 执行流程

两条铁律：

1. 每次"提交任务"生成一次 `Idempotency-Key`（uuid）：同一任务的
   登记与重试**复用同一个 key**（重放返回首次结果，不会重复扣费）；
   开始新任务才换新 key；
2. 严格按下列序列调用，不自创步骤、不编造参数。

调用序列（均带 `Authorization: Bearer <ZQ_API_KEY>`，直连通道）：

```
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/files            逐张登记商品图片
                                          （skill_id=zq-amazon-product-video；
                                          返回 file_ref + 预签名 URL）
PUT  {预签名 URL}                          上传原图（一次性地址，限时有效）
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/product-video/analysis     输入分析：素材覆盖+事实表+
                                                    提示词草稿（202 + analysis_id，
                                                    此处计费）
GET  https://skills-platform-api-dev.zhiqingresearch.com/v1/product-video/analysis/{analysis_id}   轮询直到 completed / failed
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/image/generation            缺失视角补图（仅素材不足时；
                                                    202 + generation_id）
GET  https://skills-platform-api-dev.zhiqingresearch.com/v1/image/generation/{generation_id}       轮询直到 completed / failed
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/video/generation            视频生成（202 + generation_id）
GET  https://skills-platform-api-dev.zhiqingresearch.com/v1/video/generation/{generation_id}       轮询直到 completed / failed
POST https://skills-platform-api-dev.zhiqingresearch.com/v1/product-video/qc            双质检+返修建议（202 + qc_id）
GET  https://skills-platform-api-dev.zhiqingresearch.com/v1/product-video/qc/{qc_id}    轮询直到 completed / failed
```

### 四个确认门（会话内逐一执行，任何一门不得跳过）

1. **素材确认门**：向用户逐张展示原图并转述素材检查摘要（可用/不可用
   及原因、已覆盖与缺失视角）。确认图不足 5 张或关键视角缺失时，先
   建议用户补充真实拍摄图；用户同意使用 AI 补图才调用补图能力。AI
   补充图必须明确标注"AI 生成"，逐张请用户确认；用户拒绝时，把用户
   指出的可观察错误（如"接口数量错误"）**原样**作为 `feedback`、连同
   `retry_of` 重新提交——不得自行改写、概括或组装补图提示词。**未获
   用户确认的图片不得计入生成用的 file_refs。**
2. **事实确认门**：精简转述商品事实表（重点标出冲突项与不允许进入
   广告的高风险宣称），请用户批准、修改或删除；用户修改原样记录，
   不替用户补写或取舍。
3. **提示词确认门**：完整展示平台生成的视频提示词、采用的卖点和追加
   约束，请用户批准；用户提出修改时，把修改应用到提示词文本后**再次
   展示全文**请用户批准，确认无误才可作为 `prompt` 提交生成。
4. **成片确认门**：展示成片下载链接与质检摘要。通过 → 交付；不通过
   → 逐条转述问题与平台的返修建议，用户批准 `revised_prompt`（或
   分镜草稿）后才以其**原文**回到生成步骤；同一返修链 ≥3 轮仍失败时，
   转述平台给出的降级建议并停止自动重试。

### 本 skill 特有的执行要点

- **计费透明**：各能力独立按次计费（输入分析、补图、视频生成、质检
  单价不同，视频生成最贵）。提交第一个计费动作前，向用户说明全程
  大致费用构成，由用户确认开始；
- **上传预检**：登记前确认每张图可读、扩展名与大小符合物料要求；
- **轮询节奏**：补图 ≥3 秒/次，分析与质检 ≥5 秒/次，视频生成 ≥10
  秒/次；视频生成轮询约 20 分钟仍未完成时，告知用户稍后凭
  generation_id 查询，不要空转；
- **任务不可中止**：已受理的异步任务无法取消，计费以平台结算为准；
  用户要求停止时，停止提交新任务并说明；
- **返修不换目标**：返修只针对失败镜头对应的约束与提示词（平台已
  生成建议），不要在无失败证据时主动堆叠新要求，也不要无限重复
  同一个失败请求。

### 结果的后处理（执行模式自检，按顺序判断）

本 skill 无胶水脚本，交付物即平台产物：

1. 能下载文件（本机或会话沙箱均可出网）→ 把成片与过程记录下载到
   工作区供用户直接取用；
2. 不能 → 向用户转述结果中的短时效下载链接（24 小时有效，过期凭
   对应任务 id 重取）。

## 交付

交付说明与话术见 `deliverables.md`（渲染安装包时会并入本文）。

## 出错处理

| 情况 | 处理 |
| --- | --- |
| 401 | key 无效或过期 → 回到"首次使用引导"重新配置 |
| 402 | 积分不足 → 指引用户到市场充值后重试（可告知还差哪一步） |
| 404 | 任务 id 不存在或已过期 → 产物链接过期时凭 id 重取；任务本身过期则重新提交 |
| 409 | 幂等重放/状态冲突 → 沿用同一 Idempotency-Key 的首次结果，不要换 key 重发 |
| 422 | 输入不合法（如图片不符物料要求、枚举值越界）→ 把平台返回的字段级提示原样转述，不解释规则来源 |
| 5xx / 网络 | 间隔重试 ≤2 次（写请求有幂等键保护），仍失败告知稍后再试 |

除错误提示外，不要把原始 JSON 响应整段丢给用户，转述要点即可。
