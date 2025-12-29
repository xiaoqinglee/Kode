# Kode CLI 升级设计（结构重构 + 保持行为完全一致）

> 目标：在**不改变任何外部行为**（CLI 参数/输出、协议、默认路径、交互体验、模型路由、工具 schema、MCP/ACP/stdio 协议等）的前提下，将代码库重构为更清晰的分层结构，并把测试体系迁移到可维护的分级目录，最终形成可持续演进的工程基线（Bun 开发/测试 + Node.js 运行 + Bun 单文件二进制、跨平台）。

本设计文档以本仓库当前实现与测试门禁为依据，用于记录长期维护的结构约定与迁移策略。

---

## 1. 兼容性红线（Hard Contract）

以下内容被视为“对外契约”，重构期间**任何变化都视为回归**（必须由测试锁定）：

1) **CLI 可执行文件与命令名**
- `kode` / `kd` / `kwa` / `kode-acp` 的入口与行为保持一致（包括 Node wrapper 的 fallback 策略）。

2) **CLI flags / 参数 / 默认值 / 输出格式**
- `--help-lite`、`--version`、`--print`、`--input-format`、`--output-format`、structured stdio、stream-json 等行为与顺序一致。

3) **协议与外部集成**
- MCP server：`name/version/capabilities`、tool list、tool schema（含字段/描述）稳定。
- ACP：stdio 纯协议输出约束、stdout guard 行为一致。

4) **持久化与路径**
- 配置文件、日志、session/messages/jsonl 的目录推导、命名规则、字段语义保持一致（含 Windows 路径细节）。

5) **工具系统行为**
- tool `name`、schema、权限校验、默认启用策略、输出消息 ordering、UI 文案均保持一致。

6) **模型系统行为**
- profile/pointers 解析、adapter 选择、错误语义、stream 行为与 fallback 策略保持一致。

> 强制要求：每一步重构必须在本地 `bun test` 通过后才允许进入下一步；当出现不可避免的行为变化风险，必须先补齐/加严测试再改代码。

---

## 2. 设计目标与非目标

### 2.1 目标
- **可维护性**：拆分巨型文件、减少循环依赖风险、明确模块边界与依赖方向。
- **Bun 开发 + Node 运行**：开发/测试用 Bun；npm 分发产物为 Node.js 可直接运行（`build:npm`），同时发布 Bun `--compile` 单文件二进制供“无依赖/云原生”场景直接下载运行。
- **跨平台**：macOS/Linux/Windows 路径、权限、沙箱、二进制缓存逻辑一致。
- **测试完善**：保留并迁移旧版测试；新增离线 E2E 覆盖关键 CLI 行为；对协议/工具 schema/文案做回归锁定。

### 2.2 非目标（本轮不做）
- 不改变任何外部 UI/交互细节（除非仅修复明显 bug，且必须由测试证明“旧行为错误/不稳定”）。
- 不引入 monorepo/workspaces 等更重的仓库形态；除 npm 构建使用 esbuild（必要的 Node 运行兼容）外，尽量减少工具链变更面。

---

## 3. 目标仓库结构（提案）

保持“单仓单包（single package）”以降低迁移成本，用 `src/` 内部分层表达 core/services/ui/entrypoints 等边界（不引入额外 workspaces 复杂度）。

### 3.1 顶层目录

```
.
├── src/
│   ├── entrypoints/           # cli/mcp/acp 入口（薄编排）
│   ├── core/                  # 纯业务核心（不依赖 UI）
│   ├── services/              # 外部服务集成（LLM/MCP/…）
│   ├── tools/                 # 工具实现（分类）
│   ├── ui/                    # Ink/React UI（screens + components + hooks）
│   ├── commands/              # slash/builtin 命令
│   ├── utils/                 # 可复用工具（尽量无副作用）
│   └── types/                 # 类型定义（跨层共享）
│
├── tests/                     # 统一测试入口（迁移自 src/test）
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   ├── fixtures/
│   └── helpers/
│
├── scripts/                   # 构建/发布/维护脚本（Bun/Node）
├── docs/                      # 文档（保留必要、无用迁移到 _archive）
├── dist/                      # 构建产物（Node.js 可运行的 dist + split chunks）
└── cli.js / cli-acp.js        # npm bin shim（优先二进制，fallback 到 Node dist）
```

### 3.2 兼容层策略（关键）

为了避免一次性修改大量 import 路径，采取“兼容层”逐步迁移：
- 保留现有的 TS path alias（如 `@utils/*`、`@services/*`、`@tools` 等）。
- 在迁移阶段，优先通过 **`tsconfig.json` paths 的精确映射** 保持旧导入 specifier 可用（避免在目录根部留下大量 stub 文件）。
  - 例如把 LLM 实现收敛到 `src/services/ai/llm.ts`，并将 `@services/llm` 映射到该路径。
- 仅在必要时使用 barrel/re-export（例如需要合并导出面或保留旧文件边界）。
- 每次迁移“一个模块域”并配套测试锁定，避免大爆炸式变更。

### 3.3 已落地结构（当前仓库）

本仓库已完成若干高风险域的结构拆分，并通过“兼容层”（`tsconfig.json` paths 精确映射 + 必要时 re-export）保持旧导入路径与对外行为不变（测试锁定）。

- Config：`src/core/config/*`（兼容层：`src/utils/config/index.ts`，通过 `@utils/config` 导入）
- Permissions：`src/core/permissions/*`（兼容层：`@permissions` → `src/core/permissions/index.ts`）
- Services：`src/services/{ai,mcp,plugins,system,auth,telemetry,context,ui}/*`（兼容层：`tsconfig.json` 的 `@services/*` 精确映射，保持 `@services/<name>` specifier 不变）
- ModelSelector UI：`src/ui/components/model-selector/*`（兼容层：`src/ui/components/ModelSelector.tsx`）
- Tests：`tests/{unit,integration,e2e,fixtures,helpers}/`（迁移自旧 `src/test/` 并扩展）

发布前检查清单：见 `docs/release_checklist.md`（build/test/资产/回滚策略）。

---

## 4. 模块边界与依赖方向（避免循环依赖）

### 4.1 分层依赖规则（建议）

- `src/core/**`：**不得**依赖 `src/ui/**`（核心逻辑不应绑定 UI）。
- `src/ui/**`：可以依赖 `core/services/utils/types`。
- `src/services/**`：可以依赖 `core/utils/types`，不依赖 `ui`。
- `src/tools/**`：
  - 运行时逻辑依赖 `core/services/utils/types`
  - UI（Ink 渲染）尽量依赖 `ui/components`，但不反向让 `ui` 依赖具体工具实现
- `src/entrypoints/**`：允许依赖所有层（作为编排层）。

### 4.2 如何落地（不引入重依赖工具）

短期以约定 + code review 落地，长期可考虑：
- 添加轻量脚本检测循环依赖（例如 `madge`）作为可选 CI（不强制引入到运行时）。
- ESLint 或自定义脚本检查“禁止导入路径”（仅在开发期启用）。

---

## 5. 构建与发行（Bun dev + Node runtime + Bun binaries）

发行链路：
- `scripts/build.mjs` 使用 esbuild 产出 Node.js 可运行的 `dist/`（ESM splitting + external deps）。
- `cli.js` / `cli-acp.js` 作为 npm bin shim：
  - 优先执行 postinstall 缓存的 native binary（Windows OOTB）
  - 其次 Node.js runtime 执行 `dist/index.js`（npm 安装用户无需 Bun）
  - 最后输出可操作的错误提示
- GitHub Release 额外提供 Bun `--compile` 单文件二进制（`kode-<platform>-<arch>[.exe]`），供离线/云原生场景直接下载运行。

关键要求：
- `dist/package.json` 保持 `type: module` 与 `main` 指向一致（与旧行为兼容）。
- `yoga.wasm` 复制到 `dist/`（并保留根目录文件），避免运行时缺失。

---

## 6. 测试体系迁移（分层）

目标：让默认 `bun test` 离线稳定，真实 API/真实网络场景用 env gate 控制；同时把“核心逻辑”和“CLI 集成”用目录与测试类型隔离。

Kode 的落地方式（不启用 workspaces 也能实现清晰分层）：
- 把现有 `src/test/**` 迁移到顶层 `tests/**`，并按类型拆分：
  - `tests/unit`: 纯函数/纯模块测试（不 spawn 进程）
  - `tests/integration`: 多模块组合、协议解析、文件系统/进程交互（可控离线）
  - `tests/e2e`: spawn CLI，覆盖 `--help-lite`、`--version`、`--print` 等关键路径
  - `tests/fixtures`: 固定输入输出与样例仓库/配置
  - `tests/helpers`: spawn、路径归一化、临时目录、mock server helpers

约束：
- 默认 `bun test` 必须离线可跑完（真实 API/真实网络测试必须用 env gate 控制）。

---

## 7. 分阶段迁移策略（可执行顺序 + 回滚点 + 验证清单）

> 每阶段都必须满足：`bun test` + `bun run typecheck` + `bun run build:npm` 通过。

### Phase A：基线迁入与验证（已完成）
- 内容：迁入旧版代码/测试/构建脚本，跑通 build/test/typecheck
- 回滚点：本仓库当前 `T004 success` 的状态
- 验证：见 `docs/baseline_verification.md`

### Phase B：测试目录统一（tests/ 分级）
- 目标：将 `src/test/**` 迁移为 `tests/**`，并修复导入路径
- 回滚点：迁移前 tag/commit（或保留原目录直到迁移完成）
- 验证清单：
  - `bun test` 结果与迁移前一致（pass/skip 数量不下降）
  - 关键离线协议测试必须仍能运行

### Phase C：src 分层骨架 + 兼容层
- 目标：创建 `src/core/services/tools/ui/commands/entrypoints` 并逐步迁移模块
- 回滚点：每迁移一个“域”（例如 config/permissions/llm）都能单独回滚
- 验证清单：
  - tool list/schema 变更必须有测试解释
  - CLI 启动顺序、早退出分支（`--help-lite/--version`）保持一致

### Phase D：巨型文件拆分（按风险从低到高）
建议顺序（每步都配套回归测试）：
1) `src/entrypoints/cli.tsx`：先拆启动/初始化（低风险）→ 再拆 print/stdio（高风险，协议敏感）
2) `src/services/ai/llm.ts`：拆 adapter/stream processor（以 adapter 单测为门槛）
3) `src/ui/components/ModelSelector.tsx`：拆 UI 子组件（以 ink render 回归测试为门槛）
4) tools/permissions/config/mcp：按领域拆分并保留兼容 re-export

### Phase E：文档与清理
- 将低价值/重复文档移动到 `docs/_archive/`
- 保留“用户/开发/验证”最小集合，并修复 README 导航

---

## 8. Monorepo（workspaces）是否采用？

### 8.1 不采用（本轮默认）
原因：
- 当前 `build:npm`（Node runtime）与 TS path alias 已经稳定；workspaces 会引入额外的打包与路径复杂度。
- 目标是“行为保持一致”的重构，优先减少工具链变更面。

### 8.2 结构化方法（仅借用边界思想）

在本仓库映射为：
- `src/core`（core）
- `src/entrypoints`（cli）
- `tests/helpers`（test-utils）

未来如果确有需要（例如拆出独立 SDK 或 MCP server 包），再引入 workspaces，届时以“零行为变化 + 双入口兼容”为迁移前提。

---

## 9. 成功标准（Definition of Done）

- 默认离线：`bun test` 全绿（允许 env gate 的 skip，但不得影响默认离线覆盖面）
- `bun run build:npm` 可产出可运行的 `dist/`（node wrapper + node fallback 可用）
- 目录结构清晰：新增模块都能自然归位，巨型文件显著减少
- 外部契约不变：CLI/协议/参数/路径/文案均由测试锁定
