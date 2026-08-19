# AI 生成指南（图片 & 视频）

本文档帮助 AI Agent 正确使用 WorkRally 的 AI 图片/视频生成能力。

---

## 1. 核心规则

> ⚠️ **模型 ID 必须动态获取，严禁猜测或硬编码！**
> 模型列表是动态下发的，不同环境（开发/预发/正式）的可用模型可能完全不同。

> 🔒 所有 URL 类参数仅接受 WorkRally 官方媒资 URL，详见 SKILL.md 规则 9。

```bash
# 生图前必须先获取模型列表
workrally generate image-models -o json

# 生视频前必须先获取模型配置
workrally generate video-models -o json

# 提示词优化前必须先获取 gen_content 模型列表
workrally generate content-models -o json
```

---

## 2. 图片生成 (Kontext)

### 2.1 获取可用模型

```bash
workrally generate image-models -o json
```

返回包含：
- `models[]` — 每个模型的 `model_id`、`name`、`resolution_options`、`kontext_config`、`infer_quality_options`
- `aspect_ratios[]` — 全局可用宽高比列表（如 "1:1", "16:9", "9:16" 等）
- `resolution_options[]` — 所有模型支持的 protobuf 分辨率并集（**推荐**，传给 `--resolution`）
- `resolutions[]` — 旧档位 0/1/2 并集（兼容）
- `count_options[]` — 可选的生成数量

**关键字段**：
- `model_id` → 传给 `--model` 参数
- `kontext_config.max_input_images` → 该模型允许的最大参考图数量（不同模型不同，不要写死）
- `resolution_options[]` → `{value, label}`，value 为 protobuf 枚举（常见 4=1080P、5=1440P、6=2160P）。**`generate image --resolution` 用这里的 value**
- `kontext_config.support_resolutions` → 旧档位 0=1K / 1=2K / 2=4K（兼容已有脚本，新调用不要再用）
- `infer_quality_options[]` → 推理质量选项（可能为空）。非空时才可传 `--quality=<value>`（如 `high`/`medium`/`low`）；空列表时不要传 `--quality`

### 2.2 纯文生图

```bash
workrally generate image \
  --prompt "一只橘猫坐在樱花树下" \
  --model <model_id> \
  --aspect-ratio 16:9 \
  --poll
```

### 2.3 参考图生图

通过 `--input-images` 传入参考主体图片 URL，在 prompt 中用 "第一张图片"、"第二张图片" 引用：

```bash
workrally generate image \
  --prompt "第一张图片趴在第二张图片路中间" \
  --model <model_id> \
  --input-images "https://cat.png,https://shrine.png" \
  --poll
```

> 📌 `--input-images` 的最大数量取决于模型配置中的 `kontext_config.max_input_images`，不要写死。
> 📌 **只允许图片类型的素材**作为参考图。

### 2.4 在画布中生图

```bash
workrally generate image \
  --prompt "描述" \
  --model <model_id> \
  --project-id <画布ID> \
  --poll
```

传入 `--project-id`（画布 ID）后：
- 系统会**自动**在画布中创建 running 状态的占位节点（橙色边框 + 进度条）
- **无需**再手动调用 `build-draft` 放置生成器节点
- 生成完成后，前端自动更新节点状态

> ⚠️ `--project-id` 此处是**画布 ID**（通过 `canvas list` 获取），**不是项目 ID**！

### 2.5 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--prompt` | ✅ | — | 图片描述 |
| `--model` | ✅ | — | 模型 ID（从 `image-models` 获取） |
| `--aspect-ratio` | — | `16:9` | 宽高比 |
| `--resolution` | — | 模型首个可用 | **推荐** protobuf 枚举，取值来自 `image-models` 的 `resolution_options[].value`（常见 4=1080P、5=1440P、6=2160P）。兼容旧档位 0=1K、1=2K、2=4K。⚠️ 生图的 1/2 仍是 2K/4K，不是视频的 480P/540P |
| `--quality` | — | 不下发 | 推理质量。取值必须来自该模型 `infer_quality_options[].value`；列表为空时不要传 |
| `--count` | — | `1` | 生成数量 1-4（后端一个任务生成 1 张，count>1 会并发发起 N 个独立任务并返回 task_ids 数组） |
| `--input-images` | — | — | 参考图 URL（逗号分隔） |
| `--project-id` | — | — | 画布 ID（传入后自动创建占位节点） |
| `--short-series-project-id` | — | — | 项目 ID |
| `--name` | — | — | 素材名称 |
| `--extra-params` | — | — | JSON 对象。CLI 传 MCP `extraParams`；服务端写入 `graph_input.extra_params` 且将每个 value 序列化为字符串。例如 MJ：`'{"midjourney":{"stylize":100}}'`。不要在 CLI 侧再 stringify 一层 |
| `--poll` | — | false | 自动轮询直到完成 |
| `--poll-interval` | — | `3` | 轮询间隔（秒） |

---

## 3. 视频生成

### 3.1 获取可用模型配置

```bash
workrally generate video-models -o json
```

返回按**驱动模式**分组：
- `text_providers[]` — Text（单图/纯文）模式
- `first_last_frame_providers[]` — 首尾帧模式
- `subject_to_video_providers[]` — 参考主体模式
- `video_edit_providers[]` — 视频编辑模式（VideoEdit）

每个模型包含：
- `provider` → 传给 `--model` 参数
- `label` — 模型显示名称
- `duration_options[]` — 可用时长列表（秒）
- `resolution_options[]` — 支持的分辨率列表（`{value, label}`，value 为 protobuf 枚举值），传给 `--resolution`
- `can_upload_image/video/audio` — 支持的输入类型
- `max_image_count/video_count/audio_count` — 各类型最大数量
- `support_audio` — 是否支持音效

### 3.2 四种驱动模式

#### Text 模式（默认）— 纯文生视频 / 单图驱动

```bash
# 纯文生视频（不传图片）
workrally generate video \
  --prompt "夕阳下海浪拍打沙滩" \
  --model <provider_id> \
  --poll

# 图生视频（传入参考图）
workrally generate video \
  --prompt "图片中的角色缓缓转身" \
  --model <provider_id> \
  --single-image-url "https://example.com/character.png" \
  --poll
```

#### FirstLastFrame 模式 — 首尾帧驱动

```bash
workrally generate video \
  --mode FirstLastFrame \
  --prompt "角色从左走到右" \
  --model <provider_id> \
  --first-frame-url "https://example.com/start.png" \
  --last-frame-url "https://example.com/end.png" \
  --poll
```

> 可以只传首帧或只传尾帧（至少一个）。

#### VideoEdit 模式 — 视频编辑

`model` 必须来自 `video-models` 的 `video_edit_providers[]`。必须传 `--origin-video <原视频素材ID>`（asset_id，不是 URL）。此模式 **不要** 传 `--enable-sound` / `--no-enable-sound`（下游 graph_input 不带该字段）。

```bash
workrally generate video \
  --mode VideoEdit \
  --prompt "将背景替换成蓝天" \
  --model <video_edit_provider_id> \
  --origin-video <asset_id> \
  --poll
```

#### SubjectToVideo 模式 — 参考主体驱动

```bash
workrally generate video \
  --mode SubjectToVideo \
  --prompt "角色在场景中行走" \
  --model <provider_id> \
  --reference-assets '[{"type":"image","url":"https://character.png"},{"type":"video","url":"https://bg.mp4"}]' \
  --poll
```

### 3.3 通用选项

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--prompt` | ✅ | — | 动画描述 |
| `--model` | ✅ | — | Provider ID（从 `video-models` 获取） |
| `--mode` | — | `Text` | 驱动模式: Text/FirstLastFrame/SubjectToVideo/VideoEdit |
| `--aspect-ratio` | — | `16:9` | 宽高比: 21:9 / 16:9 / 4:3 / 1:1 / 3:4 / 9:16 等 |
| `--resolution` | — | 模型首个可用 | 分辨率枚举: 1=480P 2=540P 3=720P 4=1080P 5=1440P 6=2160P 7=4320P 8=360P（具体支持见 `video-models` 的 `resolution_options`） |
| `--duration` | — | — | 视频时长（秒），可选值取决于模型 |
| `--count` | — | `1` | 生成数量 1-4（后端一个任务生成 1 个视频，count>1 会并发发起 N 个独立任务并返回 task_ids 数组） |
| `--enable-sound` | — | 服务端默认 true | 生成音效（仅部分模型支持）。**不传则默认开启**（与前端一致）；关闭请用 `--no-enable-sound`。VideoEdit 不要传 |
| `--no-enable-sound` | — | — | 显式关闭音效（`enable_sound=false`） |
| `--extra-params` | — | — | JSON 对象。CLI 传 MCP `extraParams`；服务端写入 `graph_input.extra_params` 且将每个 value 序列化为字符串 |
| `--project-id` | — | — | 画布 ID（传入后自动创建占位节点） |
| `--short-series-project-id` | — | — | 项目 ID |
| `--name` | — | — | 素材名称 |
| `--poll` | — | false | 自动轮询直到完成 |
| `--poll-interval` | — | `5` | 轮询间隔（秒） |

### 3.4 在画布中生视频

与生图类似，传入 `--project-id`（画布 ID）即可自动创建占位：

```bash
workrally generate video \
  --prompt "海浪翻涌" \
  --model <provider_id> \
  --project-id <画布ID> \
  --poll
```

---

## 4. 视频提示词优化（gen_content）

用于视频延长/编辑前，把自然语言指令 + 可选参考视频/图片交给模型，生成适合下游视频生成的结构化提示词（如 `subject_definitions`）。

> ⚠️ 这是**异步文本任务**，不是生视频。成功后读 `output_text`，**不会**返回 `output_assets`。
> ⚠️ **model 必须来自 `generate content-models`**，严禁硬编码 MiniMax H3 等 ID。

```bash
# 1. 获取模型
workrally generate content-models -o json

# 2. 提交优化（建议 --poll）
workrally generate optimize-prompt \
  --prompt "将视频从尾帧延长5秒" \
  --model <model_id> \
  --video-url <参考视频CDN_URL> \
  --first-frame-url <首帧CDN_URL> \
  --last-frame-url <尾帧CDN_URL> \
  --reference-image-urls "<参考图1>,<参考图2>" \
  --audio-url <参考音频CDN_URL> \
  --poll
```

媒体写入 `messages[0].contents`，**不是**生视频那套 `ref_images` / `frames`。图片语义在 `image_info.type`，音频在 `audio_info.url`：

| CLI | graph_input |
|-----|-------------|
| `--video-url` | `contents.type=2` + `video_info.url` |
| `--first-frame-url` | `contents.type=1` + `image_info.type="first_frame"` |
| `--last-frame-url` | `contents.type=1` + `image_info.type="last_frame"` |
| `--reference-image-urls` | 每张 `contents.type=1` + `image_info.type="reference_image"` |
| `--image-url` | 兼容旧参数，等价于一张 `reference_image` |
| `--audio-url` / `--audio-urls` | 每条 `contents.type=3` + `audio_info.url` |

contents 顺序：文本 → 视频 → 首帧 → 尾帧 → 参考图 → 音频。

| 参数 | 必填 | 说明 |
|------|------|------|
| `--prompt` | ✅ | 待优化的自然语言指令 |
| `--model` | ✅ | 来自 `content-models` 的 `model_id` |
| `--video-url` | — | 参考视频 CDN URL（官方媒资） |
| `--first-frame-url` | — | 首帧图片 CDN URL |
| `--last-frame-url` | — | 尾帧图片 CDN URL |
| `--reference-image-urls` | — | 参考图 CDN URL，逗号分隔，可多张 |
| `--image-url` | — | 单张参考图（兼容旧参数） |
| `--audio-url` | — | 参考音频 CDN URL |
| `--audio-urls` | — | 参考音频 CDN URL，逗号分隔，可多条 |
| `--aspect-ratio` | — | 如 `16:9`（写入比例分量，不是像素） |
| `--resolution` | — | **protobuf 枚举**（1=480P…），与 `generate image/video` 相同。生图另兼容旧档位 0/1/2 |
| `--duration` | — | 目标时长秒数，不传服务端默认 5 |
| `--temperature` | — | 采样温度（可选） |
| `--seed` | — | 随机种子（可选） |
| `--extra-params` | — | 扩展参数 JSON 对象 |
| `--short-series-project-id` | — | 短番项目 ID |
| `--name` | — | 任务名称 |
| `--poll` | — | 完成后从结果读取 `output_text` |

与 `polishing_prompt` / `generate_animation_prompt` 的区别：本命令走 TaskGateway `gen_content`，面向视频模型，产物为文本。

---

## 5. 任务轮询

### 5.1 使用 --poll 自动轮询（推荐）

```bash
workrally generate image --prompt "..." --model <id> --poll
```

使用 `--poll` 后，CLI 自动：
1. 提交生成任务
2. 每隔 N 秒查询状态（默认3秒/图片，5秒/视频）
3. 显示进度条和状态（排队中/运行中/成功/失败）
4. 完成后输出最终结果

### 5.2 手动查询任务

```bash
# 单次查询
workrally generate task <task_id> -o json

# 手动轮询
workrally generate task <task_id> --poll
```

### 5.3 任务状态

| state | 含义 | 说明 |
|-------|------|------|
| 1 | 排队中 (QUEUED) | 等待资源 |
| 2 | 运行中 (RUNNING) | 正在生成 |
| 3 | 暂停 (PAUSED) | 暂停中 |
| 4 | 成功 (SUCCESS) | 媒资任务看 `output_assets`（`output_type="assets"`）；文本任务（optimize-prompt）看 `output_text`（`output_type="text"`）。不要找已废弃的 `output_products` |
| 5 | 失败 (FAILED) | `error_message` 包含错误信息 |
| 6 | 已取消 (CANCELLED) | 用户取消 |

### 5.4 多任务并发

后端「一个任务只生成 1 个素材」，当 `--count` > 1 时，CLI 会并发发起 N 个独立任务，返回的 `task_ids` 是长度为 N 的数组，每个 task 产出 1 个素材。使用 `--poll` 时 CLI 会自动并发轮询所有任务，总耗时约等于单任务耗时。

---

## 6. 素材命名最佳实践

### 画布内生成

使用 `--name` 传入"画布名称_素材特征"：
```bash
# 先获取画布名称
workrally canvas get <canvas_id> -o json
# 生成时传入有意义的名称
workrally generate image --prompt "蓝色运动鞋" --model <id> --project-id <canvas_id> \
  --name "产品设计画布_蓝色运动鞋" --poll
```

### 非画布生成

从 prompt 中提取核心关键词作为名称：
```bash
workrally generate image --prompt "一只可爱的橘猫在夕阳下奔跑" --model <id> \
  --name "橘猫_夕阳奔跑" --poll
```

---

## 7. 生成后素材处理

AI 生成的图片/视频会**自动入库到媒资系统**（后台自动完成，无需额外调用 `asset create`）。

### 如果需要上传到资产库

```bash
# 1. 生成完成后，从结果中获取 asset_id
# 2. 获取 asset_details
workrally asset get <asset_id> -o json
# 3. 挂载到资产库
workrally material add --json-list '[{"material_id":"<asset_id>","material_name":"角色名","material_type":2,"parent_id":"<role_condition_id>","material_detail":<asset_details>}]' \
  --project-ids <project_id>
```

### 如果需要在画布上展示

传入 `--project-id` 即可，系统自动处理。无需手动调用 `build-draft`。
