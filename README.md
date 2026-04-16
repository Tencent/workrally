# WorkRally CLI — Agent Skill

🎬 面向 AI Agent 的 AIGC 漫剧视频创作全流程工具集。

本目录是 WorkRally CLI 的 Agent Skill 定义。

## 目录结构

```
skill/
├── SKILL.md                ← Skill 入口（元数据 + 指令），ClawHub 解析此文件
└── references/             ← 深度参考文档，AI Agent 按需加载
    ├── canvas-guide.md         无限画布操作指南
    ├── upload-and-assets-guide.md  上传与素材管理指南
    ├── ai-generation-guide.md      AI 生成指南
    └── common-pitfalls.md          常见易错点
```

## 核心能力

- **AI 生图** — Kontext 模型，支持多参考图
- **AI 生视频** — 4 种驱动模式（文本/首尾帧/序列帧/参考主体）
- **无限画布** — Yjs 协同编辑，8 种节点类型，实时同步
- **项目 & 媒资管理** — 项目 CRUD、素材上传入库、资产库树形管理
- **通用透传** — 可调用 WorkRally MCP Server 全部工具

## 第三方商店

- [SkillHub](https://skillhub.cn/skills/workrally)
- [ClawHub](https://clawhub.ai/tencent-adm/workrally)
- [Skills](https://skills.sh/tencent/workrally/workrally)

## 快速开始

```bash
npm install -g workrally
workrally auth login
workrally auth status
```

详细用法、命令速查、工作流指南请参阅 **[SKILL.md](./SKILL.md)**。
