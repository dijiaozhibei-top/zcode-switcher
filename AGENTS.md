# AGENTS.md

ZCode Switcher — Tauri 2 desktop app for managing and hot-switching multiple ZCode (Z.ai) accounts. Windows is the primary target (NSIS bundle); macOS lives in a separate repo (`git-l-1031/zcode-switcher-mac`).

## Layout

- `src/` — React 19 + TypeScript frontend (Vite, Tailwind CSS 3 + daisyUI 5, zustand).
  - `src/App.tsx` — main UI; `src/store.ts` — zustand store; state persisted to localStorage under `zcs:`-prefixed keys.
  - `src/i18n.ts` — all UI strings in zh/en/ru. **Any new user-facing string must be added to all three languages.**
  - `src/lib/api.ts` — typed wrappers over Tauri `invoke`; the source of truth for backend command signatures.
  - `src/lib/glm52.ts` — GLM-5.2 quota/aggregation logic.
- `src-tauri/` — Rust backend. One module per concern: `profile.rs` (account store: `~/.zcode/v2/account-profiles/profiles.json` + encrypted credentials), `oauth.rs`, `quota.rs`, `crypto.rs` (AES-GCM), `jwt.rs`, `captcha.rs`, `proxy.rs`/`proxy_pool.rs` (local proxy, default port 17860), `restart.rs`, `custom_provider.rs`.
- `docs/` — `development.md`, `release.md`, `publish-and-format-guide.md`, `changelog.md`, `usage.md`. Read `publish-and-format-guide.md` before touching release/versioning.
- `scripts/release-notes.mjs` — extracts the release notes section from `docs/changelog.md` (used by CI).

## Commands

```bash
npm install                                  # frontend deps
npm run tauri dev                            # run the desktop app in dev mode (devUrl :1420)
npm run build                                # tsc && vite build — this is also the typecheck; no separate lint/test exists
cargo fmt --manifest-path src-tauri/Cargo.toml --check
cargo check --manifest-path src-tauri/Cargo.toml
npm run tauri build                          # Windows NSIS installer
```

There is no test suite and no ESLint config; `npm run build` (tsc) + `cargo check` are the verification gate.

## Conventions

- Rust DTOs returned to the frontend use `#[serde(rename_all = "camelCase")]`; commands return `Result<T, String>`. Error strings are user-facing and written in Chinese (e.g. `下载失败：{}`).
- Frontend never calls Tauri commands directly — go through `src/lib/api.ts`.
- Windows-specific code is gated behind `[target.'cfg(windows)'.dependencies]` in `Cargo.toml`; keep it that way.
- Version numbers must stay in sync across `package.json`, `src-tauri/tauri.conf.json`, and `src-tauri/Cargo.toml`.

## Release (see docs/publish-and-format-guide.md for full detail)

1. Bump version in `package.json` + `src-tauri/tauri.conf.json` (+ `Cargo.toml`), add a section at the top of `docs/changelog.md`.
2. Run the verification commands above.
3. Tag `v*` and push — `.github/workflows/release.yml` builds, uploads the installer, and publishes `latest.json` (the in-app updater manifest; it must exist or update checks 404). Updater signing private key lives only in GitHub Secrets; the pubkey is in `src-tauri/tauri.conf.json`.

## Known gotchas

- **GLM-5.3-Flash（2026-08 起唯一模型）**：ZCode 已下线 GLM-5.2/GLM-5.3，仅提供 GLM-5.3-Flash。代码中 `glm52`/`Glm52` 标识符（函数名、store 字段、i18n key、`zcs:glm52*` localStorage key）是历史命名，**不要重命名**——localStorage key 变了会丢用户设置。实际匹配逻辑在 `src/lib/glm52.ts`：优先取含 `glm-5.3` 的条目（Flash 变体被子串命中），兜底任意 `glm-5.x`（不含 GLM-5-Turbo）。本地代理（`proxy.rs`）的 provider 声明、默认模型、`/v1/models` 端点均已是 GLM-5.3-Flash；`normalize_zcode_model` 把旧模型名 `glm-5.2`/`glm-5.3` 重定向到 `GLM-5.3-Flash`，兼容旧配置请求。下次模型升级（如 5.4）时，改 `glm52.ts` 的优先匹配串、`proxy.rs` 的 provider/默认模型//v1/models/normalize 四处、三语 i18n 文案即可。
- `src/components/FloatingCapsule.tsx`、`src-tauri/src/zcode_cdp.rs`、`src-tauri/src/zcode_launcher.rs` 曾在 Release 1.1.7（commit 594d77d）被误删而引用未清（2026-08 已从 `594d77d^` 恢复）。若检出旧提交编译失败，可从该提交的父提交取回这三个文件。
- package.json scripts `notice:preview`, `notice:check`, and `build:captcha-runtime` reference `scripts/preview-notice.mjs` / `scripts/prepare-captcha-runtime.cjs` that were never committed — these npm scripts fail. Ignore the `npm run notice:check` step in the publish guide.
- `docs/development.md`'s macOS section is stale (references `build-macos.yml`, `tauri.macos.conf.json`, `docs/macos.md` that don't exist). `docs/release.md` is the current macOS story.
- Exported JSON and local profiles contain credentials — never commit account data; `docs/account-identity.md` is gitignored on purpose.
- This dev box has Node but no Rust toolchain — `cargo check`/`cargo test` cannot be run locally here; Rust changes must be verified in CI or on a machine with cargo.
