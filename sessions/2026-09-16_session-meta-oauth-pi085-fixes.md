# Session Closeout — 2026-09-16 — pi-meta-oauth: pi 0.85 support, 400 fix, entitlement probe

## 1) TL;DR

- Adopted Meta's `pi-meta-oauth` extension (npm 0.6.1) as our own maintained fork: `antonioc-cl/pi-meta-oauth`, local copy in `~/Documents/PROYECTOS/pi-extensions/pi-meta-oauth` (path-installed in pi, no copy/symlink — live edits apply on restart).
- Verified the extension against installed pi 0.85.1: typechecks, 36 tests pass.
- Opened upstream issue #17 and PR #18 with the compat fixes (peer range, test casts, encrypted-reasoning handling).
- Diagnosed the `/login meta` → HTTP 400 `reasoning encrypted_content was not issued to this caller`: OAuth-minted keys aren't always entitled to encrypted reasoning replay.
- Replaced the hard-coded strip with a per-key entitlement probe (200 → keep include; 400 mentioning encrypted_content → strip; else safe-strip + 5-min cooldown retry). Live probe on the real key returned **entitled → continuity ON**.
- Documented the mechanism in the README.

## 2) Goals vs Outcome

**Planned goals**

- Read Meta Model API quickstart; find a way to use Muse Spark on pi.
- Adopt/own a pi extension for Meta OAuth; submit issues/PRs upstream.
- Fix the login-then-400 failure.
- Best long-term solution for encrypted-reasoning entitlement (probe, not static flag).

**What actually happened**

- All achieved. Quickstart page is JS-rendered and unfetchable; used the `/docs/coding-agents` markdown as source of truth (Responses API, `include: ["reasoning.encrypted_content"]`, 24h prompt cache).
- No `pi-muse` extension exists; `pi-meta-oauth` (BlockedPath) implements exactly this via OAuth device flow + `/muse-code/key`.
- Our fork: peer range widened to `<0.86.0`, devDeps 0.85.1, 4 legacy-shape test fixtures cast to `as unknown as RefreshModelsContext`, then include-strip, then entitlement probe + README.
- Modalities fix (pdf/video) proposed then **reverted**: pi's `Model.input` type only allows `"text" | "image"`; upstream's limitation is correct for pi.

## 3) Key decisions (with rationale)

- **Decision:** Maintain as fork under `~/Documents/PROYECTOS/pi-extensions/pi-meta-oauth` (in the pi-extensions collection), with `origin` (our fork) + `upstream` remotes.
  - **Why:** Local path packages are resolved live by pi (`install` only adds the settings.json entry, no copy) — edits apply on next pi restart, no reinstall.
  - **Tradeoff:** Untracked local `bun.lock` appears on `bun install`; repo ships without a lockfile.
  - **Status:** confirmed
- **Decision:** `applyMetaResponsesCacheHints` gets `keepEncryptedReasoning` param; default still strips.
  - **Why:** backwards-compatible with existing tests; hook explicitly passes the probe outcome.
  - **Status:** confirmed
- **Decision:** Entitlement probe fires in the background on first request per key; in-process FNV-1a key-hash cache; 5-min cooldown after inconclusive probes.
  - **Why:** self-healing across daily key rotation; no env vars/credentials plumbing; cheap (~16 tokens, `max_output_tokens` ≥ 16 is Meta's floor — 8 400s).
  - **Tradeoff:** first request after pi restart may strip before the probe lands; inconclusive probes keep safe-strip until retry.
  - **Status:** confirmed
- **Decision:** Reverted pdf/video modalities after typecheck failed — pi type system only models text/image.
  - **Why:** pi doesn't strip parts by declared modality (no `"pdf"` reference in responses adapter); declaration would fight the type system.
  - **Status:** confirmed

## 4) Work completed (concrete)

- Forked `BlockedPath/pi-meta-oauth` → `antonioc-cl/pi-meta-oauth`; cloned to `~/DEV` then rsynced (with `.git`) into `~/Documents/PROYECTOS/pi-extensions/pi-meta-oauth`.
- Commits on `fix/pi-0.85-peer-range` (4, all pushed to origin → PR #18 updated automatically):
  - `5db8719` — fix(meta): support pi 0.85 (peer range `<0.86.0`, devDeps 0.85.1, 4 test-fixture casts)
  - `270540f` — fix(meta): drop encrypted_content include on OAuth-minted keys
  - `b849411` — feat(meta): probe encrypted-reasoning entitlement per key
  - `af5a9c3` — docs: document prompt caching and encrypted-reasoning entitlement probe
- Upstream: issue #17 (peer range rejects 0.85), PR #18 (the fixes).
- Installed in pi via settings.json `packages` entry `../../Documents/PROYECTOS/pi-extensions/pi-meta-oauth`; `pi --list-models meta` shows 7 live models (incl. muse-image-1.0, muse-voice-transcribe-1.0 from live catalog).
- `/login meta` completed (auth.json has oauth credential; key re-mints daily).
- Files touched: `extensions/meta.ts`, `tests/meta-cache.test.ts`, `tests/meta.test.ts`, `package.json`, `README.md`.

## 5) Changes summary (diff-level, not raw)

- **Added:** entitlement probe (`probeEncryptedReasoningEntitlement`, FNV-1a hash cache, `scheduleEntitlementProbe`/`keepEncryptedReasoningFor`), async `before_provider_request` hook, `keepEncryptedReasoning` param on `applyMetaResponsesCacheHints`, 5 new tests (keep/strip split, 3 probe outcomes), README section.
- **Changed:** peerDependencies `>=0.83.0 <0.86.0`; devDependencies pi-ai & pi-coding-agent → 0.85.1; 4 `satisfies RefreshModelsContext` → `as unknown as RefreshModelsContext`; hook test awaits async handler.
- **Removed:** unconditional include-strip (replaced by probe-driven strip).
- **Behavioral impact:** on entitled keys, reasoning continuity now survives across turns (docs-recommended); on unentitled keys, requests never 400. First request per pi start may lack continuity until the background probe resolves.
- **Migration/rollout notes:** none — path-installed; restart pi to load new hook code.

## 6) Open items / Next steps

- **Task:** Decide whether to merge `fix/pi-0.85-peer-range` into our local `main` (or wait for upstream review of PR #18).
  - **Owner:** user
  - **Priority:** P1
  - **Suggested approach:** `git checkout main && git merge fix/pi-0.85-peer-range` locally; upstream `main` stays at v0.6.1.
- **Task:** Decide whether to delete the duplicate `~/DEV/pi-meta-oauth` checkout (two copies of the repo exist).
  - **Owner:** user
  - **Priority:** P2
  - **Suggested approach:** `rm -rf ~/DEV/pi-meta-oauth` if the pi-extensions copy is authoritative.
- **Task:** Verify Muse in real pi usage (switch to `meta/muse-spark-1.3`, run a multi-turn agentic task, confirm reasoning continuity + no 400).
  - **Owner:** user
  - **Priority:** P1
  - **Blockers:** none.
- **Task:** `bun.lock` untracked in repo — decide to ignore or delete locally.
  - **Owner:** user
  - **Priority:** P2

## 7) Risks & gotchas

- Probe is in-process only: after pi restart, entitlement is unknown until first meta request re-probes; stale key continues stripping/keeping until probe lands.
- `max_output_tokens: 8` in the probe 400s (Meta requires ≥16) — already fixed to 16; do not "optimize" back down.
- Hook is now async and calls `ctx.modelRegistry.getProviderAuth()` per meta request; fallback try/catch keeps it safe in test fakes lacking the registry.
- Contributor models train on your data (Meta terms) — prefer standard `muse-spark-1.3`.
- `include: ["reasoning.encrypted_content"]` is honored only on the Responses surface; Chat Completions (~0% cache, no continuity) should stay unused.

## 8) Testing & verification

- `bun run typecheck` → clean (devDeps 0.85.1, strict, skipLibCheck — matches repo CI exactly).
- `bun test` → 36 pass / 0 fail (cache file 18/18). Live-key tests skip without a credential.
- Live wire tests vs `https://api.meta.ai/v1/responses` with the real auth.json key: pre-fix include shape 200 now; reasoning call returned `Paris` with 349 reasoning tokens; `probeEncryptedReasoningEntitlement(real key)` → `true`.
- Suggested next-session test plan: multi-turn loop on `meta/muse-spark-1.3` (verify continuity), rotate key and confirm re-probe, run probe with a stripped-entitlement key if one exists.

## 9) Notes for the next agent

- If you only read one thing: `extensions/meta.ts` — `probeEncryptedReasoningEntitlement` + the `before_provider_request` hook are the whole 400/continuity story.
- pi internals relied on: `ModelRegistry.getProviderAuth()` (hook credential access), `RefreshModelsContext` = `stored` + required `publish` (0.84+ shape; 0.83 `store` gone), `OAuthCredentials` open index allows extra fields.
- The extension is installed live from this folder: edit → `bun test` → ask user to restart pi. No `pi install` needed.
- Upstream PR #18 will show 4 commits; if the maintainer merges, `git pull upstream main` descends cleanly (our commits are upstream's base + ours).
- pi.swift pipe: quickstart page is JS-rendered; use `/docs/*.md` variants (e.g. `dev.meta.ai/docs/coding-agents.md`) for fetchable docs.