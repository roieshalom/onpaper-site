# Processing-animation redesign — plan & findings

**Status:** Step 1 (inspection + plan) DONE. Implementation NOT started.

---

## 0. Workflow (do this FIRST every implementation session)

**Never edit `main` directly** — `main` publishes to production (`onpaper.fit`, GitHub Pages).

Environment split (all keyed at runtime by hostname; SAME `index.html` serves both):
- **Production** = GitHub Pages `onpaper.fit`. Untouched until merge.
- **Staging/beta** = separate Vercel project `onpaper-site*.vercel.app` — `noindex`, red "STAGING"
  edge bar, and it talks to the **preview API twin** (cost-isolated pool). Detected by
  `IS_BETA = /^onpaper-site/i.test(location.hostname)` (`index.html` ~L1560).

Release path:
1. Branch off `main` (e.g. `processing-animation`). Do NOT build on `main`.
2. Push the branch → Vercel auto-builds a preview at `onpaper-site-git-<branch>…vercel.app` = the
   staging URL (STAGING bar + API twin). Verify the animation there.
3. When approved, merge branch → `main` → GitHub Pages publishes to `onpaper.fit`.
4. Rollback if needed = `git revert` of the one merge/commit; prod returns to the old loader instantly.

The animation code is environment-agnostic, so what's approved on staging ships byte-for-byte.
Currently untracked on `main`: `.claude/`, `.vercel/`, `DESIGN.md`, this plan `.md` — leave them out of the animation branch's commits unless intended.
**Goal:** Replace the current CV↔JD "waves" loader with a calmer, more editorial
"two documents being read → brought together → compared → gaps found" animation.
Smoothness + professionalism over copying the reference screenshot exactly.

Single-file app: everything lives in **`index.html`** (~222 KB, no build step, no framework).
Deploys via Vercel (static). No dependencies may be added.

---

## 1. How the current flow works (verified)

### Views
- `#viewInput` — the two input cards (`.op-io-sec` → `.op-io` with two `.op-col`).
  - **CV (left):** `#cvDrop` (upload button) / `#cvChip` (parsed-file chip) / `#cvPasteText`
    — whichever is visible is the "left card". Helper `visibleLeftCard()` (~L2262) picks it.
  - **JD (right):** always `.op-jd-card` (`#jdCard`).
- `#viewResults` — the gap report. Shown by `showResults(text, isSample)`.
- Docked action bar `#actionBar`: run button `#fitRun` (label `#fitRunLabel`, spinner `#fitSpin`).

### Run lifecycle (the hook points — DO NOT change these)
- Button click (`els.run`, ~L3375) → `attemptRun(false)` → **`doRun(opts)`** (~L3114).
- `doRun` calls **`setLoading(true)`** (~L2296) then `await callClaude(payload)` (~L3014, the fetch).
- **`setLoading(on)`**:
  - scrolls `.op-scroll` to top, sets `loading`, toggles button `.is-loading` + `disabled`,
    shows/hides `#fitSpin`, `lockInputs(on)`, and calls **`startLoaderOverlay()`** / **`stopLoaderOverlay()`**.
- **`startLoaderOverlay()`** (~L2267): builds `new OnPaperLoader({ leftCard, rightCard, color:--red, ink:--ink })` and `.start()`.
- **`stopLoaderOverlay()`** (~L2278): `opLoader.stop(); opLoader = null;`
- **Success:** enforces `MIN_THINK_MS = 2500` floor (L1777) so a fast reply still reads as work,
  then `setLoading(false)` → `showResults()`.
- **Errors** (all inside `doRun` catch):
  - `blocked` + `fallback_disabled` → setLoading(false) + inline paste redirect.
  - `device_limit`/`ip_limit`/`turnstile_failed`/`pool_exhausted`/`rate_limited` → setLoading(false) + wall/modal.
  - legacy `blocked` (fallback on) → **keeps loader running**, calls `doRun({search:true})` (loader is reused; start is a no-op while one exists).
  - `search_failed` → `idleRunButton()` (releases button, **keeps overlay + lifted cards**) + opens `#jdFailBack` modal.
    Overlay is torn down later in **`closeConfirm`** (~L3508) only when `back === els.jdFailBack`: `stopLoaderOverlay(); lockInputs(false)`.
  - generic error → setLoading(false) + `#failBack` modal.

### Supporting CSS (input view during a run) — L229–252
- `#viewInput.is-locked .op-io-sec` → `opacity:.5; filter:blur(1.5px) saturate(.9); pointer-events:none`.
- `#viewInput.is-locked` → `transform: translateY(-230px)` **only** `@media (min-height:680px)` (lifts cards so animation clears the modal).
- `@media (prefers-reduced-motion: reduce)` → disables the lift + transition.

### The current animation = `OnPaperLoader` (~L2147–2265) — THIS is what we replace
- IIFE returning a `Loader` constructor. Public API used by the app:
  - `new OnPaperLoader({ leftCard, rightCard, color, ink })`
  - `.start()` — appends a `position:fixed; inset:0; width:100vw; height:100vh; z-index:4` SVG to `<body>`, starts rAF.
  - `.stop()` — fades SVG opacity to 0 over 350 ms, then removes it + cancels rAF.
- Per frame (`_frame`): reads `left`/`right` `getBoundingClientRect()`, decides `horizontal`
  (cards side-by-side, desktop) vs vertical (stacked, mobile), draws:
  - animated stroke **rectangles** tracing each card border (`_drawRect`, dashoffset intro),
  - 3 wavy **ray** paths (`_wavePath`) between the cards in `--red`, each with a base + a
    traveling dash "pulse". Loops forever via `loopDur`/phase; no fixed total.
- **z-index note:** SVG is z-index 4; action bar is z-index 5; modals are z-index 46. Keep this ordering.

---

## 2. Palette facts (from `:root`, L150–205)

- `--red #ed2c0a` (sole accent), `--red-dark #c41f00`
- `--ink #111A28` (navy), `--ink-3 #384A66`, `--ink-4 #4E6180`, `--faint #8793A3`
- `--paper #FFFFFF`, `--paper-bg #F0F4F7`, `--paper-lt #E3EAF0`
- `--ok #4F9E6E` (success green), `--line-in rgba(17,26,40,.14)`
- **No blue / lavender / purple tokens exist.** Only soft blue in `::selection` = `rgba(143,180,217,.30)`.
- Fonts: `--serif-d` Libre Caslon Display, `--serif-b` Newsreader, `--mono` IBM Plex Mono.
- Motion easing token: `--ease-lift: cubic-bezier(.22,1,.36,1)`.

**Decision:** introduce processing-ONLY colors (do not touch global `:root`, do not re-skin site):
- CV stream (blue): `#8FB4D9` family (matches existing selection blue).
- JD stream (lavender/purple): `#A99BD6` / `#B4A7DE` family (new, muted, low-sat to fit).
- Keep the tiny coral accent (`--red`) for the center node's single spark only; green `--ok` reserved for completion.
Keep all opacities low; no glow/neon/gradients/glassmorphism/3D/blur beyond existing.

---

## 3. Redesign concept (the animation to build)

Keep the two cards recognizable and roughly in place (reuse existing lift + dim). Everything
new is drawn in the same full-viewport SVG overlay, tracking the two cards' live rects each frame.

- **Stage 1 — scan (~1–1.5 s):** a soft highlight sweeps a few "text lines" inside the CV card,
  then the JD card (CV → JD). Cards breathe ~1–2 px. (Draw faux line-highlights as short rounded
  rects positioned inside each card's rect; or CSS shimmer. Prefer SVG so it tracks geometry.)
- **Stage 2 — streams:** 3–4 very thin curved SVG paths from CV→center (blue) and JD→center
  (lavender). Small glowing dots travel along them toward the center (dash pulse, like today).
- **Stage 3 — center node:** a minimal 4-point sparkle + 2–3 thin concentric rings. Fades/scales
  0.9→1, gentle pulse, rings expand/contract subtly, one ring rotates very slowly. Editorial, not sci-fi.
- **Stage 4 — compare:** dots flow into the node; a few pass through and emerge the other side.
  Occasional small **monospace** words appear near center (one at a time, 500–800 ms, varied position;
  rarely two): `skills experience keywords tools impact systems design interaction`. Not pills.
- **Stage 5 — loop (MOST IMPORTANT):** indefinite, non-restarting cycle (scan → stream → compare →
  word → deeper → stream → word…). ~3–5 s per cycle with slight timing jitter. Must look identical
  whether the API takes 2 s or 20 s. (Today's rAF phase loop already does this; keep that spirit.)
- **Stage 6 — completion (~500–900 ms):** dots accelerate inward, node gives one stronger pulse,
  rings expand + fade, streams retract toward cards, words fade, node fades → results. Implement by
  enhancing `.stop()` to play a short "settle" before the opacity fade (MIN_THINK_MS gives us the room).
- **Status line: NONE.** (User decision, confirmed.) Let the animation speak for itself — do NOT add
  "Finding the gaps…" or any processing text. The extracted comparison words near the center (Stage 4)
  remain, but no fixed status/ellipsis line.

### Accessibility
- `prefers-reduced-motion: reduce`: no traveling dots, no rAF churn, no big movement. Show a static
  center node + streams at rest with a gentle opacity pulse, and the status line. Still communicates "working".
  (Current class ignores reduced-motion — MUST add a branch. Detect via `matchMedia`.)

---

## 4. Implementation strategy (smallest safe change)

**Only touch the isolated component. Do not change `doRun`, `setLoading`, the fetch, walls, results, analytics.**

1. **Rewrite the `OnPaperLoader` IIFE body** (L2147–2265), keeping the exact public API:
   `new OnPaperLoader({leftCard, rightCard, color, ink})`, `.start()`, `.stop()`.
   - Optionally accept new opts (blue/lav colors) but default them internally so callers need no change.
   - Add reduced-motion handling inside the class.
   - Enhance `.stop()` to accept nothing (same signature) but play the ~600 ms completion settle,
     then fade+remove. Callers (`stopLoaderOverlay`) stay identical.
2. **Add scoped CSS** for the new pieces (status line, any faux text-line shimmer) in the existing
   `<style>` block, namespaced (`.op-proc-*`). Reuse `.is-locked` lift/dim as-is.
3. **No status line** (user decision) — skip any processing text element.
4. Keep z-index 4 for the SVG, below the bar (5) and modals (46).

**Files:** `index.html` only.
**Lines to edit:** L2147–2265 (class). Possibly small CSS additions near L229–252. No other JS.

---

## 5. Verification checklist (after implementing)

Run locally (static): `python3 -m http.server 8000` in repo, open via Browser pane, or use the app's
mock preview. Note: `?mock=1` (or `#mock`) jumps straight to a SAMPLE result and **short-circuits the
run button** (`els.run` handler returns early on MOCK) — good for results, NOT for seeing the loader.
To see the loader you need a real run (or temporarily stub `callClaude` with a delayed promise — revert after).

1. Animation starts on "Check fit" click.
2. Loops indefinitely & smoothly while request pending (test 2 s / 5 s / 20 s via stubbed delay).
3. No visible restart between cycles; no "finished-waiting" dead state.
4. Success → graceful completion transition → results (existing `showResults` unchanged).
5. Error paths still work: generic fail modal, blocked→search retry (loader reused), search_failed
   (loader persists behind `#jdFailBack`, torn down on dismiss). Do not "complete" on error.
6. Results content unchanged.
7. Desktop layout unchanged (cards side-by-side, horizontal streams).
8. Mobile: stacked cards, vertical streams, no horizontal overflow, node scaled down.
9. `prefers-reduced-motion`: no moving particles, opacity-only, still communicates processing.
10. No console errors. Smooth on a normal laptop (rAF, transforms/opacity only).

---

## 6. Guardrails (from the brief)

- No new dependency/library; CSS + SVG + rAF only. No canvas/GIF/video/Lottie.
- Do not change: existing visual design outside processing, CV/JD input behavior, API/analysis/coupon/
  usage/analytics/results logic, error handling (only integrate).
- Don't fake API state — reuse the existing `setLoading`/`doRun` lifecycle (single loading state).
- Desktop-first; don't break or redesign mobile.
- No generic spinner / progress bar / percentage / robot / brain / three-dot / neon / heavy glow.

## 6b. Dev harness — detach from Claude + start/stop (build this FIRST on the branch)

Requirement: during animation work the loader must be exercisable **without calling Claude** — no
API request, no run consumed, zero cost — and with a **start/stop control** since we'll be
starting/stopping the animation constantly. Staging already uses the cost-isolated API **twin**, so
even accidental runs there don't touch prod cost; this harness removes cost entirely.

**IMPLEMENTED** on branch `processing-animation` (commit at ~`index.html` L3376, block
`devAnimHarness`). Key point: **no-cost is the DEFAULT on this branch — no flag required.** The cost
being separated is the Claude API call itself; staging still bills on a real run, so opt-in wasn't
enough. On this branch the backend call is disabled unconditionally.

**Design (as built):**
- **Guard:** the whole block bails unless `IS_BETA` (staging) or localhost → can NEVER run on
  production (`onpaper.fit`). Verified: with the guard failing, no badge/button, `callClaude` untouched.
- **No-cost by default:** on staging/localhost, `callClaude` is reassigned to a stub that resolves
  `{ text: SAMPLE, remaining: <unchanged> }` after `?delay=<seconds>` (default 4). No `fetch`, no
  `onpaper-api` request, no run spent — regardless of how many times "Check fit" is clicked. Verified
  via network panel (zero `onpaper-api` calls). Balance is returned unchanged so nothing decrements.
- **Start/Stop button:** fixed bottom-left; toggles `setLoading(true/false)` → drives the REAL loader
  (`startLoaderOverlay` → `OnPaperLoader`) with no `doRun`/API — for tuning without filling CV/JD.
- **NO-COST MODE badge:** green fixed badge so the disabled state is unmistakable.
- **`?delay=<seconds>`** tunes the stubbed think-time (test 2 / 5 / 20 s of loop).

**MUST strip (or hard-disable) before merging this branch to `main`** — it's a dev-only block and
would disable real analysis on any `onpaper-site` production build. It's a single clearly-commented
IIFE, easy to remove.

## 7. Decisions
- **Status line: NONE** (confirmed by user). No processing text at all.
- Completion "settle" (Stage 6): include a light version (one stronger pulse + retract) before fade — default, revisit if it complicates.
