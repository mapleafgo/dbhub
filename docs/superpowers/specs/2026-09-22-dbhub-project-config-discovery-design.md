# 按项目自动发现 dbhub.toml（项目级配置）

日期：2026-09-22
状态：已实现（`src/config/toml-loader.ts`、`src/config/env.ts`）

## 问题

dbhub 只能通过 `--config` 显式指定 TOML 配置：`resolveTomlConfigPath()`
（`src/config/toml-loader.ts`）在 `--config` 缺失时不做任何隐式发现，这是 commit
`871aa39` 有意为之的改动，理由是「TOML 直接选定数据库，隐式发现会让从错误目录启动
的进程静默连到别的库」。

但实际用法是：Codex 在 `~/.codex/config.toml` 里为每个项目启动一个 dbhub stdio
进程，并把 `--config` 硬编码成 FLC 的 `configs/dbhub.toml`。于是「打开哪个项目」
与「连哪个库」脱钩——在 dbhub 仓库里干活时，dbhub 连的仍然是 FLC 的库。

实测（2026-09-22，三个运行中的 dbhub 进程）：

```
PID 17111  CWD=/home/mapleafgo/Projects/MyProject/floating-life-chronicle
PID 38489  CWD=/home/mapleafgo/Projects/OfficeProject/chaosheng/agentman
PID 75053  CWD=/home/mapleafgo/Projects/OpenProject/dbhub
```

三个进程的 `--config` 全部指向 FLC 那份配置，而进程 CWD 恰好等于各自「当前打开的
项目根」。因此 CWD 是天然的、无需额外登记的项目标识。

## 目标

- 未显式指定配置时，dbhub 自动使用「当前项目根」的 `dbhub.toml`，实现「打开哪个
  项目就连哪个库」。
- 不引入隐式发现的误连风险：文件必须精确位于 CWD 这一层。

## 非目标

- 不读取 MCP roots：该能力在 2026-07-28 协议中已被标记弃用，且客户端支持不一。
- 不向上遍历父目录，也不做 `.git` 边界判定。
- 不引入全局的「项目路径 → 配置文件」映射表。
- 不改动 DSN / 环境变量路径的既有语义。

## 决策记录

1. 项目识别依据：进程 CWD。
2. 配置文件位置：仅 `<cwd>/dbhub.toml`，不兼容 `configs/dbhub.toml` 之类的子目录。
3. 解析顺序：`--config` → `<cwd>/dbhub.toml` → DSN/环境变量 → 报错退出。
4. 发现范围：仅 CWD 一层，不向上遍历。

## 接口设计

### `resolveTomlConfigPath(): string | null`（`src/config/toml-loader.ts`）

行为扩展为两段：

1. 显式 `--config` 存在时，沿用现有语义——文件不存在立即抛错，存在则返回路径。
2. 否则探测 `<cwd>/dbhub.toml`：存在返回其绝对路径，不存在返回 `null`。

不做隐式发现时的副作用：不创建文件、不读取内容、不修改任何状态。

### `isImplicitTomlConfig(): boolean`（新增导出）

仅当「未提供 `--config`」且「`<cwd>/dbhub.toml` 存在」时为 true。供
`resolveSourceConfigs()` 判断是否预加载 `.env` 复用，避免在两处重复写探测逻辑。

### `loadTomlConfig()` 的 `source` 字段

显式与隐式都返回 `path.basename(configPath)`，即 `dbhub.toml`，不改动。既有断言
（如集成测试里的 `Configuration source: environment variable`）依赖的是
`resolveSourceConfigs()` 的 `source`，不受影响。

## 行为矩阵

| `--config` | `<cwd>/dbhub.toml` | `--dsn` / DSN 环境变量  | 结果                                                   |
| ---------- | ------------------ | ----------------------- | ------------------------------------------------------ |
| 有         | 任意               | 无                      | 用显式文件                                             |
| 有         | 任意               | 有 `--dsn`              | 报错：`--dsn` 不能与 TOML 同用，提示去掉 `--config`    |
| 无         | 存在               | 无                      | 用项目文件                                             |
| 无         | 存在               | 有 `--dsn`              | 报错：`--dsn` 不能与 TOML 同用，提示改项目配置         |
| 无         | 存在               | `DSN` / `DB_*` 环境变量 | 用项目文件（环境变量不视为冲突，可用于 `${VAR}` 插值） |
| 无         | 不存在             | 有                      | 用 DSN / 环境变量                                      |
| 无         | 不存在             | 无                      | 现有「需要配置」错误并退出                             |
| 无         | 不存在             | `--demo`                | 跳过 TOML，走 demo（不变）                             |

`--demo` 行是现有语义的延续：`resolveSourceConfigs()` 本来就在 demo 模式下跳过
TOML，隐式发现同样跳过。

实现期确认的一条决策：`--dsn` 与隐式项目配置冲突时**报错**，而不是让 `--dsn` 覆盖项目
配置。理由是与现有 `--config` + `--dsn` 的互斥保护保持一致，且不静默改变「连哪个库」。
报错文案按实际情形分岔：隐式发现场景提示「移除/重命名项目配置」，显式场景才提示
「去掉 `--config`」——否则会让用户去找一个自己从未传过的参数。

## 错误处理

- `--config` 指向不存在的文件：仍立即报错，保持现状。
- 隐式发现的文件解析失败：沿用 `loadTomlConfig()` 的现有错误并终止启动。不静默回退
  到 DSN——用户明确把该文件放进当前项目，静默回退会把错误藏起来。
- 未显式指定配置而隐式命中时，向 stderr 打印一行提示，写明实际使用的路径，便于排查
  「为什么连到了这个库」。

## 实现要点

- `resolveSourceConfigs()`（`src/config/env.ts`）的 `.env` 预加载条件从
  `parseCommandLineArgs().config` 改为「显式或隐式命中 TOML」。这段预加载必须发生在
  `loadTomlConfig()` 之前，否则 `dsn = "${DSN}"` 这类插值拿不到 `.env` 的值。
- `startConfigWatcher()`（`src/utils/config-watcher.ts`）无需改动：它调用
  `resolveTomlConfigPath()`，自动获得隐式路径并监听热重载。
- 不新增命令行参数。

## 文档改动

- `docs/config/command-line.mdx`：补充 `--config` 缺省时会在 CWD 查找 `dbhub.toml`。
- `docs/config/toml.mdx`：更新「Load a TOML file with `--config`」段落，说明项目级
  自动发现与优先级。
- `CLAUDE.md`：更新 Configuration 段落中对 `resolveTomlConfigPath` 的描述。

## 测试计划

- `src/config/__tests__/toml-loader.test.ts`：无 `--config` 时从 CWD 发现；
  `--config` 优先于 CWD 文件；CWD 无文件时返回 `null`；不回退到父目录。
- `src/config/__tests__/env.test.ts`：无 `--config` 且 CWD 有 `dbhub.toml` 时
  `resolveSourceConfigs()` 返回 TOML；隐式 TOML 场景下 `.env` 会被预加载；
  `--demo` 仍跳过隐式 TOML。
- 现有测试影响：`src/__tests__/json-rpc-integration.test.ts` 与
  `src/__tests__/http-bind-host.integration.test.ts` 已用空目录隔离 CWD 并带
  `DSN`，隐式发现在空目录不命中，回退到 DSN，行为不变，复跑确认即可。
- 端到端验证：在 dbhub 仓库根放一份 `dbhub.toml`（该路径已被 `.gitignore` 忽略），
  以 `--transport=stdio` 启动，确认启动日志读取的是项目配置、工具可用；再用
  `--config` 指向别处文件，确认显式配置覆盖生效。

## 风险

- 行为变更（对 upstream 而言是 BREAKING）：CWD 里出现 `dbhub.toml` 时会自动生效，
  与显式 DSN 意图冲突时是报错而非静默忽略。
- FLC 的配置位于 `configs/`，按本设计不会自动生效，需要把文件移到项目根，或继续用
  `--config` 指定。

## 交付方式

- 本地分支开发，提交信息用中文 Conventional Commit，推送到 `origin`
  （`git@github.com:mapleafgo/dbhub.git`）。
