# 场次操作指南（shotlist · 对齐 Web /shot 批量制作）

本文档帮助 AI Agent 通过 `workrally shotlist` / `workrally series` 完成场次全生命周期管理，对应前端 `/shot`（`anime-new-shot`）批量制作能力：

- **生成走统一任务通道**（`TvShortSeriesTask.SubmitTask/BatchSubmitTask`）
- **配置聚合在 `shot.extra.gen_config.{image,video,audio}`**
- **模型统一来自 `shotlist models`（GetTaskModelList，画布同源）**
- **支持生图 / 生视频 / 生音频**，结果查询 `--type image|video|audio`

---

## 1. 概念图谱

```
项目 (project)
 └─ 剧集 (series)
     └─ 场次 (shot/story)
         ├─ ⭐ 图片提示词 (image_prompt)              ← 决定关键帧画面
         ├─ ⭐ 视频提示词 (animation_prompt)           ← 决定动效/运镜
         ├─ ⭐ 音频提示词 (extra.audio_prompt)          ← 音色用 <音色名> 引用
         ├─ 参考资产 (video_role_data_json)            ← 图片+视频统一存放
         ├─ 音频参考 (extra.audio_role_data_json)      ← 仅音频，独立字段
         └─ 生成配置 (extra.gen_config.{image,video,audio})  ← 模型/比例/时长/分辨率等
```

---

## 2. 标准工作流（唯一推荐）

`series create` → `shotlist create` → `shotlist recognize` → `shotlist models` → `shotlist set-model` → `shotlist generate-image / generate-video / generate-audio` → `shotlist get-result --type image|video|audio --watch`

```bash
# 1) 新建剧集
SERIES_ID=$(workrally series create --project-id $PROJECT_ID --name "第一集" -o json | jq -r '.series_id')

# 2) 批量创建场次（按用户意图填提示词：仅图 / 仅视频 / 图+视频）
workrally shotlist create --series-id $SERIES_ID --json-list \
  '[{"image_prompt":"古风庭院全景","animation_prompt":"镜头缓推"},
    {"image_prompt":"两位侠客对峙","animation_prompt":"推近脸部特写"}]'

# 3) 识别角色/资产（三路：image/animation/audio）
workrally shotlist recognize --series-id $SERIES_ID --project-id $PROJECT_ID

# 4) 选模型（勿硬编码）→ 写配置
workrally shotlist models --category image,video,audio -o json
workrally shotlist set-model --series-id $SERIES_ID \
  --image-model <img_id> --image-aspect-ratio 16:9 \
  --video-mode SubjectToVideo --video-model <vid_id> --duration 5 --video-aspect-ratio 16:9 \
  --audio-model <aud_id>

# 5) 生成（多场次一起；均需 --project-id）
STORY_IDS=$(workrally shotlist list --series-id $SERIES_ID -o json | jq -r '[.story_list[].story_id] | join(",")')
workrally shotlist generate-image --project-id $PROJECT_ID --story-ids "$STORY_IDS"
workrally shotlist generate-video --project-id $PROJECT_ID --story-ids "$STORY_IDS"
workrally shotlist generate-audio --project-id $PROJECT_ID --story-ids "$STORY_IDS"

# 6) 查结果（image/video/audio 三条独立进度，分别 watch）
for sid in $(echo "$STORY_IDS" | tr ',' ' '); do
  workrally shotlist get-result --story-id $sid --type image --watch
  workrally shotlist get-result --story-id $sid --type video --watch
  workrally shotlist get-result --story-id $sid --type audio --watch
done
```

---

## 3. 模型与生成配置（`extra.gen_config`）

> ⚠️ 模型来自 `shotlist models`（GetTaskModelList，画布同源），返回的 `id` 直接写入 `gen_config`；
> 旧 `shot image-models`（Kontext en_name）/ `shot video-models`（Wuji provider）**不适用于 shotlist**。

```bash
# 拉可用模型（category 逗号分隔）
workrally shotlist models --category image,video,videoSingle,videoFrame,audio -o json
# 返回每模型：id / name / aspect_ratios / resolutions(enum_value) / durations（audio 含 restrictions）
```

`set-model` 把配置写入 `extra.gen_config`（不传 `--story-ids` 则对 `--series-id` 全剧集生效）：

| 维度 | flags | 写入字段 |
|------|-------|---------|
| 图片 | `--image-model` / `--image-aspect-ratio` / `--image-resolution` / `--image-count` | `gen_config.image` |
| 视频 | `--video-mode`(SubjectToVideo\|Text\|FirstLastFrame) + `--video-model`\|`--text-model`\|`--first-last-model` / `--video-aspect-ratio` / `--duration` / `--video-resolution` / `--enable-sound` | `gen_config.video` |
| 音频 | `--audio-model` / `--audio-count` | `gen_config.audio` |

**视频三种模式**（模型字段互不通用，切模式要用对应类别的模型）：
- `SubjectToVideo`（参考主体，默认）：模型写 `model`；资产走 `video_role_data_json`。
- `Text`（单图）：模型写 `textModel`；资产走 `gen_config.video.singleAssets`。
- `FirstLastFrame`（首尾帧）：模型写 `firstLastModel`；资产走 `gen_config.video.firstLastAssets`。

---

## 4. 资产识别（recognize）

底层仍是 `Material.MatchContentRole`，但新版支持**符号分词**与**音频路**：

```bash
# 默认：both（图片+视频+音频三路）、text 规则、仅本项目资产库
workrally shotlist recognize --series-id <sid> --project-id <pid>

# 只识别某一路
workrally shotlist recognize --series-id <sid> --project-id <pid> --scope image
workrally shotlist recognize --series-id <sid> --project-id <pid> --scope audio

# 符号+文字识别：【角色】=图片/状态，<音色>=音频
workrally shotlist recognize --series-id <sid> --project-id <pid> --match-rule symbol_text

# 全资产库范围
workrally shotlist recognize --series-id <sid> --recognize-scope all
```

- `--scope`：`image | animation | audio | both`（识别哪条 prompt；默认 both）。
- `--match-rule`：`text`（默认，去掉 `【】<>` 全部当图片）/ `symbol_text`（区分图片与音色）。**音频路内部强制 `symbol_text`**。
- `--recognize-scope`：`project`（默认，仅本项目）/ `all`（全资产库）。
- 落库：图片/视频路写 `role_data_json` + `video_role_data_json`（统一）+ prompt 占位 + `extra.*_prompt_json`；音频路写 `extra.audio_role_data_json` + `extra.audio_prompt` + `extra.audio_prompt_json`。

> ⚠️ `--scope` 控制识别路（image/animation/audio/both），`--recognize-scope` 控制资产库范围（project/all），二者勿混用。

---

## 5. 生成与结果查询

> ⚠️ `shotlist generate-*` **仅提交任务**，不返回可轮询的画布 task_id；进度归属「场次 + 类型」，用 `shotlist get-result` 查。
> 三个生成命令都需要 `--project-id`（短番项目ID，SubmitTask 需要）。

```bash
# 生图（每场次 count 张）
workrally shotlist generate-image --project-id <pid> --story-ids st_1,st_2 --count 2
# 生视频（按各场次 gen_config.video.mode 分支，每场次 1 条）
workrally shotlist generate-video --project-id <pid> --story-ids st_1,st_2
# 生音频（reference_to_audio，每场次 count 条）
workrally shotlist generate-audio --project-id <pid> --story-ids st_1,st_2

# 查结果（--type image|video|audio；--watch 轮询至 state=all_done）
workrally shotlist get-result --story-id st_1 --type audio --watch --interval 5
```

`get-result` 返回关键字段：`state`(all_done/running/no_data) / `doing_count` / `done_count` / `failed_count` / `results[]` / `doing_tasks[]` / `failed_tasks[]`。`doing_count===0` 即该类型任务全部结束。

---

## 6. 音频生成（新增能力）

1. 在 `image/animation` 之外，场次可独立生成音频（配音/音效），走 `reference_to_audio` 模板。
2. 音频提示词存 `extra.audio_prompt`；**音色用 `<音色名>` 引用**，参考资产仅音频（`extra.audio_role_data_json`），与视频/图片参考完全隔离。
3. 流程：`shotlist models --category audio` 选模型 → `shotlist set-model --audio-model <id>` → （可选 `shotlist recognize --scope audio --match-rule symbol_text` 识别音色）→ `shotlist generate-audio --project-id <pid> --story-ids <ids>` → `shotlist get-result --type audio --watch`。

---

## 7. 资产绑定（bind）

```bash
# 图片/视频参考 → 统一写入 video_role_data_json
workrally shotlist bind --story-id st_1 --type image --assets '[{"asset_id":"a1","url":"https://..."}]'
workrally shotlist bind --story-id st_1 --type video --assets '[{...}]'
# 音频参考 → 独立写入 extra.audio_role_data_json
workrally shotlist bind --story-id st_1 --type audio --assets '[{"asset_id":"au1","url":"https://..."}]'
# 替换而非追加
workrally shotlist bind --story-id st_1 --type image --mode replace --assets '[...]'
```

---

## 8. 易错点

| 错误 | 正确做法 |
|------|----------|
| 用 `shot image-models/video-models` 给 shotlist 配模型 | 新版用 `shotlist models --category ...`（GetTaskModelList），id 直接写 gen_config |
| `shot` 与 `shotlist` 混用同一剧集 | 配置字段不互通（扁平字段 vs gen_config），一个剧集固定用一套 |
| `generate-*` 不传 `--project-id` | 新版走 SubmitTask，必须传短番项目 ID |
| 等 `generate-*` 返回 task_id 去轮询 | 不返回；用 `shotlist get-result --type image\|video\|audio [--watch]` |
| 一次 `get-result` 同时查三类 | image/video/audio 是三条独立进度，分别用 `--type` 查 |
| 音色识别不出来 | 音色要用 `<音色名>` 且 `--match-rule symbol_text`（音频路已强制） |
| 硬编码模型 id | 必须先 `shotlist models` 动态获取 |
