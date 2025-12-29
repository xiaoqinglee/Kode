# 发布前检查清单（Release Checklist）

> 目标：发布 `@shareai-lab/kode` 时，确保与旧版实现的 **CLI 行为/协议/参数/体验** 完全一致，并确保构建产物与二进制分发链路可用。

## 0. 版本与标签（Versioning）

- 更新 `package.json` 的 `version`（SemVer）。
- 若发布 **native binary**（postinstall 下载安装的可执行文件），确保 GitHub Release tag 与资产命名保持一致：
  - tag：`v<version>`
  - asset：`kode-<platform>-<arch>[.exe]`（例如 `kode-darwin-arm64`、`kode-win32-x64.exe`）

## 1. ripgrep（rg）

当前实现不再分发 `vendor/ripgrep`，ripgrep 路径解析顺序为：

1. `KODE_RIPGREP_PATH`（显式指定）
2. 系统 `rg`（PATH）
3. `@vscode/ripgrep` 的 `rgPath`（npm 依赖兜底）

可选环境变量：
- `USE_BUILTIN_RIPGREP=1`：强制使用 `@vscode/ripgrep`（即使系统已安装 `rg`）

## 2. 构建/测试/静态检查（必须全绿）

在干净工作区执行：

```bash
bun run lint
bun run typecheck
bun test
bun run build:npm
```

发布前额外校验（检查产物布局与必要文件）：

```bash
bun run scripts/prepublish-check.js
```

构建产物 smoke（至少验证）：

```bash
node cli.js --help-lite
node cli.js --help
node cli.js --version
node cli-acp.js --help
node cli-acp.js --version
```

如本机存在旧版参考仓库，可运行离线“reference parity”对比（stdout/stderr/exit code + tool schema）：

```bash
KODE_REFERENCE_REPO=/path/to/legacy-kode-cli bun run parity:reference
```

## 3. native binary 下载与缓存（postinstall）

安装后脚本会尽力下载当前平台的 native binary 到用户可写缓存；失败不应阻断安装。

缓存路径（默认）：
- `~/.kode/bin/<version>/<platform>-<arch>/<kode|kode.exe>`

环境变量：
- `KODE_SKIP_BINARY_DOWNLOAD=1`：跳过 postinstall 下载
- `KODE_BIN_DIR` / `ANYKODE_BIN_DIR`：覆盖缓存根目录
- `KODE_BINARY_BASE_URL`：设置镜像下载源（需包含与 GitHub Release 一致的 asset 文件名）
  - 默认：`https://github.com/shareAI-lab/kode/releases/download/v<version>/...`
  - 镜像：`${KODE_BINARY_BASE_URL}/kode-<platform>-<arch>[.exe]`

## 4. 跨平台验证矩阵（最小集）

建议每个平台至少验证以下能力：

- CLI 入口：`kode/kd/kwa/kode-acp` 可用
- 早退出：`--help-lite`、`--version`
- 离线打印分支：`--print` + `stream-json` 约束（见 `tests/e2e/cli-smoke.test.ts`）
- MCP：stdio client/server 集成测试（见 `tests/integration/mcp/*`）
- ACP：stdio 协议 smoke（见 `tests/integration/*acp*`）

推荐矩阵：
- macOS：`arm64`、`x64`
- Linux：`arm64`、`x64`
- Windows：`arm64`、`x64`（重点验证“无 Bun 也可运行”的 native binary 路径）

## 5. 回滚方案（Rollback）

- npm 包发布错误：
  - 优先发布修复版本（patch/minor），并使用 `npm deprecate` 标记有问题版本。
  - 避免依赖 “unpublish” 作为常规回滚手段（生态限制多、风险高）。
- native binary 资产错误：
  - 修复并重新上传对应 tag 的 release assets，或发布新版本并更新 tag/资产。
  - 需要紧急止血时，可引导用户通过 `KODE_BINARY_BASE_URL` 指向可用镜像，或设置 `KODE_SKIP_BINARY_DOWNLOAD=1` 走 Node.js runtime（npm 安装用户默认无需 Bun）。
