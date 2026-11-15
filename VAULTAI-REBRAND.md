# Northwest Harness white-label & VaultAI change map

Purpose: track every user-facing DeepSeek / VaultAI -> **Northwest Harness** white-label change and GMKtec local gateway wiring in this repository (`kalhar/private-windows-build`) so pulling upstream (`deepseek-ai/deepseek-harness`) stays low-conflict.

- Fork base release: `0.1.6-alpha.2` (upstream release tag `0.1.6-alpha.2`)
- Branch: `kalhar/private-windows-build`
- Guiding rule: white-label only what a user actually sees (product name, logos/wordmarks,
  desktop metadata, default provider config, and model-facing persona). Do NOT rename
  packages (`@deepseek-ai/*`), provider ids (`deepseek-official`, `llm-deepseek`), env vars
  (`DSH_*`), code identifiers, or upstream API defaults. That keeps future upstream merges
  clean while shipping **Northwest Harness** as the white-labeled build.

## How the merge stays easy

Two kinds of change live here:

1. Overlay / additive (no upstream file touched): the Northwest Harness mark is injected
   through existing UI slots and a runtime deployment patch (`apps/desktop/resources/vaultai-defaults.patch.yml`).
2. In-place string/asset edits to upstream files: listed below so a conflicting hunk on
   `git pull` is a fast re-apply.

---

## 1. Logos and image assets (binary, regenerate from source)

Northwest Harness uses a custom geometric "NW" rounded-square badge (`#141923` charcoal + `#FFFFFF` / `#38BDF8` accent) rendered via Playwright Chromium from vector SVG sources.

| File | Description |
| --- | --- |
| `apps/desktop/resources/icon.png` (`1104x1104`), `icon-windows.png` (`1024x1024`), `icon-macos.png` (`1024x1024`) | Northwest Harness `NW` badge PNGs (transparent corners) + matching `icon.svg`, `icon-windows.svg`, `icon-macos.svg` |
| `apps/desktop/installer/assets/brand.png` (`600x196`), `brand-2x.png` (`1200x392`) | Light-mode NSIS installer wordmark ("NORTHWEST HARNESS") |
| `apps/desktop/installer/assets/brand-dark.png` (`600x196`), `brand-dark-2x.png` (`1200x392`) | Dark-mode NSIS installer wordmark |
| `apps/desktop/installer/assets/uninstaller-sidebar.png` (`164x314`) | Uninstaller left panel with Northwest Harness badge on `#F5F6F8` |
| `apps/web/public/favicon.svg` | Monochrome Northwest (`NW`) favicon with `@media (prefers-color-scheme: dark)` inversion support |

favicon note: the PWA e2e (`apps/web/tests/pwa-manifest.e2e.ts`) requires the favicon to contain `fill="#000"` and a `@media (prefers-color-scheme: dark){ path { ... fill:#fff }` rule.

## 2. User-visible product-name text (in-place edits)

All user-facing product names updated to **Northwest Harness** (`short_name: "NW"`):

- `scripts/client-build-environment.ts` (`DSH_CLIENT_TITLE: 'Northwest Harness'`)
- `apps/web/public/manifest.webmanifest` (`name: "Northwest Harness"`, `short_name: "NW"`)
- `packages/client/ui-sidebar-documentpreview/src/client/office/locales.ts`
- `packages/client/ui-plugin-manager/src/client/locales.ts`
- `packages/client/ui-settings-models/src/client/locales.ts`
- `packages/bundle/web-app/src/{index,startup}.ts` (`Northwest Harness Web GUI`)
- `packages/boot/app-boot/src/index.ts`
- `apps/desktop/{README.md,README.zh.md,package.json,src/main.ts,src/locale.ts,scripts/electron-builder-config.mjs,scripts/desktop-upload-plan.ts,scripts/installed-update-distribution.ts,scripts/package-macos.ts,installer/strings.nsh}` (packaged Windows productName `Northwest Harness`, artifactName `northwest-harness-${version}-${os}-${arch}.${ext}`)
- Corresponding unit/scaffold test assertions in `apps/web/tests/scaffold.ts`, `packages/client/ui-settings-models/tests/*`, `packages/bundle/web-app/tests/web-app.spec.ts`, `packages/boot/app-boot/tests/app-boot.spec.ts`, and `apps/desktop/tests/*`.

## 3. Official brand UI slots (`packages/client/ui-brand-official`)

Mounted when `DSH_CLIENT_BUILD_PROFILE === 'official'`:

- `src/client/Brand.tsx`: `OfficialBrandMark` (geometric `NW` SVG mark), `OfficialBrandName` rendering `OFFICIAL_BRAND_COPY.name` (`Northwest Harness`).
- `src/client/locales.ts`: centralizes `OFFICIAL_BRAND_COPY` so `pnpm run verify-client-ui-i18n` passes cleanly.
- `src/client/index.ts`: occupies `sidebar.brand.mark`, `sidebar.brand.name`, and `conversation.hero.brand.mark`.

## 4. GMKtec / Lemonade gateway wiring & prompt cache (`vaultai-defaults.patch.yml`)

`apps/desktop/resources/vaultai-defaults.patch.yml` wires the default first-party `vaultai` provider route directly to the connected GMKtec EVO-X2 gateway (`http://vaultai.local/v1`):

- `displayName: Northwest Harness`
- `api: openai-completions`
- `baseURL: http://vaultai.local/v1`
- `apiKeyEnv: VAULTAI_API_KEY` (resolved from env, `$DSH_HOME/.credentials.yaml`, or `.env`)
- `headers: { X-VaultAI-Reasoning: preserve }` — instructs `VaultLM-Dashboard` proxy to preserve upstream `delta.reasoning_content` verbatim instead of flattening into `<think>...</think>` inside `delta.content`, so `@earendil-works/pi-ai` cleanly separates `reasoning` and `text` blocks.
- `cacheRetention: long` + `compat.supportsLongCacheRetention: true` — enables OpenAI prompt cache affinity (`prompt_cache_key`) against Lemonade Server's KV prefix cache.
- Registered models (`agent-default-model`: `Qwen3.8-27B-Gufo`):
  - `Qwen3.8-27B-Gufo` (Primary — vision + reasoning + tools, `off: none`)
  - `Qwen3.8-27B-GGUF` (Concurrent — vision + reasoning + tools, `off: none`)
  - `qwen3.6-35b-a3b` (Fast MoE — vision + reasoning + tools, `off: none`)
  - `gpt-oss-20b` (Ultra-Fast — text + harmony reasoning)
  - `Qwen3.8-27B-ROCmFP4-STRIX-MTP` (ROCm FP4 — vision + reasoning + tools, `off: none`)
  - `nvidia-nemotron-3-nano-30b-a3b`
  - `gpt-oss-120b-mxfp4` (Heavy)
- Disables `llm-deepseek` and `skill-badge`, and sets `system-prompt.personaPrefix` to `'You are an AI agent powered by Northwest Harness.'`.
- **Prompt Processing Bar (`: vaultai-progress`) implemented end-to-end:** `packages/llm/llm-pi-ai/src/progress.ts` (`createProgressFetch`, `parseVaultAiProgressLine`) tees the raw SSE `Response.body` before `@earendil-works/pi-ai`'s SDK parser discards SSE comments, parses `: vaultai-progress {"stage":"prompt_eval","processed":...,"total":...,"percent":...,"cached":...,"elapsed_ms":...}` frames (injected by `VaultLM-Dashboard` + `vaultai.local` `vaultai-slots-relay.service` `:13306` across `gufo:rocm` and `llamacpp:rocm`), and forwards transient `TurnProgress` updates (`Agent` -> `SessionAssistantStream` -> `ChatView` `TurnStatus` progress bar with `chat.deepDiving` localized to `'Processing...'` / `'正在处理...'`, keeping `node_modules/@earendil-works/pi-ai` and persisted `StreamChunk` schemas untouched).
- **Live Desktop Electron UI E2E Verification (`http://vaultai.local/v1`, `Qwen3.8-27B-Gufo`):**
  - Live `Northwest Harness` Desktop App (`DSH_BUILD_CLIENT_PROFILE=official`, CDP interactive session against a fresh workspace):
    - Verified white-labeled branding across the main window, `Settings -> General`, `Models`, and `Built-in plugins` (`29` session / `166` global plugins, zero `DeepSeek` strings), and `Select model` listing all `7` GMKtec models (`Qwen 3.8 27B Gufo (Primary)`, `Qwen 3.8 27B GGUF (Concurrent)`, `Qwen 3.6 35B-A3B (Fast MoE)`, `GPT-OSS 20B (Ultra-Fast)`, `Qwen 3.8 27B ROCm FP4`, `Nemotron 3 Nano 30B-A3B`, `GPT-OSS 120B MXFP4 (Heavy)`).
    - **Settings & credentials note:** `Settings -> Models -> Edit -> Customized settings` Base URL and all 7 model fields are editable (`disabled: false`, `readOnly: false`, not read-only). Note that the dev data profile (`.desktop-build/development/home/.credentials.yaml`) does not pre-populate `VAULTAI_API_KEY` (returning `MISSING_CREDENTIAL` unless set via `Settings -> Models -> Edit -> API key -> Apply`, environment variable, or `DSH_HOME=%USERPROFILE%\.dsh`). Saving the API key through the `Settings -> Models -> Edit` UI persists cleanly.
    - **Multi-turn tool execution, `: vaultai-progress`, & KV cache:**
      - Turn 1: Read an `1,800`-line `telemetry_report.txt` file in `4` read calls; live DOM `[role="progressbar"]` stepped `aria-valuenow` from `0/100` to `88/100` (`0/11,595` to `10,240/11,595`) and then `34%` to `99% · cached` (`38,268/38,341`); completed in `180,437ms` (`Ran for 2m 59s`, `185K tok`, `Cache hit 59%`) and accurately extracted all markers (`SIGNAL-HARNESS-889`, `Dr. Maya Chen`, `Sector-North-9`).
      - Turn 2: Completed in `2,932ms` (`Ran for 2s`, `Usage 38.6K tok`, `Cache hit 66%`).
      - Turn 3: Generated `generate_nw_report.py` and `nw_report.docx` (`37,007` bytes), observing `98%`–`99% · cached` (`43,190/43,195`) prompt progress, `101,988ms` (`Ran for 1m 41s`), and cumulative session footer `3 turns 14 steps · 30 tok/s · 515K tok · Cache hit 85%`.
    - **Word/Office checker & live PDF preview:** `packages/skill/skill-office/assets/scripts/check_office.py --contains` returned `exit 0` (`verdict: "pass"`, `7` paragraphs, `2x2` table); clicking `button[aria-label="Open nw_report.docx in sidebar"]` rendered a live `<canvas>` PDF preview (`1020x1320`, `7,293` non-white sampled pixels).
    - **Desktop workspace & test environment caveats:** In CDP automation on this Windows host, `Add workspace` used a temporary `dialog.showOpenDialog` IPC override on `:9229` (restored afterward) to avoid blocking on the native OS folder modal. In automated Vitest runs, `packages/skill/skill-office/tests/checkers.spec.ts` requires `DSH_OFFICE_TEST_PYTHON` pointing at the bundled primary-runtime `python.exe` (otherwise skipped), and `apps/desktop/tests/upload-with-credentials.spec.ts` requires PowerShell 7 (`pwsh`) which is not installed on this Windows host.
