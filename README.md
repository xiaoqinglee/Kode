# Kode CLI

AI-powered terminal assistant that understands your codebase, edits files, runs commands, and automates development workflows.

## Install (npm, Node.js)

Requirements: Node.js `>=20.18.1`

```bash
npm i -g @shareai-lab/kode
kode --help-lite
```

Notes:
- Postinstall will try to download a standalone binary to `~/.kode/bin/<version>/<platform>-<arch>/` for faster startup, but npm users can run purely on Node.js without Bun.
- Configure binary download (optional): `KODE_SKIP_BINARY_DOWNLOAD=1`, `KODE_BIN_DIR`, `KODE_BINARY_BASE_URL`.

## Install (standalone binary)

Download from GitHub Releases:
- `kode-<platform>-<arch>[.exe]` (e.g. `kode-darwin-arm64`, `kode-linux-x64`, `kode-win32-x64.exe`)

Run:
```bash
chmod +x kode-darwin-arm64
./kode-darwin-arm64 --help-lite
```

## Development

```bash
bun install
bun run dev
bun test
bun run typecheck
bun run lint
bun run build:npm
bun run build:binary
```

## Docs

- `todo_tasks.json`: status-driven refactor task list
- `todo_tasks_detail.md`: execution manual + change log
- `docs/upgrade_design.md`: architecture + refactor plan
- `docs/release_checklist.md`: release checklist (npm + binaries)
- `docs/baseline_verification.md`: baseline verification record
