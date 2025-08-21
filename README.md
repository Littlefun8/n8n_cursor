# 项目：n8n 批改高考英语读后续写（手写作答）Agent

### 一、简介
该工作流实现：
- 从 Supabase 存储或飞书云盘批量获取手写答题卡图片；
- 通过 GLM-4V（或你配置的视觉模型）进行英文手写 OCR；
- 基于高考读后续写评分标准进行自动批改，输出分数与评分理由；
- 结果结构化为 JSON 并写入飞书多维表（Bitable）。

已提供可导入的 n8n 工作流：`workflows/gaokao_essay_grader.json`。

你也可以使用预填参数版本：
- `workflows/feishu_bitable_ensure_fields_prefilled.json`
- `workflows/gaokao_essay_grader_siliconflow_prefilled.json`

### 二、准备工作
1) n8n 环境变量（.env 或 n8n Variables）：
```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_BUCKET=
SUPABASE_PREFIX=

GLM_API_URL=https://open.bigmodel.cn/api/paas/v4/chat/completions
GLM_API_KEY=
GLM_VISION_MODEL=glm-4v

FEISHU_APP_ID=cli_a82f0cdab0f1901c
FEISHU_APP_SECRET=hO6sN2fiD5Gz43vxQkIj8eEHzjkvE85T
FEISHU_FOLDER_TOKEN=Z7RjfDX73ln7UhduYodcAyL2n4v          # 从飞书文件夹取图
FEISHU_BITABLE_APP_TOKEN=VS1ibKDeFaPjaxsfwFtcAlMjnQb     # 多维表 App Token
FEISHU_BITABLE_TABLE_ID=tblx1qE1V7WHiP2Q      # 多维表 Table ID

# 若使用 SiliconFlow（与你现有工作流一致）：
SILICONFLOW_API_URL=https://api.siliconflow.cn/v1/chat/completions
SILICONFLOW_API_KEY=sk-ikfyyvzeqlmpwifqgvvicplvypexmdwlzpnjjuwikfpvaqoc
SILICONFLOW_VISION_MODEL=THUDM/GLM-4.1V-9B-Thinking
```

说明：你使用的是社区节点，具体 Key/Token 请在对应节点凭证里自己填写；上面列的是运行所需参数清单，便于你核对。

2) 飞书多维表表头（中文）：
- 来源类型（文本）
- 文件标识（文本）
- 图片URL（文本或超链接）
- OCR文本（长文本）
- 总分（数字）
- 内容相关性（数字）
- 结构连贯（数字）
- 语言准确性（数字）
- 词汇丰富度（数字）
- 字数（数字）
- 评分说明-内容（长文本）
- 评分说明-结构（长文本）
- 评分说明-语言（长文本）
- 评分说明-总体（长文本）
- 审核状态（单选，默认“待审核”）
- 模型（文本）
- Rubric版本（文本）
- 置信度（数字 0-1）
- 创建时间（日期时间或文本）

3) Supabase 存储：将答题卡图片上传到对应 `bucket/prefix`；或将图片放在飞书 `FEISHU_FOLDER_TOKEN` 所指的文件夹下。

### 三、导入工作流
1) 打开 n8n → Import from file → 选择 `workflows/gaokao_essay_grader.json`；
2) 打开 `Config` 节点，可切换 `source_type` 为 `supabase` 或 `feishu`；
3) 运行 `Manual Trigger`，工作流将批量取图 → OCR → 批改 → 写入多维表。

或使用已预填 app_token/table_id 的工作流（你只需在节点里填入各自 Key）：
- 自动建表头：`workflows/feishu_bitable_ensure_fields_prefilled.json`
- 识别+批改+入表：`workflows/gaokao_essay_grader_siliconflow_prefilled.json`

### 四、评分与结构化输出
- 工作流内置 Rubric 版本：`gaokao_read_after_writing_v1.0`；
- 可在 `Config` 节点调整评分权重与总分：
  - weight_content, weight_structure, weight_language, weight_vocabulary
  - total_score_max（默认 25）
- 模型会被强制要求输出严格 JSON；若模型输出非 JSON，工作流会提取与修复，使其满足：
```
{
  "source_id": string,
  "total_score": number,
  "content_relevance": number,
  "structure_coherence": number,
  "language_accuracy": number,
  "vocabulary_richness": number,
  "word_count": number,
  "reasons": { "content": string, "structure": string, "language": string, "overall": string },
  "extracted_text": string,
  "rubric_version": string,
  "model_name": string,
  "confidence": number
}
```

建议的“批改”消息结构（SiliconFlow 文本对话）：
```json
{
  "model": "THUDM/GLM-4.1V-9B-Thinking",
  "messages": [
    { "role": "system", "content": "You are a strict grader for Chinese Gaokao English 'Read-after-writing' continuation tasks. Score out of 25 with weighted criteria. Output JSON only." },
    { "role": "user", "content": [ { "type": "text", "text": "Task: Grade the following OCR-ed handwritten essay. Return STRICT JSON only (no code fences, no extra text) following schema: {\n  \"source_id\": string,\n  \"total_score\": number,\n  \"content_relevance\": number,\n  \"structure_coherence\": number,\n  \"language_accuracy\": number,\n  \"vocabulary_richness\": number,\n  \"word_count\": number,\n  \"reasons\": { \"content\": string, \"structure\": string, \"language\": string, \"overall\": string },\n  \"extracted_text\": string,\n  \"rubric_version\": string,\n  \"model_name\": string,\n  \"confidence\": number\n}. Essay:\n{{ OCR_TEXT }}" } ] }
  ],
  "temperature": 0
}
```

OCR（SiliconFlow 视觉模型）请求体示例：
```json
{
  "model": "THUDM/GLM-4.1V-9B-Thinking",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "请帮我识别图片中的所有英文字符，只输出英文字符本身，不需要任何解释或额外信息。" },
        { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,{{ BASE64_DATA }}" } }
      ]
    }
  ],
  "temperature": 0
}
```

JSON 修复/校验（在函数节点内的示例 JS）：
```javascript
function tryParse(s){ try { return JSON.parse(s); } catch { return null; } }
let raw = $json.content?.trim?.() || $json.choices?.[0]?.message?.content?.trim?.() || '';
raw = raw.replace(/^```json[\r\n]+/i, '').replace(/```$/,'');
const i = raw.indexOf('{'); const j = raw.lastIndexOf('}');
const pick = (i>=0 && j>=i) ? raw.slice(i, j+1) : raw;
const data = tryParse(pick) || tryParse(raw) || {};
return [{ json: data }];
```

### 五、飞书多维表写入
- 工作流使用 Tenant Token 获取凭证，并调用 Bitable `batch_create` 写入记录；
- 如需批量写入优化，可将 `Split In Batches` 与聚合节点扩展（示例已留出占位节点）。

如果你希望自动创建/对齐多维表字段（而不是手动建表头），可导入：`workflows/feishu_bitable_ensure_fields.json`，设置：
- `FEISHU_APP_ID`、`FEISHU_APP_SECRET`
- `FEISHU_BITABLE_APP_TOKEN`、`FEISHU_BITABLE_TABLE_ID`
执行后，会自动创建缺失字段，目标表别名默认 `n8n_doc_output`（仅用于标识）。

最小化 API 调用（可用来自测）：
- 取租户 Token
```json
POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
Body: { "app_id": "YOUR_APP_ID", "app_secret": "YOUR_APP_SECRET" }
```
- 新增记录
```json
POST https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_create
Headers: Authorization: Bearer {tenant_access_token}
Body: { "records": [ { "fields": { "文件标识": "1.jpg", "OCR文本": "...", "总分": 20, "审核状态": "待审核" } } ] }
```

### 六、常见问题
1) 模型输出带 ``` 包裹：工作流包含清洗逻辑，会移除并重新解析。
2) OCR 不准：可换更高质量图片或增设超分辨/去噪节点；也可追加“(?)”不确定标记的再审步骤。
3) 表头不匹配：确保多维表字段中文名与上文一致；或在 `Fn Map -> Bitable Fields` 中调整映射。

### 七、后续人工审核与对齐优化
- 在多维表中新增“人工评分/复核意见/是否通过”等字段；
- 通过 n8n 监听表更新，驱动一个“对齐微调”流水线：收集模型评分与人评差异，自动回注提示词权重或阈值；
- 可将“待审核”状态下的记录推送至飞书群机器人提醒人工复核。

### 八、与您现有工作流对齐（SiliconFlow 版）
你提供的工作流关键节点流转：
- `List Files`（Supabase 列表，POST `/storage/v1/object/list/{bucket}` 带 `prefix`）→ `Loop Over Items`
- `Download Files`（GET 公网对象 URL，带 Bearer）→ `64base`（Code，将 Binary 转 Base64）
- `API`（HTTP Request 调 SiliconFlow `chat/completions`，模型 `THUDM/GLM-4.1V-9B-Thinking`，输入 data URL）

你现有的 OCR 请求体（可直接复用）：
```json
{
  "model": "THUDM/GLM-4.1V-9B-Thinking",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "请帮我识别图片中的所有英文字符，只输出英文字符本身，不需要任何解释或额外信息。" },
        { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,{{ $json.base64String }}" } }
      ]
    }
  ]
}
```

将你现有工作流接入本工程的“批改+写入多维表”下游，建议两种方式：
- 方式 A（复用本工程下游）：在你的 `API`（OCR）节点之后，新增一个“批改”HTTP 节点，使用文本模型（可仍用 `THUDM/GLM-4.1V-9B-Thinking` 作为纯文本 LLM）并传入 OCR 文本，返回严格 JSON；再接入 README 第五节的多维表写入节点（或导入本仓库工作流，替换 OCR 节点）。
- 方式 B（直接替换）：导入本仓库的 `workflows/gaokao_essay_grader.json`，把其中 `GLM-4V OCR` 替换为你的 SiliconFlow OCR 节点，并保持后续“批改→JSON 修复→Bitable 写入”不变。

示例“批改”请求体（SiliconFlow 文本对话，强制 JSON 输出）：
```json
{
  "model": "THUDM/GLM-4.1V-9B-Thinking",
  "messages": [
    {
      "role": "system",
      "content": "You are a strict grader for Chinese Gaokao English 'Read-after-writing' continuation tasks. Score out of 25 with weighted criteria. Output JSON only."
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Task: Grade the following OCR-ed handwritten essay. Return STRICT JSON only (no code fences, no extra text) following schema: {\n  \"source_id\": string,\n  \"total_score\": number,\n  \"content_relevance\": number,\n  \"structure_coherence\": number,\n  \"language_accuracy\": number,\n  \"vocabulary_richness\": number,\n  \"word_count\": number,\n  \"reasons\": { \"content\": string, \"structure\": string, \"language\": string, \"overall\": string },\n  \"extracted_text\": string,\n  \"rubric_version\": string,\n  \"model_name\": string,\n  \"confidence\": number\n}. Essay:\n{{ $json.ocr_text }}"
        }
      ]
    }
  ],
  "temperature": 0
}
```

若模型偶发返回非严格 JSON，可在“批改”节点后添加一个 Code 节点做 JSON 修复（与你现在的 `pinData` 现象一致）：
```javascript
// 输入：json.content 为 LLM 原始返回文本
function tryParse(s){ try { return JSON.parse(s); } catch { return null; } }
let raw = $json.content?.trim?.() || '';
raw = raw.replace(/^```json[\r\n]+/i, '').replace(/```$/,'');
const i = raw.indexOf('{'); const j = raw.lastIndexOf('}');
const slice = (i>=0 && j>=i) ? raw.slice(i, j+1) : raw;
const data = tryParse(slice) || tryParse(raw) || {};
return [{ json: data }];
```

然后按本 README 的“第五节”映射字段写入飞书多维表即可。

补充：你还提供了一个 `Google Gemini Chat Model` 节点，若需要也可作为“批改”文本 LLM 的替代；只需保证提示词强制“仅输出 JSON”，并在下游做上述 JSON 修复与校验。

参考：飞书页面（用于确认 table 参数即 table_id）：
- [飞书页面（Wiki 嵌入）](https://qcnnkxsuz7mq.feishu.cn/wiki/M7GZwNGJmiyxXikApeUcbP0mnbf)
- [飞书多维表页面（含 app_token 与 table_id）](https://qcnnkxsuz7mq.feishu.cn/base/VS1ibKDeFaPjaxsfwFtcAlMjnQb?table=tblx1qE1V7WHiP2Q&view=vewVVUV8Cx)

### 九、按你的“社区节点”使用方式的流程与参数（精简）
- 输入来源（2 选 1）
  - Supabase：`SUPABASE_URL`、`SUPABASE_BUCKET`、`SUPABASE_PREFIX`（列表/下载用），`SUPABASE_SERVICE_ROLE_KEY`（若需签名/管理接口）
  - 飞书云盘：`folder_token`（可选）或你自己编排的取图方式
- OCR（视觉模型）
  - SiliconFlow：`SILICONFLOW_API_KEY`、`SILICONFLOW_API_URL`、`SILICONFLOW_VISION_MODEL`
  - 或 GLM 官方接口：`GLM_API_KEY`、`GLM_API_URL`、`GLM_VISION_MODEL`
- 批改（文本模型）
  - 复用上面模型或替换为你习惯的 LLM（提示词见“第四节”）
- JSON 修复
  - 用上方 JS 片段清洗模型输出，确保严格 JSON
- 写入飞书多维表
  - `FEISHU_APP_ID`、`FEISHU_APP_SECRET`→ 换取 `tenant_access_token`
  - `FEISHU_BITABLE_APP_TOKEN`（base_ 开头）与 `FEISHU_BITABLE_TABLE_ID`
  - 字段名需与“第二节-表头”一致（或改映射）

备注：你可以完全使用社区节点，自行在节点的 Credential 中填写各类 Keys/Token，上述参数仅作为核对清单与请求模板参考。

### 十、从零搭建：节点与严格数据契约（飞书云盘版）
- 目标：从飞书云盘批量取图片 → 模型 OCR → 模型批改 → 严格 JSON → 写入飞书多维表。
- 要求：每个节点输入/输出契约一致，字段命名固定；失败可定位到具体节点。

1) `Manual Trigger`
- 功能：手动启动，无输入/输出约束。

2) `Config`（Set 节点）
- 作用：集中出参，供后续所有节点引用。
- 输出契约（单条）：
```json
{
  "feishu_folder_token": "${FEISHU_FOLDER_TOKEN}",
  "bitable_app_token": "${FEISHU_BITABLE_APP_TOKEN}",
  "bitable_table_id": "${FEISHU_BITABLE_TABLE_ID}",
  "rubric_version": "gaokao_read_after_writing_v1.0",
  "model_name": "${SILICONFLOW_VISION_MODEL}",
  "siliconflow_api_url": "${SILICONFLOW_API_URL}",
  "audit_status_default": "待审核",
  "total_score_max": 25,
  "weight_content": 0.4,
  "weight_structure": 0.25,
  "weight_language": 0.25,
  "weight_vocabulary": 0.1
}
```

3) `Feishu Tenant Token`（HTTP Request）
- 请求：POST `open-apis/auth/v3/tenant_access_token/internal`
- 输入：来自 `Config`
- 输出契约：
```json
{ "tenant_access_token": "...", "expire": 7200 }
```

4) `Feishu List Files`（HTTP Request）
- 请求：GET `open-apis/drive/v1/files?folder_token={{feishu_folder_token}}`
- 头：`Authorization: Bearer {{tenant_access_token}}`
- 输出（原始）：`{ data: { files: [ { token, name, ... } ] } }`

5) `Fn Normalize Files`（Function）
- 作用：将文件列表标准化为逐项图片条目。
- 输入：上一步原始输出
- 输出契约（多条）：
```json
{
  "source_type": "feishu",
  "file_token": "FILE_TOKEN",
  "file_name": "IMG_001.jpg",
  "tenant_access_token": "..."
}
```

6) `Feishu Download Image`（HTTP Request）
- 请求：GET `open-apis/drive/v1/files/{{file_token}}/download`
- 头：`Authorization: Bearer {{tenant_access_token}}`
- 输出：二进制 `data`

7) `Binary -> Base64`（Move Binary Data）
- 输入：二进制
- 输出契约：
```json
{ "data": "<BASE64>" }
```

8) `Fn To Data URL`（Function）
- 作用：拼接 `data:image/*;base64,` 供视觉模型使用。
- 输出契约：
```json
{
  "file_name": "IMG_001.jpg",
  "image_data_url": "data:image/jpeg;base64,...."
}
```

9) `SiliconFlow OCR`（HTTP Request）
- URL：`{{siliconflow_api_url}}`
- 头：`Authorization: Bearer ${SILICONFLOW_API_KEY}`
- 请求体示例：
```json
{
  "model": "{{model_name}}",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "请帮我识别图片中的所有英文字符，只输出英文字符本身，不需要任何解释或额外信息。" },
        { "type": "image_url", "image_url": { "url": "{{image_data_url}}" } }
      ]
    }
  ],
  "temperature": 0
}
```
- 输出（原始）：`{ choices: [ { message: { content: "...ocr text..." } } ] }`

10) `Fn Extract OCR`（Function）
- 作用：抽取纯文本 OCR 结果，绑定源文件名。
- 输出契约：
```json
{ "ocr_text": "...", "file_name": "IMG_001.jpg" }
```

11) `SiliconFlow Grade JSON`（HTTP Request）
- URL：`{{siliconflow_api_url}}`
- 头：同上
- 请求体（将你的评分标准 Prompt 放入 system 或 user 中的 text 段）：
```json
{
  "model": "{{model_name}}",
  "messages": [
    { "role": "system", "content": "You are a strict grader for Chinese Gaokao English 'Read-after-writing' continuation tasks. Score out of {{total_score_max}} with weighted criteria. Output JSON only." },
    { "role": "user", "content": [ { "type": "text", "text": "Task: Grade the following OCR-ed handwritten essay. Return STRICT JSON only (no code fences, no extra text) following schema: {\n  \"source_id\": string,\n  \"total_score\": number,\n  \"content_relevance\": number,\n  \"structure_coherence\": number,\n  \"language_accuracy\": number,\n  \"vocabulary_richness\": number,\n  \"word_count\": number,\n  \"reasons\": { \"content\": string, \"structure\": string, \"language\": string, \"overall\": string },\n  \"extracted_text\": string,\n  \"rubric_version\": string,\n  \"model_name\": string,\n  \"confidence\": number\n}. Essay:\n{{ ocr_text }}" } ] }
  ],
  "temperature": 0
}
```
- 输出（原始）：`{ choices: [ { message: { content: "<should be JSON>" } } ] }`

12) `Fn Ensure JSON`（Function）
- 作用：移除 ``` 包裹，截取花括号片段并 JSON.parse，补全默认值与边界裁剪。
- 输出严格契约（单条）：
```json
{
  "source_id": "IMG_001.jpg",
  "total_score": 22.5,
  "content_relevance": 9.0,
  "structure_coherence": 6.0,
  "language_accuracy": 5.5,
  "vocabulary_richness": 2.0,
  "word_count": 185,
  "reasons": {
    "content": "...",
    "structure": "...",
    "language": "...",
    "overall": "..."
  },
  "extracted_text": "<ocr_text>",
  "rubric_version": "gaokao_read_after_writing_v1.0",
  "model_name": "THUDM/GLM-4.1V-9B-Thinking",
  "confidence": 0.78
}
```

13) `Fn Map -> Bitable Fields`（Function）
- 作用：与多维表字段严格映射。
- 输入：上一步严格 JSON + `Config`
- 输出契约（写表所需）：
```json
{
  "fields": {
    "来源类型": "feishu",
    "文件标识": "IMG_001.jpg",
    "图片URL": "",
    "OCR文本": "<extracted_text>",
    "总分": 22.5,
    "内容相关性": 9.0,
    "结构连贯": 6.0,
    "语言准确性": 5.5,
    "词汇丰富度": 2.0,
    "字数": 185,
    "评分说明-内容": "...",
    "评分说明-结构": "...",
    "评分说明-语言": "...",
    "评分说明-总体": "...",
    "审核状态": "待审核",
    "模型": "THUDM/GLM-4.1V-9B-Thinking",
    "Rubric版本": "gaokao_read_after_writing_v1.0",
    "置信度": 0.78,
    "创建时间": "2024-01-01T10:00:00.000Z"
  }
}
```

14) `Feishu Bitable Create`（HTTP Request）
- 请求：POST `open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_create`
- 头：`Authorization: Bearer {{tenant_access_token}}`
- 请求体：
```json
{ "records": [ { "fields": { /* 同上 fields */ } } ] }
```

15) 可选：`Feishu Bitable Ensure Fields`
- 若未提前建表或字段不齐，需先对齐字段。所需字段与“第二节-表头”一致（类型：文本/数字/单选）。

16) 审核与对齐建议（可后续扩展）
- 在表中新增：`人工总分`、`人工意见`、`是否通过`、`AI-人评差值`。
- 新建监听工作流：当人工字段更新时，写回差值与统计；用于迭代提示词与权重。

### 十二、流程概览（节点与数据流）
下图概括了节点职责与关键数据在各节点之间的传递格式（仅列核心字段）。

```mermaid
flowchart LR
  A[Manual Trigger]
  B[Config\nfeishu_folder_token / app_token / table_id\nmodel_name / rubric / ...]
  C[Feishu Tenant Token\n输出: tenant_access_token]
  D[Feishu List Files\n输出: data.files[]]
  E[Fn Normalize Files\n输出: {file_token, file_name, token}]
  F[Feishu Download Image\n输出: (binary data)]
  G[Binary -> Base64\n输出: {data: <base64> }]
  H[Fn To Data URL\n输出: {image_data_url, file_name}]
  I[SiliconFlow OCR\n输出: choices[0].message.content]
  J[Fn Extract OCR\n输出: {ocr_text, file_name}]
  K[SiliconFlow Grade JSON\n输入: ocr_text / 关键词 / 背景\n输出: 严格JSON(新版契约)]
  L[Fn Ensure JSON\n输出: {meta, scores, feedback, ocr_text}]
  M[Fn Map -> Internal Fields\n输出: fields(全量)]
  N[Feishu Bitable batch_create]
  V[用户视图(仅关键列)]

  A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K --> L --> M --> N --> V
```

关键字段沿线传递（缩写）：
- 文件发现：`{ file_token, file_name }`
- OCR：`{ image_data_url } → ocr_text`
- 评分：`{ ocr_text } → { meta, scores, feedback }`
- 写表：`{ fields }`（内部全量），用户视图只展示关键列（见下一节）。

### 十三、最小可见输出字段（面向用户）
为便于教师/家长阅读，推荐只展示必要信息；其余元数据保留在表中但可隐藏或放入“内部视图”。

1) 建议“用户视图”仅展示以下列：
- 文件标识（必选）
- 词数是否达标（是/否，来自 `meta.meets_length_requirement`）
- 关键词使用（`keywords_used/keywords_required`）
- 内容/语言/篇章/总分（`scores.content/language/organization/total`）
- 档位（可选：`scores.band`）
- OCR文本（可选，较长可隐藏或切换显示）

2) 面向用户的最小 JSON（评分节点可直接产出或由清洗节点裁剪）：
```json
{
  "words": 185,
  "meets_length_requirement": true,
  "keywords_used": 3,
  "keywords_required": 5,
  "scores": {
    "content": 9,
    "language": 6,
    "organization": 6,
    "total": 21,
    "band": "第六档"
  },
  "ocr_text": "..."
}
```

3) 将“内部字段”与“用户可见字段”同时保留的方法
- 仍按“十一”节写入全量字段，便于追溯与对齐；
- 在多维表创建两个视图：`内部视图`（全列）、`用户视图`（仅关键列）。

4) 可选：新增“评分摘要”字段（便于快速浏览）
```javascript
// 在映射节点中追加：
const g = $json; // 严格结果 {meta, scores, ...}
const words = g.meta?.words ?? 0;
const used = g.meta?.keywords_used ?? 0;
const reqd = g.meta?.keywords_required ?? 0;
const ok = g.meta?.meets_length_requirement ? '达标' : '不足';
const s = g.scores || {};
const band = s.band ? ` | ${s.band}` : '';
const summary = `${s.total}/25 | C:${s.content}/10 L:${s.language}/8 O:${s.organization}/7 | 词数${words}${ok} | 关键词 ${used}/${reqd}${band}`;

return [{ json: { fields: {
  // ...已有字段,
  '评分摘要': summary
}} }];
```

5) 如果要彻底精简写入字段
- 你也可以仅写入下列字段，其他全部不写（或后续逐步加回）：
  - 文件标识、词数是否达标、关键词使用（used/required 文本）、内容/语言/篇章/总分、档位（可选）、OCR文本（可选）、评分摘要（可选）
- 相应映射示例：
```javascript
const g = $json; const src = $items(0,0).json; const s = g.scores || {}; const m = g.meta || {};
return [{ json: { fields: {
  '文件标识': src.file_name || '',
  '词数是否达标': m.meets_length_requirement ? '是' : '否',
  '关键词使用': `${m.keywords_used || 0}/${m.keywords_required || 0}`,
  '内容': s.content,
  '语言': s.language,
  '篇章': s.organization,
  '总分': s.total,
  '档位': s.band || '',
  'OCR文本': g.ocr_text || ''
}} }];
```

这样即可在界面上呈现最关键的评分依据信息，同时保留扩展空间。


### 十一、新版评分 JSON 契约与飞书多维表映射（推荐）
为贴合阅卷实际，将评分维度统一为“内容(0-10)/语言(0-8)/篇章(0-7)”，并提供完整元信息与反馈结构。以下为严格 JSON 契约与字段映射，按此创建节点即可。

1) 模型输出 JSON 契约（严格）
```json
{
  "meta": {
    "words": 185,
    "keywords_required": 5,
    "expected_keywords": ["rescue", "concert", "volunteer", "accident", "friendship"],
    "keywords_used": 3,
    "keywords_used_list": ["rescue", "volunteer", "friendship"],
    "meets_length_requirement": true,
    "flags": ["书写影响交际"]
  },
  "scores": {
    "content": 9,
    "language": 6,
    "organization": 6,
    "total": 21,
    "band": "第六档",
    "band_reason": "关键词覆盖一般但情节推进完整；语法与表达偶有瑕疵不影响理解。"
  },
  "feedback": {
    "strengths": ["情节连贯", "续写与原文情境匹配"],
    "weaknesses": ["词汇多样性一般", "部分句法可更简洁"],
    "keyword_coverage": "使用了3/5个关键词，语境基本恰当",
    "rewrite_suggestion": "≤180词的英文优化版本..."
  },
  "ocr_text": "完整还原识别出的学生续写正文（英文）"
}
```

2) 评分请求体（文本对话；将 OCR 结果作为输入）
```json
{
  "model": "{{$json.model_name || $env.SILICONFLOW_VISION_MODEL}}",
  "messages": [
    {
      "role": "system",
      "content": "You are an experienced Chinese Gaokao English continuation writing grader. Score using content(0-10), language(0-8), organization(0-7), total=25. Return STRICT JSON only with fields: meta, scores, feedback, ocr_text."
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "【任务】读后续写评分。输入为 OCR 文本与背景参数。严格输出 JSON（meta/scores/feedback/ocr_text）。\n【原文片段】{{ $json.source_text || '(未提供)' }}\n【作答要求】{{ $json.requirements || '(未提供)' }}\n【关键词】{{ Array.isArray($json.keywords_list) ? $json.keywords_list.join(', ') : '(未提供)' }}\n【七档划分】第七档(22–25) … 第一档(0)。\n【字数规则】<120词：在meta.flags标记‘字数不足’，并合理影响得分。\n【OCR结果】\n{{ $json.ocr_text }}"
        }
      ]
    }
  ],
  "temperature": 0
}
```

3) JSON 清洗/校验（Function 节点）
```javascript
function tryParse(s){ try { return JSON.parse(s); } catch { return null; } }
let raw = $json.content || $json.choices?.[0]?.message?.content || '';
raw = String(raw).trim().replace(/^```json[\r\n]+/i,'').replace(/```$/,'');
const i = raw.indexOf('{'); const j = raw.lastIndexOf('}');
const pick = (i>=0 && j>=i) ? raw.slice(i, j+1) : raw;
const data = tryParse(pick) || tryParse(raw) || {};

function n(v, d=0){ const x=Number(v); return Number.isFinite(x)?x:d; }
function b(v){ return typeof v==='boolean'?v:false; }
function s(v, d=''){ return typeof v==='string'?v:d; }
function arr(v){ return Array.isArray(v)?v:[]; }

const words = Math.max(0, Math.floor(n(data?.meta?.words)));
const content = Math.max(0, Math.min(10, Math.floor(n(data?.scores?.content))));
const language = Math.max(0, Math.min(8, Math.floor(n(data?.scores?.language))));
const organization = Math.max(0, Math.min(7, Math.floor(n(data?.scores?.organization))));
const total = Math.max(0, Math.min(25, n(data?.scores?.total, content+language+organization)));

return [{ json: {
  meta: {
    words,
    keywords_required: Math.max(0, Math.floor(n(data?.meta?.keywords_required, 5))),
    expected_keywords: arr(data?.meta?.expected_keywords).map(String),
    keywords_used: Math.max(0, Math.floor(n(data?.meta?.keywords_used, arr(data?.meta?.keywords_used_list).length))),
    keywords_used_list: arr(data?.meta?.keywords_used_list).map(String),
    meets_length_requirement: b(data?.meta?.meets_length_requirement ?? (words >= 120)),
    flags: arr(data?.meta?.flags).map(String)
  },
  scores: {
    content,
    language,
    organization,
    total,
    band: s(data?.scores?.band),
    band_reason: s(data?.scores?.band_reason)
  },
  feedback: {
    strengths: arr(data?.feedback?.strengths).map(String),
    weaknesses: arr(data?.feedback?.weaknesses).map(String),
    keyword_coverage: s(data?.feedback?.keyword_coverage),
    rewrite_suggestion: s(data?.feedback?.rewrite_suggestion)
  },
  ocr_text: s(data?.ocr_text)
}}];
```

4) 飞书多维表字段映射（新版推荐表头）
- 建议直接使用下列字段；若已有旧表头，可参考第 6 点兼容策略。
- 表头（中文名与类型）：
  - 来源类型（文本）
  - 文件标识（文本）
  - 图片URL（文本/超链接）
  - OCR文本（长文本）
  - 总分（数字）
  - 内容（数字 0-10）
  - 语言（数字 0-8）
  - 篇章（数字 0-7）
  - 七档档位（文本 或 单选：第一档..第七档）
  - 档位说明（长文本）
  - 词数（数字）
  - 关键词应使用数（数字）
  - 关键词清单（长文本）
  - 已使用关键词数（数字）
  - 已使用关键词（长文本）
  - 是否达标字数（单选：是/否 或 复选框）
  - 标记（flags，长文本；或多选）
  - 优势（长文本）
  - 劣势（长文本）
  - 关键词覆盖说明（长文本）
  - 优化建议（长文本）
  - 模型（文本）
  - Rubric版本（文本）
  - 置信度（数字 0-1，可选）
  - 审核状态（单选，默认“待审核”）
  - 创建时间（日期时间或文本）

- 字段映射（Function 节点输出给写表节点）：
```javascript
const now = new Date().toISOString();
const src = $items(0,0).json; // 包含 file_name / image_url 等
const g = $json; // 严格JSON结果

return [{ json: {
  fields: {
    '来源类型': src.source_type || 'feishu',
    '文件标识': src.file_name || '',
    '图片URL': src.image_url || '',
    'OCR文本': g.ocr_text || '',
    '总分': g.scores?.total,
    '内容': g.scores?.content,
    '语言': g.scores?.language,
    '篇章': g.scores?.organization,
    '七档档位': g.scores?.band || '',
    '档位说明': g.scores?.band_reason || '',
    '词数': g.meta?.words,
    '关键词应使用数': g.meta?.keywords_required,
    '关键词清单': (g.meta?.expected_keywords || []).join(', '),
    '已使用关键词数': g.meta?.keywords_used,
    '已使用关键词': (g.meta?.keywords_used_list || []).join(', '),
    '是否达标字数': (g.meta?.meets_length_requirement ? '是' : '否'),
    '标记': (g.meta?.flags || []).join(', '),
    '优势': (g.feedback?.strengths || []).join('\n'),
    '劣势': (g.feedback?.weaknesses || []).join('\n'),
    '关键词覆盖说明': g.feedback?.keyword_coverage || '',
    '优化建议': g.feedback?.rewrite_suggestion || '',
    '模型': $items('Config')[0].json.model_name || '',
    'Rubric版本': $items('Config')[0].json.rubric_version || 'gaokao_read_after_writing_v1.0',
    '置信度': Number.isFinite(Number(src.confidence)) ? Number(src.confidence) : 0.7,
    '审核状态': $items('Config')[0].json.audit_status_default || '待审核',
    '创建时间': now
  }
}}];
```

5) 执行顺序与节点出入参（摘要）
- OCR 节点输出：`{ ocr_text, file_name }`
- 评分请求节点输入：`{ ocr_text, model_name, rubric_version, keywords_list(optional), source_text(optional), requirements(optional) }`
- JSON 清洗节点输出：严格契约对象（见 1)）
- 映射节点输出：`{ fields: { ... } }` → 直接用于 `batch_create`

6) 兼容旧表头的建议
- 如已有“内容相关性/结构连贯/语言准确性/词汇丰富度”等字段，可折中映射：
  - 内容 ← 新版 content
  - 结构连贯 ← 新版 organization
  - 语言准确性 ← 新版 language
  - 词汇丰富度 ← 可留空或由 language 的“多样性”子维度估算（可选）