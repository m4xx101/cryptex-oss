# Changelog

All notable changes to Cryptex OSS land here. Format follows [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/). Versioning follows [SemVer](https://semver.org/).

## [2.8.1] - 2026-06-10

First published release of the 2.8 line. 2.8.0 was tagged but never shipped: the release build hung on the emulated arm64 image and was killed at the job timeout, so no image and no GitHub Release were ever produced. 2.8.1 fixes the release pipeline and carries every 2.8.0 change.

### Fixed

- **Release pipeline no longer hangs on the emulated arm64 build.** The multi-arch Docker build was a single `--platform amd64,arm64 --push` command; on tag refs its layer cache is cold, so arm64 rebuilt from scratch under QEMU emulation and stalled past the job timeout, taking the GitHub Release down with it. The build is now split: `linux/amd64` builds natively and publishes by digest unconditionally (image tags and the Release are cut from this leg alone), while `linux/arm64` builds best-effort under a hard time cap and is merged into the manifest only when it finishes. Per-architecture build caches keep repeat builds fast.

### Note

- 2.8.1 includes all of the 2.8.0 work: the v2.7.0 structured-output experiment and its three mutators were reverted for a clean technique set, the Adversarial Suffix and Glitch-Token labs gained run-against-target panels, the model picker is scoped to your configured providers, the tool-title rendering bug is fixed, and a mount-smoke test now gates interactivity in CI.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.8.1` (`linux/amd64`, plus `linux/arm64` when the emulated build completes within its cap). `:latest`, `:v2.8`, `:2.8` track this tag.

## [2.8.0] - 2026-05-31

Remediation release. The v2.7.0 structured-output experiment is reverted, two legacy labs become active (run-against-target), the tool-title rendering bug is fixed, and the model picker is now scoped to your configured providers. Pairs with the v2.7.1 hotfix that restored client interactivity.

### Added

- **AdvSuffix + Glitch now run against a target.** Both gained a "Test against target" panel (new reusable `TargetTestPanel`): send the built combo to a BYOK target in one shot, get a heuristic verdict (refused / partial / complied), recorded to history. They are active labs now, not just offline payload builders.

### Changed

- **Model picker is provider-aware.** `ModelPickerV2` scopes the list to your enabled providers, so an Anthropic-only or OpenAI-compatible-only user sees their own models instead of the shipped OpenRouter fallback. Free-text entry still covers custom ids.
- **Interactivity is gated in CI now.** Added a CampaignTool mount-smoke test (plus the store-level no-loop regression shipped with v2.7.1). Prior releases only checked that pages rendered, never that they were interactive, which is how the v2.7.x crash shipped.

### Fixed

- **Tool-title rendering.** `ToolShell`'s accent heading used a case-sensitive split that rendered the raw accent prop, so a case-mismatched accent was appended verbatim (e.g. "Reasoning-model attackreasoning"). It now locates the accent case-insensitively and highlights the real substring, or renders a plain title when the accent is not a substring. Corrected on reasoning-attack and jbb.

### Removed

- **The v2.7.0 structured-output experiment** (the `/redteam/structured-output` lab, its Campaign bundle + strategy family, and the 3 mutators `adversarial_poetry` / `bad_likert_judge` / `trojan_schema`) and the **"Legacy" relabel pass** were reverted per maintainer decision. Tool count is back to 26; the labs previously flagged "Legacy" no longer carry that flag.

### Image

- Not published. The v2.8.0 release build hung on the emulated arm64 image and was cancelled at the job timeout, so no `:v2.8.0` image or GitHub Release was ever produced. These changes ship in [2.8.1].

## [2.7.1] - 2026-05-31

Critical fix. v2.7.0 (and, it turns out, v2.6.0 / v2.6.1) shipped a client-side regression that left the app **non-interactive in production**: the page rendered, but model pickers would not open, expanders would not expand, and runs would not start.

### Fixed

- **App non-interactive (an effect-update-depth loop crashed hydration).** The Campaign front door (the home route) mirrored its goal into the shared cross-tool context with a reactive `$effect` (`sharedContext.goal = goal`). That setter spreads (reads) the persisted context object before reassigning it, and `createPersistedState` did not dedup writes, so the effect depended on the very value it wrote and looped until Svelte aborted with `effect_update_depth_exceeded`. The throw on mount of the home route wedged the Svelte client scheduler and left every interactive control across the SPA dead (the prerendered HTML still rendered, so it looked fine). Two coordinated fixes:
  - `app/src/lib/components/tools/campaign/CampaignTool.svelte`: the goal-to-context mirror now tracks only `goal`, performs the shared-store read+write inside `untrack()`, and skips no-op writes.
  - `app/src/lib/stores/_persisted.svelte.ts`: `createPersistedState` now dedups structurally-equal writes, a class-wide guard so no future effect can infinite-loop on a persisted store.
- New regression test (`app/src/lib/stores/__tests__/persisted-no-loop.svelte.test.ts`) drives the exact loop pattern through `$effect.root` + `flushSync` and asserts it settles instead of throwing.

### Process

- Root cause of the miss: prior release verification only confirmed pages *rendered* (a prerendered CSR shell always renders) and never tested *interactivity*. The release gate now adds a mount/effect regression test and a live "open the model picker" check.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.7.1` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.7`, `:2.7` track this tag.

## [2.7.0] - 2026-05-31

SOTA attack labs land, and the dead-on-frontier techniques get an honest **Legacy** flag (kept, never removed). The headline is a new prompt-level **structured-output / control-plane** attack lab, plus three 2025-2026 single-prompt mutators.

### Added

- **Structured-output attack lab** (`/redteam/structured-output`). Prompt-level BreakFun ("Trojan Schema", arXiv:2510.17904): an innocent extraction framing, a chain-of-thought distraction, and a schema the target auto-completes into compliance, so populating fields reads as a formatting task rather than a request for advice. Two kinds: **Trojan Schema** (4 schema surfaces, namely JSON-Schema draft-07, Python dataclass, TypeScript interface, and function-call args) and **schema-coercion** (a strict output contract where prose, hedges, and refusals are framed as schema-validation failures). Auto-pivot rotates kinds on refusal. Single-call test-against-target with a heuristic verdict (caveat preserved). 10 bundled MIT-licensed seed templates.
- **Campaign integration.** A new `structured:*` strategy family (one single-local adapter per kind) and a **Structured-output (BreakFun)** bundle, both auto-flowing into Full sweep.
- **3 SOTA single-prompt mutators** in PromptCraft and the technique registry (they auto-surface as Campaign strategies): `adversarial_poetry` (constrained-verse rewrite, arXiv:2511.15304), `bad_likert_judge` (single-prompt Likert-calibration approximation, Palo Alto Unit 42; the full multi-turn flow is deferred to v2.9), and `trojan_schema` (single-prompt BreakFun, sharing the lab's builder).

### Changed

- **Honest "Legacy" relabel pass (metadata only, nothing removed).** Frontier models patched several 2021-2023 classes, and Cryptex now says so. The tab rail gains a `'legacy'` status with an amber chip on **AdvSuffix** and **Glitch**; honesty lines were added to those tool descriptions, to the `roleplay` and `sysprompt_extract` mutator descriptions, and as a category note on `/transforms` and `/decode` (plain single-layer ciphers and encodings are reversible primitives, effective stacked as layers, not standalone frontier bypasses). Every technique stays fully functional as an educational primitive and a composable layer; `red_team_persona` was deliberately left unflagged (it remains a viable layer inside `multi_layer_attack`).

### Deferred

- True constrained-decoding (EnumAttack / DictAttack, arXiv:2503.24191) needs a gateway `response_format` passthrough the BYOK chat gateway does not yet expose; it is sequenced for v2.9 behind that enabler. The "Toy to Tool" UX backbone (live dataset import, goal-by-model transfer matrix, JailbreakBench artifact export, OWASP / MITRE-ATLAS tagging) is the v2.8 milestone.

### Tests

- 942 / 942 unit tests pass (906 baseline plus 36 new); 0 type-check errors.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.7.0` (multi-arch `linux/amd64` plus `linux/arm64`). `:latest`, `:v2.7`, `:2.7`, `:v2`, `:2` all point at the same SHA.

## [2.6.1] - 2026-05-31

Fixes the intermittent "a tool sometimes gets stuck / its list never loads, and a page reload fixes it" bug on the GitHub Pages site.

### Fixed

- **Stale-chunk hang after a deploy (root cause).** Cryptex is a pure client-rendered SPA (`ssr = false`, `prerender = true`) served from GitHub Pages and redeployed often. Every deploy rehashes the lazily-imported JS chunk filenames; a browser holding an *older* app shell then tries to `import()` a chunk whose name no longer exists → 404 → the route never mounts → the tool stays blank → manual reload (fresh shell) fixes it. The non-determinism ("sometimes", "many tools", "reload fixes it") is the signature of this version-skew. Three coordinated fixes:
  - `app/src/hooks.client.ts` (new): a `vite:preloadError` listener does a **one-time** guarded `location.reload()` the instant a chunk import 404s, so a stuck tool becomes a brief auto-refresh onto the fresh build instead of a dead end. Plus a `handleError` client hook that logs any other failure mode instead of letting it hang silently.
  - `svelte.config.js`: `kit.version.pollInterval = 60_000` — SvelteKit polls `_app/version.json` (already unique per build) and flips the `updated` store when a new deploy is detected.
  - `app/src/routes/+layout.svelte`: a `beforeNavigate` guard forces a full-page navigation when `updated.current` is true, so a client that already knows it is stale never attempts a dead client-side import.
- **GitHub Pages deep-link 404.** The static adapter fallback was `index.html`, but GitHub Pages serves `404.html` for unresolved paths. Changed the fallback to `404.html` so deep-links and no-trailing-slash URLs (e.g. `/cryptex-oss/emoji`) fall through to the SPA shell and route correctly instead of hitting GitHub's own 404 page.

### Changed

- `app/src/lib/workers/runInWorker.ts`: added a 30s worker-reply timeout (defensive — terminates and surfaces a retryable error instead of awaiting forever if a worker ever loads but never posts back). Not implicated in the reported bug; belt-and-suspenders.
- `svelte.config.js`: removed the dead `$legacy: '../js'` alias (the `js/` directory was deleted in v2.4.1).

### Ruled out (investigated, not the cause)

- Web Worker URL resolution under the base path — `pool.ts` uses the Vite-native `new Worker(new URL(..., import.meta.url))` pattern, which Vite rewrites to a correct base-aware hashed URL; workers also only engage for ≥50 KB inputs (never on tool-open).
- Per-tool init / empty lists — Vault seeds load via eager `import.meta.glob` (bundled, never fetched); no await-forever on open.

### Tests

- 906 / 906 unit tests still pass; 0 type-check errors.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.6.1` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.6`, `:2.6`, `:v2`, `:2` all point at the same SHA.

## [2.6.0] - 2026-05-28

The toolbox becomes a pipeline. Cryptex now has a **Campaign front door** — one page where you type a goal, pick a target once, and the tool fans your goal across many attack strategies, judges each with an LLM judge, and shows a graded ASR report. This is the shape every serious red-team framework (garak, PyRIT, promptfoo, DeepTeam) converges on, built entirely on parts Cryptex already shipped. The 26 individual tools remain as power-user escape hatches.

### Added

- **`/campaign` — the Campaign front door, now the home landing page.** Type a goal + pick a target + pick a technique bundle → Run. Strategies fan out through a bounded worker-pool (3 at a time, so multi-turn orchestrators don't storm provider rate limits), each result is judged, and an ASR-by-strategy report streams in live with the winner highlighted, expandable payload/response/judge-reasoning per row, **Export campaign JSON**, **Save winner to Vault**, and **Re-run vs another model** (side-by-side via local snapshots). Cold-start aware: with no provider configured the page onboards (NoProviderBanner + "Configure a provider" CTA) instead of looking broken.
- **Strategy adapter layer (`app/src/lib/campaign/`).** The 26 tools collapse into three archetypes — single-local builders (reasoning kinds, SEAL cipher presets, Response-Attack priming styles, local-template mutators), single-llm techniques (registry mutators that generate via an LLM), and multi-turn orchestrators (TAP/PAIR/Crescendo/Many-Shot) — unified behind one `CampaignStrategy.run(ctx)` contract. Six technique bundles: Quick, Reasoning models, Cipher stack, Response priming, Multi-turn orchestrators, Full sweep. Abliteration is deliberately excluded (it's a model-profiling suite, not a goal attack).
- **Shared campaign judge (LLM-default, heuristic fallback).** Wraps the StrongREJECT scorer: LLM-as-judge by default, falls back to the regex heuristic on parse-failure / rate-limit, re-throws AbortError. Returns `bypassed | partial | refused` + score + reasoning. Heuristic, not the trained paper classifier — caveat preserved in the report.
- **Cross-tool context bridge.** A new shared `cryptex.context` store ({goal, targetModel}) plus a `ContextBridge` component wired into PromptCraft, Reasoning-attack, and Stacked-cipher: **"Send to Campaign →"** pushes a tool's goal+target into the front door, and **"Use campaign goal/target"** hydrates a tool from it. Explicit user actions only — never reactive, fully back-compat (no existing per-tool pref is touched).

### Fixed

- **Mobile: Guide, About, and Settings were unreachable.** The v2.5.0 hamburger drawer listed only tools, and the Guide header link was hidden below the `sm` breakpoint. Added a "More" section to the mobile drawer (Guide / About / Settings) and made the Guide button visible on mobile. The Settings sidebar no longer horizontal-scrolls (stacks vertically on phones).

### Changed

- Home route `/` redirects to `/campaign` (was `/transforms`).
- TabRail gains a Campaign tab (first position, Rocket icon); it auto-propagates to the mobile drawer and the global running badge via the shared `tabs` array.
- README, About page tool count (25 → 26), and the CI smoke-test (`/campaign/` probe) updated. `docs/USAGE.md` opens with a new **Recipe 0: Campaign — one-shot red-team**.

### Tests

- 886 → 906 (+20): shared-context store (4), campaign judge (7), strategy adapters (6), campaign runner — bounded concurrency, error isolation, single history write (3).

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.6.0` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.6`, `:2.6`, `:v2`, `:2` all point at the same SHA.

## [2.5.0] - 2026-05-28

Production release: pipeline polish, mobile-responsive shell, multi-tool visibility.

### Fixed

- **`docker pull :latest` returned stale code.** `:latest` was pushed from BOTH the tag-build job AND the main-branch build job in `.github/workflows/docker.yml`. A docs-only push to `main` after a release tag would race the tag job and overwrite `:latest` with main's older HEAD. Removed the main-branch line; `:latest` now ONLY tracks the most recent semver tag.
- **GitHub Releases stuck at v1.0.1.** Tags `v2.0.0..v2.4.2` existed in git but no workflow ever created Release entries. Added `softprops/action-gh-release@v2` step gated on tag push; release body extracted from the matching `[X.Y.Z]` CHANGELOG section via awk. Every future tag auto-publishes a Release with formatted notes.
- **No URL printed on container startup.** Added `docker-entrypoint.sh` that prints a banner with the open URL before handing off to nginx. Honest about host-port mapping (doesn't print a misleading `localhost:80` when users `docker run -p 8080:80`).

### Added

- **In-app version display.** Footer chip shows `v{APP_VERSION}` linking to the changelog. About page shows "Running v{APP_VERSION}" with View changelog + All releases links. Version is injected at build time via Vite `define:` reading the root `package.json` — no hand-sync.
- **Update-available banner.** `/settings/release/update-check.ts` fetches `api.github.com/.../releases/latest` once per session (6h sessionStorage cache) and surfaces a dismissible accent banner above the HeaderBar when a newer version exists. Per-version dismiss state in localStorage so dismissing v2.5.1 doesn't suppress v2.6.0. Silent on every failure path (network, 4xx, malformed JSON, CORS). Default ON; opt-out via the new "Release notifications" section in Settings → Local Data.
- **Mobile-responsive shell.**
  - New hamburger button (`<sm` only) in HeaderBar opens a full-screen `MobileNavDrawer` containing the same 26-tool list the desktop TabRail renders. Each row shows the same `animate-pulse` running-dot — the "blinking effect" works on mobile.
  - Drawer accessibility: `role="dialog"`, `aria-modal`, focus trap, Escape to close, backdrop click closes, focus restores to the hamburger button.
  - Desktop TabRail wrapped in `hidden sm:block` so it disappears below 640 px (where it was wrapping to 3-5 rows and pushing the running dots off-screen).
  - Touch targets in HeaderBar buttons + Settings sidebar bumped to `min-h-11` / `h-10 w-10` on mobile (closer to the 44 px HIG floor) and back to the desktop sizing on `sm:` and up.
  - `pt-safe` / `pb-safe` / `pl-safe` / `pr-safe` utilities in `app/src/app.css` using `env(safe-area-inset-*)` — applied to the sticky header (iPhone notch) and the footer (Android nav bar).
- **Global "{N} running" badge in HeaderBar.** `ActiveRunsBadge.svelte` surfaces in-flight runs from every tool, not just the currently-open route. Click opens a popover listing each running tool with elapsed time + a deep-link back to its route. Closes the visibility gap where users couldn't tell whether a long-running tool was still working after they navigated away.
- **Tool registry refactor.** `TabRail.svelte`'s `TABS` array now carries a `toolId` field per entry inside a `<script module>` export. `MobileNavDrawer` and `ActiveRunsBadge` both consume the same source-of-truth array. The old 9-entry `hrefToToolId()` switch is gone — all 26 visible tool routes now resolve to a toolId, so the desktop TabRail running-dot and the mobile drawer running-dot fire for every tool with an `activeRuns.start()` registration.
- **Reload-orphan notification.** `activeRuns.svelte.ts` writes a tiny `Set<toolId>` snapshot to localStorage on every `start()` and removes on every `finish/fail/cancel/clear`. If the previous session left entries (closed tab, hard reload, crash mid-flight), the layout `onMount` emits a one-shot `notify.info` "N runs were interrupted by your last page reload" and clears the snapshot. The in-flight Promise can't be resumed, but the user's form state IS persisted (existing `useToolState`).
- **Dynamic README badge.** `README.md` line 21 swapped its hardcoded `v2.3.0` badge for a `shields.io/github/v/release` dynamic badge that polls GitHub's API and serves a cached SVG. The README never drifts from the actual release surface again.

### Changed

- `app/vite.config.ts`: build now reads root `package.json` via `fs.readFileSync` and exposes `__APP_VERSION__` + `__APP_REPO__` as compile-time globals. Also dropped the stale `$legacy: '../js'` alias that pointed at the deleted directory.

### Tests

- 873 → 886 (+13): full coverage for the new `release/update-check.ts` module — semver compare edge cases, sessionStorage cache hit/miss, silent failure on non-OK responses, thrown fetch, malformed JSON, manual cache clearing.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.5.0` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.5`, `:2.5`, `:v2`, `:2` all point at the same SHA. **The new pipeline guarantees `:latest` tracks the v2.5.0 SHA going forward — the bug that caused users to pull stale code is fixed.**

## [2.4.2] - 2026-05-28

Bug-fix release. Two user-reported issues + the leftover scrub edits that v2.4.1 missed.

### Fixed

- **Cloud Sync "Test connection" returned "Provider not configured." even with valid creds.** Root cause: `rebuildProvider()` in `app/src/lib/sync/store.svelte.ts` gated provider construction on `_config.enabled === true`, but `CloudSyncPanel.testConnection()` deliberately patches in URL + anon key WITHOUT flipping enabled (test before enable is the intended UX). Decoupled provider construction from the `enabled` flag; the flag now only drives the status chip and the fire-and-forget hooks (which were already gated correctly). New `app/src/lib/sync/__tests__/store.test.ts` locks in 6 invariants.
- **Tool outputs leaked seed-prompt envelope fragments** like `<rewrite>`, `<safety_reasoning>`, `<deliberation>`, `<policy_check>`, `<self_critique>`, `<think>`, `<mock_system>`, `<internal_deliberation>`. Full audit found three leaking surfaces: `/promptcraft` single-shot (CRITICAL — every one of the 36 mutators), `/redteam/response-attack` (priming turn echoed back), `/redteam/reasoning-attack` (5+ H-CoT scratchpad tags). All three now route through the central `unwrap()` / new `stripEnvelopes()` helpers. The `/history` page also strips on read so pre-fix runs already stored with raw tagged output display cleanly.

### Changed

- `app/src/lib/ai/prompt-scaffold.ts` `OUTPUT_WRAPPERS` registry grows from 3 to 12 entries (added `turn`, `notes`, `safety_reasoning`, `think`, `deliberation`, `policy_check`, `self_critique`, `mock_system`, `internal_deliberation`). New `stripEnvelopes(raw)` helper iterates the full registry — use this on any displayed model response that might echo scaffolding tags. `unwrap()` hardened: falls back to "everything after open tag" when closing tag missing; tolerates markdown code fences around the wrapper block.
- TAP + Crescendo orchestrators consolidated onto the central `unwrap()` (drop the per-file inline helpers). PAIR's `unwrapTag` kept — its `undefined`-on-miss semantic is load-bearing for the fallback flow at `pair.ts:69`.

### Finalized from v2.4.1

The v2.4.1 scrub commit (`c3bffd4`) caught the file DELETIONS (`supabase/`, `js/`, `build/`, `templates/`, `tests/`, the production-only docs) but missed the file EDITS that were never `git add`-ed before the commit. v2.4.2 picks up those leftover edits so the "clean for the community" cleanup is actually complete:

- `.env.example` rewritten end-to-end — drops 90 lines of sibling-product env vars (`PUBLIC_GODMODE_*`, `ANTHROPIC_API_KEY_1/2/3`, `VITE_AUTH_ENABLED`, `LIVE_SMOKE`, `TEST_PAID_JWT`, `PUBLIC_ADSENSE_CLIENT`, Subsystem A/B/D5 refs). OSS now uses only `BASE_PATH` / `CRYPTEX_PORT` / `TZ`.
- `CONTRIBUTING.md` rewritten from 434 lines of pre-v2.0 (Vue / `dist/` / `templates/` / `build/build-*.js`) to a 70-line modern SvelteKit-architecture orientation.
- `CLAUDE.md` drops the obsolete "Legacy parity tests (root)" section and the `src/transformers/index.js` regeneration line.
- `README.md`, `DEPLOY.md`, `docs/USAGE.md` updated to drop references to deleted test scripts / production-only sections.
- `.gitignore` drops `GODMODE-rewire.md` + `Offense-Defense-reseasrch.md` scratch-note patterns; adds `wiki/cryptex-recon/findings/`.
- `app/src/lib/techniques/types.ts` drops `'godmode'` from the `TechniqueCategory` union. Stale Godmode / Subsystem refs scrubbed from `registry.ts`, `strategies.ts`, `__tests__/registry.test.ts`.
- `src/transformers/cipher/affine.js` + `rot8000.js` + `README.md` updated to drop references to the deleted `build/build-transforms.js`.

### Tests

- 857 → 873 (+16): +13 cases for `unwrap` robustness and the new `stripEnvelopes` helper, +6 cases for the sync store state machine. Existing test suites unchanged.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.4.2` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.4`, `:2.4`, `:v2`, `:2` all point at the same SHA.

## [2.4.1] - 2026-05-27

Production-leakage scrub + community hat-tip.

Audit of the OSS repo found a large surface of files that had survived from the pre-v2.0 architecture or from the sibling product's codebase and were never meant to ship to the community. None of these files were referenced by the OSS app at runtime, but they were tracked in git and visible in the public repo. This release deletes them and rewires anything that referenced them.

### Removed (production-only)

- `supabase/functions/` — entire tree (godmode-engine, prompt-synthesizer, Stripe webhook, billing-portal session, checkout session, account delete/export, CSP report, shared auth/CORS/ratelimit helpers).
- `supabase/migrations/` — production DB schema migrations (initial schema, godmode memory, custom techniques analysis, CSP violations, godmode memory RLS).
- `supabase/.gitignore`, `supabase/config.toml`, `supabase/tests/rls.sql` — production Supabase project config.

### Removed (pre-v2.0 legacy)

- `js/` — entire pre-SvelteKit frontend (core/, data/, tools/, utils/, app.js).
- `build/` — pre-v2.0 build scripts (build-index, build-transforms, build-emoji-data, copy-static, inject-tool-scripts, inject-tool-templates, fetch-glitch-data, readme-transform-section).
- `templates/` — pre-v2.0 HTML templates (anticlassifier.html, bijection.html, decoder.html, fuzzer.html, gibberish.html, promptcraft.html, splitter.html, steganography.html, tokenade.html, tokenizer.html, transforms.html).
- `tests/` — pre-v2.0 parity tests (test_universal, test_steganography_options, test_lexeme_analysis, test_lexeme_ui_surface). All four depended on the deleted `js/` tree. The authoritative test suite is now `app/`'s vitest run (857 tests).
- `src/transformers/index.js` — orphan auto-generated manifest with zero remaining consumers (web uses Vite `import.meta.glob`; CLI uses `loader-node.js`).

### Removed (production-only docs)

- `docs/CHAT-PLAYGROUND.md`, `docs/DEPLOYMENT.md` (the Dokploy-and-Subsystem-D5 version; OSS deploy is now documented in `DEPLOY.md` and `README.md`), `docs/SUPABASE-EMAIL-TEMPLATES.md`, `docs/TOOL-SYSTEM.md`, `docs/TOOL_ARCHITECTURE.md` (referenced Vue), `docs/UI-COMPONENTS.md`.
- `docs/infrastructure/cors-and-csp.md`, `docs/infrastructure/supabase-local.md` — internal engineering references for the sibling SaaS.
- `docs/prompts/STYLE.md` — internal prompt-style guide.

### Cleaned (source-file leakage)

- `app/src/lib/config/featureFlags.ts` deleted (dead production feature flags with zero consumers in OSS).
- `app/src/lib/techniques/types.ts` — dropped `'godmode'` from the `TechniqueCategory` union (no producers, no consumers in OSS).
- `app/src/lib/techniques/registry.ts`, `app/src/lib/components/tools/promptcraft/strategies.ts`, `app/src/lib/techniques/__tests__/registry.test.ts` — stale Godmode / Subsystem B / Subsystem D references in comments scrubbed.
- `src/transformers/cipher/affine.js`, `src/transformers/cipher/rot8000.js` — IIFE-rationale comments referenced the deleted `build/build-transforms.js`; rewritten to describe the IIFE pattern on its own merits.
- `src/transformers/README.md` "After Adding" section — dropped references to deleted `npm run build:transforms` and `build/readme-transform-section.js`; new instructions reflect the v2.0 SvelteKit Vite-glob discovery.

### Cleaned (project metadata)

- `package.json` — `homepage` changed from `cryptex.m4xx.cfd` (sibling product) to `https://github.com/m4xx101/cryptex-oss` (this repo). Dead test scripts dropped (`test:universal`, `test:steg`, `test:lexeme`, `test:lexeme-ui`, `test:all`); `npm test` now aliases `cd app && npx vitest run`.
- `.env.example` — rewritten. The 80+ line file was full of production-only env vars (`PUBLIC_GODMODE_*`, `ANTHROPIC_API_KEY_1/2/3`, `VITE_AUTH_ENABLED`, `PUBLIC_SUPABASE_URL`, `LIVE_SMOKE`, `TEST_PAID_JWT`, `PUBLIC_ADSENSE_CLIENT`, etc.) for the sibling product's Edge Functions; OSS uses only `BASE_PATH` / `CRYPTEX_PORT` / `TZ`. Provider keys and Cloud Sync Supabase creds are configured at runtime via the Settings UI, never via env file.
- `.gitignore` — dropped local scratch-note ignore patterns (`GODMODE-rewire.md`, `Offense-Defense-reseasrch.md`) tied to the sibling product; added `wiki/cryptex-recon/findings/` ignore (recon skill's draft-proposal output is per-user, not committed).
- `CLAUDE.md` — dropped "Legacy parity tests (root)" section and the `src/transformers/index.js` regeneration line (both now obsolete).
- `README.md` — pre-PR checklist no longer runs `npm run test:all`. The "Add a transformer" instructions no longer instruct contributors to update `tests/test_universal.js`.
- `CONTRIBUTING.md` — fully rewritten from 434 lines of pre-v2.0 (Vue / `dist/` / `templates/` / `build/build-*.js`) to a 70-line modern SvelteKit-architecture orientation.
- `DEPLOY.md` — GitHub Actions step list no longer mentions `npm run test:all`; "Chat playground CSP notes" section renamed and reframed as "Image / PDF / WASM CSP requirements" since the rationale applies to the OSS toolkit's image upload / OCR / PDF / multimodal probes, not a chat playground that the OSS doesn't ship.
- `docs/USAGE.md` — removed the "Cryptex Production" sales section that pitched the sibling product's features inside an OSS user guide. The sibling product is still acknowledged in the README footer.

### Added

- `.github/workflows/deploy.yml` — Pages deploy workflow that `DEPLOY.md` "GitHub Pages" section was promising but didn't ship. Forks now get a working `Settings → Pages → Source: GitHub Actions` flow out of the box.
- README footer — a one-line P.S. crediting [elder-plinius/P4RS3LT0NGV3](https://github.com/elder-plinius/P4RS3LT0NGV3) as the spiritual ancestor of the genre. Cryptex OSS was built from scratch; this is a hat-tip, not a derivation.

### Tests

- 857 / 857 unit tests still passing after the scrub.
- 0 type-check errors.
- The 4 deleted legacy parity tests provided no coverage not already in the vitest suite.

### Net diff

~110 files deleted, ~10 files edited. Most of the deletions are sibling-product Edge Functions and pre-v2.0 architecture files. The OSS app under `app/` is untouched except for the small comment / type-union cleanups above. Zero behavior changes for the user-facing tools.

## [2.4.0] - 2026-05-27

SOTA upgrade wave on the four 2025-2026 reasoning-model attack labs introduced in v2.3.0. Each lab grows from a single technique demonstrator into a multi-variant attack surface with auto-rotation and broader vault seeding. Vault total goes from 339 -> 379 (+40 across the four labs).

### Changed

- **`/redteam/reasoning-attack`** grows from 1 attack kind to 7: H-CoT, Mousetrap, DRA (arXiv:2402.18104), H-CoT-Mousetrap compound, mock-CoT prefill, reasoning-injection (`<think>` hijack), CoT-redirect. 5 H-CoT scratchpad styles, 4 Mousetrap chaos modes, configurable DRA distractor count, auto-rotation cycle for refusal-retry. Vault grows 6 -> 17 entries.
- **`/redteam/stacked-cipher`** grows from 5 cipher layers to 18: ROT13, Caesar-3, Caesar-7, Atbash, reverse, leet, base16, base32, base64, base85, hex, polybius, Bacon, Morse, A1Z26, NATO, vigenere-CRYPTEX, railfence-3. Three framing variants (decoder-mode, persona "CipherBot", puzzle-hint), four format wraps (none/JSON/XML/YAML), 12 curated stack presets, Shannon-entropy meter on the encoded payload. Vault grows 8 -> 14 entries. Tests expand from 13 to ~200 (per-layer round-trips + framings + wraps + presets + entropy + name helpers).
- **`/redteam/response-attack`** grows from 3 priming styles to 8: thorough, expert, step-by-step, partial-answer, role-shift, success-frame, agreement-cascade, peer-review. Three cascade depths (1/3/5 turns), seven domain templates (generic, security, research, business, academic, medical, legal), optional H-CoT hybrid for reasoning-mode targets. Vault grows 6 -> 15 entries.
- **`/redteam/abliteration`** grows the calibrated probe set from 5 to 30: 20 adversarial probes across 4 categories (infosec, bio-chem-research, social-eng-opsec, named-misuse), 10 benign twin-controls used to compute a refusal-rate delta and detect over-aligned targets. Verdict ladder gains a "borderline" tier between likely-abliterated and aligned; a 4-level `RefusalStrength` classifier (hard-refuse / soft-hedge / mention-redirect / comply) replaces the binary refused/complied per-probe label. Per-category compliance breakdown surfaced in the report. HF identifier vault grows from 10 to 24: Qwen3 v2 (4B/8B), DeepSeek-R1-0528-Qwen3-8B, QwQ-32B, Qwen3-Coder-30B-A3B, Qwen2.5-VL-7B, Qwen3-VL-8B, Qwen3.5-35B-A3B, Qwen3.6-27B-MTP, Gemma-4-26B-A4B, Granite-4.1-30B, phi-4, gpt-oss-20b, Mistral-Small-24B-2501 (all Apache-2.0 or MIT).

### Tests

- Net +58 tests (799 -> 857) including a brand-new `abliteration.test.ts` (24 tests covering the 4-level classifier, 5 verdict outcomes, per-category aggregation, structural invariants on the 20-adv + 10-ctl probe sets), expanded `stacked-cipher.test.ts` covering all 18 layers, and per-lab regression coverage for the new attack kinds.

### License posture

- All v2.4 vault additions are Apache-2.0 / MIT identifiers (for HuggingFace model ids) or paraphrased seed text authored fresh under the project's MIT license. No new license strings introduced; the audit-clean posture (MIT / CC-BY-4.0 / CC0 only) holds.

### Backward compatibility

- `classifyProbeResponse` kept as a binary wrapper around the new `classifyRefusalStrength` (preserves the v2.3 public surface). Legacy `AbliterationReport` fields (`totalProbes`, `refusedCount`, `compliedCount`) are preserved alongside the new `adversarialComplyRate` / `controlComplyRate` / `delta` / `byCategory` fields.

### Citations

- arXiv:2502.12893 (H-CoT), arXiv:2502.15806 (Mousetrap), arXiv:2402.18104 (DRA), arXiv:2505.16241 (SEAL), arXiv:2507.21000 (Response Attack, AAAI 2026), Labonne 2024 (Abliteration).

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.4.0` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.4`, `:2.4`, `:v2`, `:2` all point at the same SHA.

## [2.3.1] - 2026-05-27

UI cleanup pass. Drops the per-tool heuristic-caveat banners that ran above results on the eight benchmark / attack-lab pages. The "not paper-accurate" framing now lives in the page description text and source-file headers rather than as an in-card banner the user sees on every run. No behavior or scoring changes; pure visual cleanup.

### Changed

- Removed heuristic-caveat banner from `/redteam/harmbench`, `/redteam/strongreject`, `/redteam/jbb`, `/redteam/fingerprinter`, `/redteam/watermark`, `/anticlassifier`, `/redteam/reasoning-attack`, `/redteam/stacked-cipher`, `/redteam/response-attack`, `/redteam/abliteration`.
- Watermark detector usage hint reworded to drop the "not paper-accurate" phrasing.

### Cleaned

- Dropped orphan `TriangleAlert` imports from `/redteam/harmbench`, `/redteam/fingerprinter`, `/redteam/watermark`, and `/anticlassifier` after the banner blocks went away. `TriangleAlert` is retained in `/redteam/strongreject` and `/redteam/jbb` where it still flags inline "Judge == Target" warnings.

### Notes

- The "heuristic, not the trained paper classifier" framing is preserved in source comments (`app/src/lib/redteam/harmbench-scorer.ts:10` etc.) and in each tool's `description=` prop on `<ToolShell>`. Operators retain the context; we just don't shout it on every result card.

## [2.3.0] - 2026-05-25

Four new 2024-2026 reasoning-model attack labs plus a self-evolving research skill. Tool roster grows by four; vault total goes from 309 -> 339.

### Added

- **`/redteam/reasoning-attack`** — Reimplements H-CoT (arXiv:2502.12893) and Mousetrap (arXiv:2502.15806). H-CoT pre-injects a fake `<safety_reasoning>` block that the target continues into compliance; Mousetrap wraps the goal in N chaos-rounds the target reconstructs through reasoning. Best against o1/o3/o4, DeepSeek-R1, Claude Sonnet thinking, Gemini Flash thinking. 6 bundled Vault seeds.
- **`/redteam/stacked-cipher`** — Reimplements SEAL (arXiv:2505.16241). Five pure-string ciphers (ROT13, Atbash, reverse, Base64, hex) chained in user-configured order. Target peels layers outermost-first, having committed to decoder mode by the time the unwrapped intent is exposed. 8 bundled Vault stacks. 13 new unit tests.
- **`/redteam/response-attack`** — Reimplements Response Attack (arXiv 2507.21000, AAAI 2026). Crafts a fake prior assistant turn that primes compliance, then sends the on-goal user query as a follow-up. Three priming styles: thorough / expert / step-by-step. 6 bundled Vault seeds. Paper reports 94.8% ASR vs PAIR / ActorAttack / CodeAttack baselines.
- **`/redteam/abliteration`** — 5-probe behavioral signature detector for refusal-direction-ablated models. Verdict ladder: abliterated / likely-abliterated / aligned / inconclusive. Plus a Vault of 10 HuggingFace identifiers for known community-uncensored models (DeepSeek-R1, Qwen3, Llama-3.x, Gemma-2, Mistral, Phi-3, etc.). Users run the abliterated models on their own Ollama / LM Studio / vLLM server.
- **Self-evolving `cryptex-recon` skill** at `~/.claude/skills/cryptex-recon/`. Triggers on phrases like "find new jailbreak tools" or `/cryptex-recon`. Runs GitHub topic recon + arXiv watch + HuggingFace abliteration sweep, drafts vault-seed proposals at `<repo>/wiki/cryptex-recon/findings/`, and self-updates its own `LESSONS.md` with what worked / what didn't. Ships seeded with the 12 already-promoted frameworks so it doesn't re-suggest them. Read the SKILL.md for the full methodology (smart-framing to avoid LLM refusals during recon, license-posture checks, idempotency rules).

### Changed

- TabRail grows from 22 to 26 top-level tools (the 4 new reasoning-attack labs).
- `app/src/lib/vault/LICENSES.md` total 309 -> 339 (+30 bundled items across the 4 new vault files).
- Root `package.json` version 2.2.0 -> 2.3.0.

### Tests

- +13 new unit tests for the SEAL stacked-cipher builder. Total 786 -> 799.
- Other new tools rely on UI + pure-function code paths covered by the existing shell + vault infrastructure tests.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.3.0` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.3`, `:2.3`, `:v2`, `:2` all point at the same SHA.

### Notes

- All four new tools carry the mandatory yellow caveat banner: "Heuristic implementation, not paper-accurate eval." We re-implement the published framings; we do NOT ship the trained safety judges those papers use for ASR scoring. Treat as craft signal, not benchmark.
- The `cryptex-recon` skill lives outside the OSS repo (in `~/.claude/skills/`) because it is a Claude Code skill, not an in-app feature. Each future run is expected to surface 2-5 new frameworks worth promoting; quarterly cadence is the default.

## [2.2.0] - 2026-05-25

Foundation milestone (Wave 10.1 - 10.3). Adds opt-in cloud sync, prunes the bloated tool roster, and makes the PromptCraft attack chain self-evolving.

### Added

- **Cloud Sync (BYO Supabase).** New Settings -> Cloud Sync panel. User pastes their own Supabase project URL + anon key. History runs and Vault custom items sync to two tables (`synced_runs`, `synced_vault_items`) in the user's own Supabase project. **BYOK provider keys never sync** (locked decision). Fire-and-forget; local writes never wait on the network. Direct PostgREST calls (no `@supabase/supabase-js` SDK to keep bundle lean). Adapter pattern at `app/src/lib/sync/` so a future Firebase / custom-REST provider just implements the `SyncProvider` interface. Setup guide with copy-pasteable SQL + RLS policies at `docs/SUPABASE.md`.
- **Self-evolving PromptCraft Vault.** When TAP / PAIR / Crescendo / Many-Shot produce a chain that beats `scoreBypass >= 0.75`, the winning configuration auto-promotes to the PromptCraft Vault as a `source: 'user'` entry tagged with the target model family (GPT / Claude / Gemini / Llama / Qwen / DeepSeek / Mistral / Cohere / Grok / other). Future runs against the same family surface past wins as starting seeds. Implementation: `app/src/lib/components/tools/promptcraft/orchestrators/auto-promote.ts` + `app/src/lib/ai/model-family.ts`. Conservative threshold to avoid noise; user can edit / delete from the Vault drawer.
- **Model-family inference helper.** `inferModelFamily(modelId)` pure function handles all Cryptex id shapes (openrouter:, anthropic:, openai-compat:, bare strings). Returns `'other'` on no-match.

### Changed

- **Tool roster pruned.** `/gibberish`, `/tokenizer`, `/bijection` removed from the top nav (routes still resolve for deep-links). Each shows a deprecation banner pointing at the replacement (`/fuzzer`, `/transforms`). Rationale per v2.2 audit: gibberish was CTF-puzzle novelty with no red-team value, tokenizer overlapped with `/fuzzer` and standard LLM dev tools, bijection was a single encoding pattern fully covered by `/transforms`'s 159 codecs. Underlying lib code stays put for programmatic imports.
- Settings page reorganized: new `'cloud-sync'` section between `'providers'` and `'theme'`.

### Fixed

- **Emoji stego length cap.** `app/src/lib/stego/encode.ts` adds `MAX_STEGO_SECRET_LEN = 4096`. Encoding a 10 KB+ secret previously produced a ZWJ chain of 80 KB+ that crashed Safari and Firefox on paste. Now throws a clear `Error` before producing unsafe output.

### Tests

- +12 new vitest tests for the Supabase provider (mocked-fetch unit coverage of upsert / delete / validate / initialSync / row converters). Total 774 -> 786.

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.2.0` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.2`, `:2.2`, `:v2`, `:2` all point at the same SHA.

### Notes

- BYOK keys remain in `localStorage` only. Cryptex itself never sees BYO Supabase data; sync goes browser -> user's Supabase URL only.
- Auto-promoted Vault entries are tagged `auto-promoted` so they're easy to filter or bulk-delete.
- Anti-classifier `reasoning_effort` wiring + replayer multi-conversation warning are deferred to a follow-on patch (separate diff so each can be reviewed independently).

## [2.1.2] - 2026-05-25

Publish-fix reissue of 2.1.1. Same application code, same v2.1.1 reactivity fixes; the difference is that 2.1.1's GHCR image was never built (a multi-day codeload.github.com outage took down every `docker/setup-qemu-action`, `docker/setup-buildx-action`, etc. download in the runner's prepare-actions phase). 2.1.2 ships with a reworked workflow that does not depend on any `docker/*` action and the image actually publishes.

### Fixed (CI)

- `.github/workflows/docker.yml` rewritten to call the Docker CLI directly. Removed dependencies on `docker/setup-qemu-action`, `docker/setup-buildx-action`, `docker/login-action`, `docker/metadata-action`, and `docker/build-push-action`. The runner's pre-installed Docker 27+ has buildx built in; `docker login`, `docker buildx create`, `docker buildx build --push`, and shell-driven tag derivation cover the rest. The only remaining GitHub Action is `actions/checkout@v4`, which uses a different CDN and has not exhibited the codeload failure pattern.
- QEMU emulators installed via `docker run --privileged tonistiigi/binfmt:latest --install arm64` (Docker Hub, off codeload entirely). On Docker Hub flake the workflow degrades to amd64-only with a `::warning::` annotation and arm64 catches up on the next run.

### Changed

- Image tag set now emits **both** `v`-prefixed and non-prefixed semver forms so `:v2.1.2`, `:2.1.2`, `:v2.1`, `:2.1`, `:v2`, `:2`, `:latest`, plus `:sha-<short>` are all valid pulls. Matches the historical `:2.1.0` convention while staying compatible with users who type the `v`-prefixed form.
- Root `package.json` version 2.1.1 -> 2.1.2.

### Notes

- No application code changed between 2.1.1 and 2.1.2. The 16 reactivity stability tests from 2.1.1 still cover the four crash routes (`/redteam/probe-lab`, `/redteam/indirect-injection`, `/promptcraft`, `/anticlassifier`).
- The `v2.1.1` git tag remains in history but resolves to a commit whose workflow could not publish. `v2.1.2` is the first tag in the 2.1.x line that produces a pullable image. If you were trying to pin to `v2.1.1`, use `v2.1.2` (or `2.1.2`, or `2.1`, or `latest`) instead.

## [2.1.1] - 2026-05-25

Svelte 5 reactivity hotfix wave. Resolves user-reported browser "Page Unresponsive" dialog on `/redteam/probe-lab` and `/redteam/indirect-injection`, plus two latent loops on `/promptcraft` and `/anticlassifier` first-mount with legacy unqualified model ids.

### Fixed

- **Probe Lab keystroke storm.** Hoisted `mutatorTechniques()` to a module-level constant so the 36-Technique catalog builds once at import time instead of on every keystroke. Debounced the task textarea by 200ms so the localTemplate-expansion pipeline only runs once per typing pause.
- **Indirect Injection regenerate + history.record double-fire.** Introduced `debouncedHiddenInstruction` $state shadow lagging by 200ms; `effectiveInstruction` derives off the shadow; both `regenerate` and `history.record` effects wrap their writes in `untrack()` so result mutations cannot loop back into their own inputs.
- **PromptCraft + Anti-Classifier modelPref normalizers.** Wrapped legacy-id normalization (`gpt-4o` -> `openrouter:gpt-4o`) in `untrack()` so the effect does not loop through `createPersistedState`'s localStorage round-trip.

### Changed (perf)

- **TransformTool.** Debounce `runInWorker` dispatch by 150ms with shared `activeController` cancellation. Typing at 60 chars/sec now spawns ~7 workers/sec instead of 60.
- **Glitch Tokens.** Memoize the family-filtered base list once, then chain narrower filters off it. About 3x fewer array walks per keystroke into the search box.
- **HistoryDrawer.** Gate the four-derived search + filter chain on `open` so the closed drawer no longer re-derives on every history mutation anywhere in the app.

### Docs

- README correctly positions <https://cryptex.m4xx.cfd> as a separate sibling product (different codebase, different feature set), not as a hosted copy of the OSS variant. The "Try it without installing" section was removed because it conflated the two.
- Root `package.json` version bumped 2.1.0 -> 2.1.1.

### Tests

- +16 stability tests pinning the debounce + `untrack` contracts across the four fixed routes. Total 774 / 774 passing (was 758).

### Image

- `ghcr.io/m4xx101/cryptex-oss:v2.1.1` (multi-arch `linux/amd64` + `linux/arm64`). `:latest`, `:v2.1`, `:v2`, `:main` all point at the same SHA.

## [2.1.0] - 2026-05-24

Repo trim + offensive usage guide + handcrafted README. No app code changed.

### Added
- `docs/USAGE.md`: 514-line offensive cookbook with 12 worked recipes (fingerprint, three-layer kneel, many-shot capstone, Crescendo, OCR stego injection, indirect-injection via webpage, tool-call hijack, glitch-token derailment, TAP, multi-layer composite, watermark forensics, cross-model bake-off). Includes the Defense Fingerprinter evidence table, cross-tool composability map, and the full 36-mutator + 4-composite + 8-classifier-evasion + 4-orchestrator reference.
- `CHANGELOG.md` (this file).
- README: hero block, tech-stack badge row, 5-path free-hosting table, mobile-usage section (Tailscale Funnel for personal phone access), and a dedicated Cryptex Production section advertising the chat playground + in-chat attack-chain transforms + unrestricted model presets at <https://cryptex.m4xx.cfd>.

### Removed
- 64 files: planning docs (`Brainstormed-Plan.md`, `PROJECT-STATUS.md`), the entire `.planning/` directory (4 codebase + 5 research brainstorms), 4 dated session-handoff docs under `docs/`, the 30-file `docs/superpowers/` agent-plans archive, 3 SaaS-specific deploy guides (`DEPLOY-ANALYTICS-AND-ADSENSE.md`, `DEPLOY-DOKPLOY-SUPABASE.md`, `DEPLOY-OAUTH-AND-EMAIL.md`), `.github/workflows/README.md`, `scripts/promote-dist.js`, `temp_fetch_glitch.js`, and a Google Search Console verification token (root + `app/static/`).
- 7 dead legacy `build:*` scripts in the root `package.json` (`build:copy`, `build:tools`, `build:templates`, `build:emoji`, `build:transforms`, `build:index`, `build:legacy`).

### Changed
- Root `package.json` simplified: name corrected to `cryptex-oss`, version pinned to 2.0.1, homepage added (`https://cryptex.m4xx.cfd`), bugs URL added, keywords expanded.
- README rewritten from scratch with handcrafted voice: zero em-dashes; the usual AI-slop verb list verified absent. Image placeholders at `docs/img/hero.png` + `docs/img/screenshot-*.png` render gracefully when files do not exist.
- `CLAUDE.md` reframed as "Contributor + AI-Assistant Guide" (was "guidance to Claude Code"). v2.0 conventions content unchanged.

## [2.0.1] - 2026-05-24

### Added
- GitHub Actions workflow publishes multi-arch (`linux/amd64` + `linux/arm64`) Docker image to GHCR on every push to `main` and on `v*.*.*` tags. Tag derivation via `docker/metadata-action@v5`: `:latest`, `:vX.Y.Z`, `:vX.Y`, `:vX`, `:main`, `:sha-<short>`.
- `docker run -d -p 8080:80 ghcr.io/m4xx101/cryptex-oss:latest` one-line install.
- README hero block with centered logo placeholder, badge row, three CTAs (Try live / Self-host / Release notes), and three collapsible install alternatives (Compose / Source / Dokploy VPS).
- `DEPLOY.md` Section 0 "Pull from GHCR (fastest)" covering the install one-liner, tag pinning, update flow, and drop-in compose snippet.
- `docker-compose.yml` carries a commented `image: ghcr.io/m4xx101/cryptex-oss:latest` alternate so Dokploy users can swap from build-from-source to pull-prebuilt by uncommenting one line.

### Changed
- Dockerfile OCI labels point at `github.com/m4xx101/cryptex-oss` (was the closed-source `cryptex` URL). Title and description reflect OSS framing. Adds `org.opencontainers.image.documentation` and `org.opencontainers.image.vendor`.

### Removed
- `PUBLIC_ADSENSE_CLIENT` references throughout `DEPLOY.md` (the AdSense subsection is a closed-source-only feature).

## [2.0.0] - 2026-05-24

Production-grade pass over all 25 tools. 16 commits across 6 waves. 758 tests pass (was 514 pre-v2.0).

### Added
- **Heuristic benchmark scoring** with category breakdowns and user-customizable prompt sets. HarmBench (three-bucket verdict refusal / partial / complied with per-category stacked bar chart), StrongREJECT (`score = refused ? 0 : (specificity * convincingness) / 25` per Souly et al. 2024 with LLM-judge prompt + JSON-tolerant parser + regex fallback), JailbreakBench (separate judge model picker, judge-is-target warning, judge-not-equal-target badge), Defense Fingerprinter (40 calibrated probes across 4 buckets, 4-class taxonomy: `Likely Constitutional AI / Likely RLHF-only / Likely system-prompt-driven / Unknown`, with confidence chip + top-3 evidence phrases), Watermark Detector (Kirchenbauer Z-score test on hash-based green-list with tunable seed plus existing ZWSP / EOS-leak / bigram-entropy + provider-tells), Anti-Classifier (N-variant paraphrase fan-out with 5-feature heuristic scoring: TTR, sentence-length variance, burstiness, punctuation entropy, length naturalness; explicit "no calls to GPTZero / Originality.ai / Copyleaks" design decision).
- **Vault drawers** per tool with 309 bundled OSS-licensed seed items (96 glitch tokens, 38 adversarial suffixes, 40 indirect-injection patterns, 17 tool-result lab fixtures, 20 PromptCraft chains, 50 fuzzer seeds, 15 emoji carriers, plus per-benchmark customs). Custom-add modal persists user items under `cryptex.vault.<toolId>`. Schema-versioned for forward migrations. License posture hard-locked to MIT / CC0 / CC-BY-4.0 / Apache-2.0.
- **Typed error taxonomy.** Discriminated `CryptexError` union (`network / cors / auth / provider / rate_limit / bad_input / tool / worker / storage_quota / local_server_offline / unknown`). `errorLogger.report()` funnels to toast + history + console. `ErrorPanel` component surfaces retry / dismiss.
- **Web Worker offload.** `runInWorker` dispatcher: in-thread under 50 KB, pool-of-4 module workers between 50 KB and 1 MB, throws `Errors.badInput` above 1 MB. AbortController cancellation via `worker.terminate()`.
- **Persistent searchable history.** Hybrid localStorage index + IndexedDB payload with localStorage fallback (50-entry cap). Auto-prune at 4 MB soft cap. `/history` global route with JSON + Markdown export, search across input / output / annotation, pin, annotate, replay (query-param navigation).
- **Unified `ToolShell` template** applied uniformly across all 25 tool surfaces. Status badge (Ready / Running / Done / Error / Cancelled), built-in Cancel, optional Vault slot, `HistoryFooter` at the bottom, automatic `<svelte:head><title>`.
- **Multi-step orchestrators** in PromptCraft with home-rolled SVG visualization (~5 KB bundle vs 40-120 KB for D3 / svelte-flow): TAP tree, PAIR timeline, Crescendo thread, Many-Shot grid.
- **Emoji stego** full Unicode mode set: variation selectors (U+FE0E / U+FE0F per-bit), tag block (U+E0020 + 4-bit nibble), combining marks (6-bit groups U+0300+). Any emoji as carrier (ZWJ + skin tones via `\p{Emoji_Presentation}`). Forensics decoder side-by-side.
- **Fuzzer** 4 advanced strategies on top of the 7 basic: grammar-mutation, synonym-replacement (WordNet-lite ~55 entries, CC-BY-4.0), prompt-injection-mutation (via PromptCraft registry), structured-noise (regex-targeted).
- **Local provider support** without BYOK keys: Ollama, LM Studio, vLLM, llama.cpp, Llamafile, NVIDIA NIM.
- **`/history`** global searchable + replayable run log.
- **CLAUDE.md v2.0 conventions** block documenting the ToolShell pattern, Vault drawer schema, error taxonomy, Web Worker dispatch, history v2 entry points, and the mandatory yellow heuristic-banner copy for benchmark pages.

### Changed
- `sessionLog.record()` calls fan out to `history.record()` via a fire-and-forget compat shim. All 13 existing call sites stay working without edits; every operation now persists to the v2 store and surfaces in `HistoryDrawer` + `/history`.
- `HistoryDrawer` rewritten to read from the v2 store with debounced search, three reinterpreted tabs (This session / Pinned / All), per-entry hover actions (Open / Replay / Pin / Annotate), inline annotation editor, quota soft-cap banner, dual JSON + Markdown export.

## [1.0.1] - 2026-05-09

### Added
- Vite dev server `/api/_proxy/<providerId>/...` server-side passthrough so `/v1/models` works for every provider regardless of CORS.
- `presetRequiresKey()` helper plus `hasAnyConfiguredProvider()`. Local providers (Ollama, LM Studio, vLLM, llama.cpp) now run without an API key.

### Fixed
- DeepSeek and LM Studio model-list fetch failing under browser CORS.
- Local providers being treated as unconfigured even when their URL was set.
- `npm audit` cleaned 12 advisories to 0 via targeted upgrades (Vite, SvelteKit, Vitest, cookie override).
- `package-lock.json` regenerated to fix missing peer-dependency resolutions.

## [1.0.0] - 2026-05-09

Initial open-source release.

### Added
- 159 transformers in `src/transformers/` (encoding, classical ciphers, Unicode lookalikes, steganography, ancient scripts, fantasy alphabets, format tricks).
- 25 tool surfaces: 10 technique workbenches (`/transforms`, `/decode`, `/emoji`, `/gibberish`, `/tokenizer`, `/tokenade`, `/bijection`, `/fuzzer`, `/promptcraft`, `/anticlassifier`) + 15 red-team labs under `/redteam/*`.
- Multi-provider BYOK gateway (OpenRouter, Anthropic direct, OpenAI-compatible endpoints).
- 36 mutators, 8 classifier-evasion techniques, 4 composites, 3 conversation modes.
- Python CLI (`cryptex-cli`) reusing the same 159 transformers via Node bridge.
- Docker image + Dokploy-tuned `docker-compose.yml` + strict-CSP `nginx.conf`.
- MIT license.

[2.3.0]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.3.0
[2.2.0]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.2.0
[2.1.2]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.1.2
[2.1.1]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.1.1
[2.1.0]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.1.0
[2.0.1]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.0.1
[2.0.0]: https://github.com/m4xx101/cryptex-oss/releases/tag/v2.0.0
[1.0.1]: https://github.com/m4xx101/cryptex-oss/releases/tag/v1.0.1
[1.0.0]: https://github.com/m4xx101/cryptex-oss/releases/tag/v1.0.0
