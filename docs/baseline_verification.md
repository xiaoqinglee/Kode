# Baseline Verification

This document records the baseline validation results for the migrated Kode CLI codebase in this repository.

## Environment

- OS: macOS (arm64)
- Bun: 1.2.18
- Node: v22.14.0

## Commands Run

```bash
bun install --frozen-lockfile
bun run build:npm
bun test
bun run typecheck
bun run lint
KODE_REFERENCE_REPO=/path/to/legacy-kode-cli bun run parity:reference
```

## Results

- `bun run build:npm`: success
  - outputs: `dist/index.js` (+ split chunks), `dist/package.json`, `dist/yoga.wasm`, root `cli.js`, root `cli-acp.js`
- `bun test`: success
  - 433 pass, 8 skip, 0 fail (skips are gated by env flags such as `MOCK_SERVER_TEST_MODE=true` and `PRODUCTION_TEST_MODE=true`)
- `bun run typecheck`: success
- `bun run lint`: success
- `bun run parity:reference`: success

## Notes

- Some tests intentionally skip real-network / real-API scenarios unless explicitly enabled via environment variables.
