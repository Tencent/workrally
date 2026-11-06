# 企业知识库

仅企业账号。命令是 `workrally kb`，对应 MCP 的 12 个 `kb_*`。

用户说的「知识库」先对 `kb list` 里的项目名；对不上再对 `kb card-list` 的卡片。我的空间只有一条 `personal`，不要再建第二个。

`--format markdown|json` 是知识库正文格式，默认 markdown。`category` 只出现在 `format=json`。CLI 自己的打印格式仍用 `-o`。

检索命令的 `--os` 不传时按本机推断：macOS 是 `darwin`，Windows 是 `windows`，Linux 是 `linux`。

## 只读

```bash
workrally kb list --format json
workrally kb list --category project,personal
workrally kb traverse --kb-id <kb_id> --path-prefix /设定 --max-depth 2
workrally kb describe --kb-id <kb_id> --path-prefix /设定
workrally kb search --kb-id <kb_id> --query "主角设定"
workrally kb search-batch --targets '[{"kb_id":"<kb_id>","path_prefixes":["/设定"]}]' --query "场景"
workrally kb asset-schema --kb-ids <kb_id>
workrally kb asset-search --kb-ids <kb_id> --query "定妆"
workrally kb card-list --kb-id <kb_id>
workrally kb card-list --kb-id <kb_id> --card-id <card_id>
```

- 只查一个库用 `kb search`。同时查多个库用 `kb search-batch`，每条命中带 `kb_id`。
- 用户消息里已经给了 `kb_id` 时，直接检索，不要再 `kb list`。
- 只读不看 `writable`。`writable=false` 的库同样可以搜和浏览。
- `terms` / `ranges` / `tag-groups` 的字段必须来自 `kb asset-schema`，不要猜。

## 写入

写操作会真实改库，CLI 和 MCP 都不能撤销。删除去 Web。没有用户明确同意就不要调用 `create`、`card-create`、`raw-create`、`raw-create-batch`。

一问一答，每轮只问一个问题：

1. `kb list`，只把 `writable=true` 的库给用户选。一个都没有就停止，不要上传。
2. 等用户选定 `kb_id` 之后，只用这一个 id 调 `kb card-list`。只把 `writable=true` 的卡片给用户选。不要对多个库轮流拉卡片再拼成「库A/卡片1、库B/卡片2」。
3. 没有可写卡片时看 `can_create_card`。`false` 就停止。`true` 再问名称和描述，复述后调用 `kb card-create`。
4. 用选定的 `kb_id` + `card_id` 看 `dir_tree`，问放在哪个目录。目录不在树里时，单独问是否新建，等用户明确同意。用户最初那句话里的目录名不算同意。
5. 复述「将写入【库 / 卡片 / 路径 / 文件名】」，等用户同意后再写。

```bash
# 找或建「我的空间」。已有则直接返回，不要再建第二个
workrally kb create --name "我的空间" --category personal

# 新建企业空间项目。向用户展示 2D / 3D / 其他，传参用 2d / 3d / other
workrally kb create --name "第二季" --production-format 2d

workrally kb card-create --kb-id <kb_id> --name "分镜" --description "分集分镜"
```

文件载荷二选一：

- `--content`：只给 Agent 当场写的 Markdown / TXT，单段上限 10MB。
- `--asset-id`：现成文件。支持 `doc/docx/ppt/pptx/xls/xlsx/pdf/md/markdown/txt/png/jpg/jpeg`。不要读文件、不要 OCR、不要转 Markdown、不要改扩展名，也不要走 `workrally upload`（它不收文档，而且会带 Content-Type）。用户确认位置之后：

```bash
workrally kb upload ./设定.pdf ./主角.md
workrally asset create --url <file_url> --status 3 --channel kb
```

```bash
workrally kb raw-create --kb-id <id> --card-id <id> --directory /设定 --name 主角.md --content "..."
workrally kb raw-create --kb-id <id> --card-id <id> --directory /设定 --name 设定.pdf --asset-id <asset_id>
workrally kb raw-create-batch --kb-id <id> --card-id <id> --files '[{"directory":"/设定","name":"a.pdf","asset_id":"..."}]'
```

写入后要等后台解析，不要立刻 `kb search`。
