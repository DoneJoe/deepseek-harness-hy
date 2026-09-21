# 代码阅读指南 — DeepSeek Harness (dsh)

> 用途:在这份 monorepo 里"按图索骥",而不是硬啃源码。
> 核心原则:**这是一个文档先行的仓库,先读文档再读代码**;所有文档中英双语,直接读 `.zh.md` 版本。

## 前置认知(3 句话)

1. dsh 是 DeepSeek 开源的 agent harness,**一切皆插件**:模型适配、工具注册、会话日志、agent 循环本身都是插件,跑起来的 `dsh` 就是一棵由配置组合出来的插件树。
2. 底层框架是 Cordis(vendored 在 `vendor/`):插件向共享上下文 `ctx` 贡献**服务**、**类型化事件**和**可逆 effects**;没有"核心"可以打补丁,扩展 = 在旁边挂一个新插件。
3. 官方建议(`docs/architecture.md` 原话):用 agent 辅助探索代码库——遇到具体问题直接问,比顺序通读快得多。

## 第一阶段:文档阅读顺序(约半天)

| 顺序 | 文件 | 读完获得什么 |
|---|---|---|
| 1 | `README.zh.md` | 项目定位、怎么跑起来 |
| 2 | `docs/cordis-primer.zh.md` | Cordis 心智模型(插件/服务/事件/effects)——**一切的基础,不懂它后面都白读** |
| 3 | `docs/architecture.zh.md` | 顶层地图:profile → bundle → patch 组合机制、核心包表、事件三分类 |
| 4 | `docs/development.zh.md` | 仓库机制:pnpm workspace、TS 布局、常用检查命令 |
| 5 | `docs/glossary.zh.md` | 术语表(不一次读完,放着手边随时查) |
| 辅助 | `docs/graph-atlas.md` | 全部文档的索引图,找文档先查这里 |
| 辅助 | `docs/module-graph.md` | 包间依赖图,判断"这个包在体系中的位置" |

## 第二阶段:跑起来再读(约 1 小时)

```powershell
pnpm install
pnpm run build
pnpm dsh --profile web --dump-config   # ★ 最重要:打印你机器实际启动的插件树,每一行都可被 patch 替换
pnpm dsh web                            # 起 Web UI(http://127.0.0.1:3080),边用边对照代码
pnpm dsh --profile headless "task"      # 一次性任务模式,链路最短,适合追踪
```

真实 API 运行需要根目录 `.env` 里配 `DEEPSEEK_API_KEY`;没有 key 也能做前面所有阅读和 `--dump-config`。

## 第三阶段:沿一次请求的链路读代码(主线)

按"用户发一条消息到模型返回"的顺序读,每一步都有配套文档:

| 顺序 | 位置 | 作用 | 配套文档 |
|---|---|---|---|
| 1 | `apps/cli/src/bin.ts` → `args.ts` → `profile-boot.ts` | CLI 入口、参数解析、按 profile 启动 | `docs/subsystems/boot.zh.md` |
| 2 | `packages/boot/app-boot` | profile/bundle/patch 的组合机制 | 包内 `README.zh.md` |
| 3 | `packages/core/agent` + `packages/core/agent-loop` | Agent 接口与默认驱动——**整个系统的心脏** | `docs/subsystems/core.zh.md`、`docs/agent-lifecycle.zh.md` |
| 4 | `packages/core/session` | 只追加的 SessionEvent 日志(一切可重建的来源) | `docs/subsystems/session.zh.md` |
| 5 | `packages/core/tools` | 工具注册表与受管执行管道 | `docs/subsystems/tools.zh.md`、`docs/tool-execution-pipeline.zh.md` |
| 6 | `packages/llm/llm` | 消息/流式词汇与模型适配缝 | `docs/subsystems/llm-streaming.zh.md` |

## 三条目标导向路径

### A. 全面理解架构

走第一阶段 → 第二阶段 → 第三阶段主线即可,这是完整路径。

### B. 为了二次开发/定制(契合本 fork 的 Custom_Main 场景)

1. `docs/cordis-primer.zh.md`(必须先懂,尤其 waterfall 语义、effects 回收)
2. `docs/architecture.zh.md` 的 Profiles and bundles 一节
3. `packages/boot/app-boot/README.zh.md`(组合机制细节)→ `packages/bundle/` 下 `base`、`web-app`、`headless` 等 README
4. 按任务查 `docs/cookbook/`:`adding-a-tool.md`、`adding-a-package.md`、`adding-an-llm-adapter.md`、`extension-cookbook.md`
5. 动手实验:`--dump-config` 找到目标行 → 写自己的 `cordis.patch.yml` 覆盖 → 再 dump 验证

### C. 深入某个子系统

1. 在 `docs/subsystems/` 约 50 篇索引里按名选一篇(approval / sandbox / mcp / compaction / subagent……)
2. 读对应包 `packages/<group>/<pkg>/README.zh.md`
3. 读 `src/index.ts` 的导出(包的公开面)
4. 把包的测试当可执行示例读;`snapshots/` 里有真实会话回放,可看端到端行为

## 单个包的通用读法(5 步模板)

1. `README.zh.md` —— 包是干什么的
2. `package.json` 的 `dsh` 字段 —— 它是 profile 还是 bundle,声明了什么
3. `src/index.ts` —— 公开导出即"能力缝"的接口面
4. 找三角角色:Service Definition(接口)/ Service Provider(实现)/ Consumer(使用方)——缝永远三者齐全
5. 测试目录 —— 行为即文档

## 阅读工具

- **精确查找**:Grep(内容)/ Glob(路径),配合 `.rgignore` 已排除 node_modules
- **知识图谱**:codebase-memory MCP(`search_graph` 找定义、`trace_path` 查调用方与影响面),适合"谁在用这个函数"类问题
- **agent 探索**:官方推荐方式——把"X 功能在哪实现、怎么串起来的"直接交给探索 agent
- 根 `AGENTS.md` 与 `docs/AGENTS.md` 是写给贡献者/AI 的规则,阅读阶段只需知道存在,改代码前才必须遵守

## 读前须知(常见坑)

- 全仓库 **ESM only**(`"type": "module"`),本地相对导入带 `.ts` 后缀,看到不要惊讶
- 全部 `strict: true` 编译,类型即文档,优先读类型签名再读实现
- 文档双语同步是**贡献要求**,不是阅读障碍——中文版可能略滞后,有歧义时对照英文版
- 每个包私有、统一 `@deepseek-ai/dsh-*` 命名;`vendor/` 是钉死版本的 vendored Cordis,改法见 `vendor/README.md`
- `snapshots/` 是 keyless 录制的会话回放,当"活的行为样例"读,不是普通测试
