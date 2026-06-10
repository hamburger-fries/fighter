# MIRROR MATCH — Final Build Plan (Chrome, Modern Stack)

> The definitive plan. Self-contained: the combat spec is **inlined** below (§ Combat
> Specification) and is the authoritative sim contract for Phases 3–8 — there is no external
> companion spec. Format: Gold Standard Template v4.0, Greenfield.
> **Target runtime: desktop Google Chrome (stable).**
>
> **Concept:** a geometric, **Tekken-5-staged** single-player training fighter. Every character
> shares **one 8-move kit** (the "mirror"); the full **Tekken 5 roster** are data-driven geometric
> skins over that kit. Tekken 5 supplies the *staging* — camera, lighting, stage depth, weight, UI
> — **not** models. **Private/educational build (Tekken © Bandai Namco; not for distribution).**

---

## The Stack (decided — no fallbacks except where noted)

| Layer | Choice | Why this, for a Chrome fighter |
|------|--------|-------------------------------|
| Language / build | **TypeScript 5.x (strict)** + **Vite** | Fast HMR, first-class TS, trivial Chrome dev/build. |
| Package mgr / scripts | **Bun** | Fast installs/scripts; Vite drives dev/build. (npm works identically if preferred — does not affect the shipped bundle.) |
| Rendering | **Three.js `WebGPURenderer`** (single class, **automatic WebGL2 backend fallback**) | Desktop Chrome ships WebGPU stable since v113 → WebGPU is the **primary** backend; the WebGL2 backend is the same `WebGPURenderer` auto-selecting when no `GPUAdapter` is granted (and forced via the `forceWebGL: true` constructor option for the Android-Chrome coverage path). **One renderer, two backends — not two renderers.** TSL for the glow/accent shaders (TSL compiles to WGSL and GLSL, so one shader source feeds both backends). |
| 3D layer | **Bare Three.js — no React/R3F/Svelte** | A 60 Hz game loop wants direct object mutation, not a UI reconciler in the hot path. |
| Game loop | **Hand-written fixed-timestep** (accumulator, dt-clamped) + render interpolation | Frame-rate-independent sim; smooth render. The spine. We do **not** use `setAnimationLoop()` — see the async-init note below. |
| Simulation | **Pure TypeScript, deterministic, serializable** | Headless, unit-testable, rollback-ready. Spec = § Combat Specification. |
| State machine | **Hand-rolled switch over an enum** (not XState) | Frame-perfect, trivially serializable, zero runtime overhead. |
| UI store | **`zustand/vanilla`** | Tiny, framework-agnostic subscribe/select for **presentation state only**. |
| HUD | **HTML/CSS overlay** on the canvas | Health bars, combo counter, debug — plain DOM beats a 3D HUD. |
| Characters | **Data-driven `CharacterConfig`** over one shared rig | Full Tekken-5 roster as geometric skins; sim is character-agnostic. Tekken-5 = *staging* reference, not models. |
| Audio | **Howler.js** | Wraps Web Audio; pooling + Chrome autoplay-unlock handled for you. |
| Async boundaries | **Effect-TS v4 (beta — pinned)**, confined to `src/platform/**` | Typed, retryable asset load + persistence. Never in `sim/`, `loop/`, `render/`. **v4 is beta; pin the exact `4.0.0-beta.78` and treat breaking-change churn as a watched risk — see Assumptions #5 / Execution Notes.** |
| Persistence | **`localStorage`** now; **`idb`** deferred | Settings/keybinds fit localStorage; IndexedDB waits for replays. |
| Physics | **None** | Walls, gravity, juggles, bounces are deterministic sim math. |
| Tests | **Vitest** (sim units + replay determinism) + **Playwright** (one Chrome smoke at Final Audit) | The decoupled sim is fully testable headlessly; Playwright drives real Chrome once — see the WebGPU-in-Playwright note at Phase 12. |

**Considered & rejected — PixiJS v8:** evaluated as the renderer. Rejected: Pixi is **2D-only**
(no 3D scene graph, no perspective camera, no native planar/mirror reflection), and its WebGPU
backend is documented as experimental (it recommends WebGL for production). This game's look is
**true 3D** — geometric `Object3D` rigs, a reflective floor, and perspective framing — and the
plan commits to WebGPU-primary, both of which Three.js delivers natively. (The sim firewall keeps
the renderer swappable if that ever changes; the swap touches only `render/`, `effects/`, and
bootstrap.)

**WebGPU async-init note (load-bearing for bootstrap):** `WebGPURenderer` initializes the GPU
device **asynchronously**. Because we use a hand-written loop (not `setAnimationLoop()`),
`main.ts` MUST `await renderer.init()` and confirm the active backend — read `renderer.backend.isWebGPUBackend` (WebGPU) vs
`renderer.backend.isWebGLBackend` (WebGL2 fallback) — **before** the first tick/render. WebGPU is
the **default** backend (no constructor flag); pass `forceWebGL: true` only to exercise the WebGL2
fallback. The loop never renders before init resolves. (Phase 0 wires this; Phase 1 depends
on it.)

**The load-bearing boundary:** `src/sim/**` is pure and headless (no `three`, DOM, audio,
Effect, timers, network). Everything else reads the sim; nothing writes back into it.

---

## Document Control

- Product: `mirror-match`
- Plan file: `mirror-match-final-plan.md`
- Date: `2026-06-10`
- Plan type: **Greenfield**
- Scope: phased greenfield plan, Phase 0 → Phase 11, then Phase 12 (Final Audit) as a discrete gate.
- Execution rule: a checklist item is `[x]` **only when its work has shipped *and* its evidence
  has been pasted into the phase.** This bold rule repeats at the head of every phase.

---

## Repo Structure Decision

> Greenfield layout. The one immovable boundary is `src/sim/**` (pure) vs everything else
> (impure). Other module names are expected starting seams, not contracts.

**Locked identifiers (do not rename without a plan revision):**
`src/sim/**` (pure firewall), `config/constants.ts`, `config/characters.ts` (roster identity — render/UI only), `src/platform/**` (sole Effect home),
`tests/sim/**`, `tests/replay/**`, `tests/e2e/**`.

```
mirror-match/
  index.html
  vite.config.ts            # WebGPURenderer bundling (WebGPU + WebGL2 backends from one class)
  tsconfig.json             # strict
  .eslintrc                 # no-restricted-imports: src/sim (no three/howler/effect/DOM/timers);
                            #   src/{sim,loop,render} (no effect)
  playwright.config.ts      # channel: 'chrome' + WebGPU launch flags (see Phase 12)
  public/audio/             # sfx + ambience (loaded via platform/assets)
  src/
    main.ts                 # bootstrap: await renderer.init() → loop + HUD + stores
    config/constants.ts     # tick rate, gravity, speeds, arena bounds, move tables, tunables
    config/characters.ts    # CharacterConfig roster (build/colors/accent per Tekken-5 char) — render/UI only, never sim
    sim/                    # PURE / HEADLESS — firewall applies. Spec: § Combat Specification.
      types.ts  step.ts  stateMachine.ts  fighterState.ts  serialize.ts
      input/{inputTypes,inputBuffer,commandParser}.ts
      moves/{moveTypes,moveData}.ts
      combat/{hitDetection,comboManager,damageScaling,juggle,wallBounce,knockdown}.ts
      # rng.ts is DEFERRED — not created until a sim consumer exists (see Error Correction EC-1).
    loop/gameLoop.ts        # fixed-timestep accumulator + dt-clamp + interpolation alpha
    render/                 # reads sim, writes three — never the reverse
      renderer.ts           # WebGPURenderer (async init; auto WebGL2 backend), scene, lights
      camera.ts  rig.ts  fighterView.ts  arena.ts  interpolate.ts
      effects/{hitSpark,energyParticles,jointFlash,cameraShake}.ts   # energyParticles tinted by CharacterConfig.accentColor
    audio/audioManager.ts   # Howler wrapper, sfx triggers, Chrome autoplay unlock
    ui/{hud,healthBars,comboCounter,inputDisplay,debugPanel,characterSelect}.ts  ui/{hud,characterSelect}.css
    state/uiStore.ts        # zustand/vanilla — presentation state only
    platform/{assets,persistence}.ts   # Effect-TS lives here ONLY
    training/dummy.ts       # scripted dummy controller (toggles → InputFrames)
  tests/
    sim/                    # headless unit tests
    replay/                 # recorded input fixtures + determinism test
    e2e/                    # Playwright Chrome smoke (Final Audit)
```

---

## Problem Statement

There is no codebase. We need a playable single-player **training mode** for a 2.5D geometric
fighter that runs at 60 FPS in desktop Chrome, built on a bare-Three.js `WebGPURenderer` over a
deterministic, fixed-timestep, headless simulation. A human **picks any character from the full
Tekken 5 roster**, fights a **configurable dummy** (also any roster character), using all eight
moves of the **one shared kit**. "Done" is: every combat system (reactions, launchers, juggles,
sweeps, knockdowns, wall bounce, combo scaling) behaves correctly and is unit-testable with no
renderer attached; both selected characters render as **Tekken-5-staged geometric skins** over
the shared rig; the **sim carries no character identity**; and the whole thing holds 60 FPS in
Chrome.

---

## Goals

> Numbered, verifiable. Each maps to one or more Acceptance Targets (AT-XX) and Functional
> Requirements (FR-XX) below.

1. **G1 — Sim purity.** `src/sim/**` imports nothing from `three`, `howler`, `effect`, the DOM, timers, or any I/O; the firewall lint rule is green.
2. **G2 — Determinism.** A recorded input log replayed twice on the same build yields an identical state hash.
3. **G3 — Render-rate independence.** Sim ticks/wall-second are invariant under render throttling (±1 tick/s).
4. **G4 — Combat correctness.** All eight moves, the connect table, the guard matrix, juggles, wall bounce, knockdowns, and combo scaling behave per § Combat Specification and are proven headlessly.
5. **G5 — Observable play.** A human plays any selected roster character vs a configurable dummy character in Chrome; reactions render correctly.
6. **G6 — Training surface.** Character-select (player + dummy) works; HUD shows live state; dummy toggles work; settings/keybinds persist across reload.
7. **G7 — Performance.** Desktop Chrome holds ≥ 60 FPS under a full-combat scene on the documented host; `step()` stays within the Phase 1 budget.
8. **G8 — Baseline integrity.** No dead code, unused params/imports/exports; removed tests are replaced same-commit or justified.
9. **G9 — Full roster, geometric.** Every Tekken-5 character is selectable for player and dummy and renders from a `CharacterConfig` over the one shared rig; the sim stays character-agnostic (identity is render/UI only).

---

## Non-Goals

- Online / netcode (sim is rollback-*ready*; no transport, prediction, or rollback ships here).
- Cross-browser parity beyond Chrome. Firefox/Safari may work via the WebGL2 backend but are untested and out of scope.
- Z-axis sidestep movement (lane locked for MVP).
- **Unique per-character movesets.** Every character shares the one 8-move kit (the *mirror*); identity is cosmetic (build/proportions, palette, name, accent shape) — not new moves or frame data.
- Real CPU AI (the dummy is scripted toggles).
- Postprocessing stacks, **imported character models, rigged/IK/skeletal animation** (decided: characters are procedural geometric primitives; Tekken-5 is a *staging* reference, not an asset source).
- A physics engine.
- Rounds/victory flow beyond a timer placeholder.
- Cross-platform / cross-client determinism (floats are deterministic for **same-build replay** only; true rollback would need fixed-point — out of scope, see Assumptions #6).

---

## Hard Constraints

- **Evidence-driven gating.** A phase closes only when its gates pass with concrete evidence pasted in. No `na` on a required gate.
- **Failure protocol is in-phase or revert.** Fix within the phase, or revert the phase's commits and revise the plan. No emergency sub-phases.
- **Sim purity firewall (load-bearing).** `src/sim/**` is pure/headless: no `three`, `howler`, `effect`, `document`, `window`, timers, or network. Enforced by an ESLint `no-restricted-imports` rule scoped to `src/sim`, verified with `rg`.
- **Determinism.** The tick is `step(state, inputs) => state`: no `Date.now()`, no `Math.random()` (seeded RNG only, and only once a consumer exists), no ambient mutable globals. State is JSON-/typed-array-serializable. **Determinism scope = same-build replay on one machine**, not cross-platform (Non-Goal).
- **Fixed timestep.** Sim advances only in fixed 60 Hz ticks via a **dt-clamped** accumulator (clamp frame delta to ≤ 250 ms and cap sub-steps per frame to prevent spiral-of-death). Render interpolates the last two sim states. Render rate never alters sim outcomes.
- **One-way data flow.** Render and UI read sim state, never write it. The `zustand/vanilla` store holds presentation state only; combat state lives solely in the sim.
- **Effect-TS only at boundaries** (`src/platform/**`). `sim/`, `loop/`, `render/` import it nowhere. **Pin the exact Effect v4 beta version** (it is pre-stable; treat upgrades as deliberate, tested events).
- **Frame-data units.** All timing inside the sim is integer frames at the tick rate. Milliseconds live only in the loop/clock layer.
- **WebGPU is primary; WebGL2 is the auto/forced fallback backend of the same renderer.** Desktop Chrome uses the WebGPU backend. The WebGL2 backend exists for Android-Chrome coverage (and as automatic graceful degradation) and is documented, not a runtime A/B for the primary target.
- **Async renderer init before the loop.** `await renderer.init()` resolves and the active backend is logged **before** the first tick/render (WebGPU inits async; the hand-written loop must not race it).
- **Numeric tuning knobs live in code.** Frame data, gravity, speeds, `MAX_JUGGLES`, scaling constants, arena width — in `config/constants.ts` or the move tables, tuned there, not gated in this plan. The values in § Combat Specification are **tunable defaults**, not contract numbers.
- **Decide, don't branch.** The stack above is decided; no "A with fallback to B" ships in the plan body.
- **Thresholds follow measurement.** FPS, frame-time, sim-step, bundle-size numbers are "measure and record" at Phase 0/1, promoted to thresholds after.
- **No destructive recovery.** Phase-local revert is `git revert` of the phase's commits or restoring touched files. No `git reset --hard` against an unclean tree.

---

## Standing Gates

> Every phase runs these by default and does **not** restate them. A phase names only the
> standing gates it overrides, with a one-line reason.

| Gate | What it asserts | Default measurement |
|------|-----------------|---------------------|
| **Tests** | Tests covering the phase's edits pass; new behavior is covered; removed tests replaced same-commit or justified. Running test count meets the phase target (monotonic — see Tests). | `vitest run <changed/related test files>` → `M/N`. Scoped to edits. |
| **Static analysis** | No new lint/type errors on changed files; sim-firewall + Effect-boundary lint rules green. | `tsc --noEmit` + `eslint $(git diff --name-only <base>...HEAD)` → `0 new errors`. |
| **Performance** | Hot-path metric the phase declared protecting is within tolerance vs the declared baseline. | In-app frame-time meter + Chrome DevTools trace + sim-step micro-bench → delta line. |
| **Simplicity** | No new abstraction without a second consumer (or justification); no unused params/imports/exports. | Reviewer note + `rg` audit. |
| **Codebase quality** | No new dead code; line delta accounted for; module ownership honored. | `git diff --shortstat`. |
| **UX** | Any user/dev-visible surface shipped is verified in Chrome (screenshot/recording). | Phase-specific evidence. |

**Override syntax:** `Standing gates: UX (N/A — headless sim phase)`. Anything unlisted is in force.

> **Scope tests/lint to the phase's edits — never the whole repo.** The one deliberate
> repo-wide cold run is at Final Audit (Phase 12).

---

## Current Codebase Findings (Audit Baseline)

- [ ] Baseline tests: `vitest run` — empty; record `0/0`.
- [ ] Baseline build: `vite build` — record success + bundle size.
- [ ] Baseline static analysis: `tsc --noEmit` + `eslint .` — record `0 errors`.
- [ ] Runtime: `bun --version` (and `node --version`) meet Vite's minimum.
- [ ] **Chrome WebGPU probe:** target Chrome version `await renderer.init()` reports backend = WebGPU (a `GPUAdapter` was granted); record the Chrome version. Confirm the WebGL2 backend also initializes when WebGPU is forced off (`forceWebGL: true`).
- [ ] Pre-existing tree: `README.md` (stub) + `logs/` only; no source. No carried-forward cleanup (greenfield).

---

## Combat Specification (authoritative sim contract)

> **This section replaces the previously-referenced external `game-logic-prompt`.** All
> numbers are **tunable defaults** at 60 Hz integer frames; they live in `config/constants.ts`
> / the move tables and are tuned in code, not gated here. Internally consistent and chosen so
> every correctness criterion is unit-testable. **Every character shares this one kit** — it is a
> *mirror* match mechanically; the Tekken 5 roster are presentation skins over it (§ Character
> Roster & Identity). The sim has **no character identity**.

### §1 Units & coordinates
- Lane-locked 2.5D: fighters move on the X axis only; Y is height (jump/launch); Z fixed.
- Distances/velocities are sim world-units **per tick**. Timing is **integer frames**.
- Facing: each fighter has `facing ∈ {+1, −1}`; "forward" = toward the opponent.

### §2 Tunable constants (`config/constants.ts`)
```
TICK_HZ            = 60
GRAVITY           = 0.045    // units/tick², applied to airborne fighters each tick
WALK_SPEED        = 0.12     // units/tick
JUMP_VELOCITY     = 0.90     // initial upward units/tick
GROUND_Y          = 0.00
ARENA_HALF_WIDTH  = 6.0      // invisible walls at ±ARENA_HALF_WIDTH
MAX_JUGGLES       = 5
INPUT_BUFFER_LEN  = 12       // frames retained for command parsing
CHAIN_WINDOW      = 12       // frames after a chainable move to accept the next chain link
COMBO_RESET_ON    = ['recover', 'groundContact']
```

### §3 Fighter state machine (`stateMachine.ts`)
States (enum): `idle, walk, crouch, jumpRise, jumpFall, attackStartup, attackActive,
attackRecovery, hitstun, blockstun, launched, juggled, knockdown, wakeUp`.
- Transitions are switch-based with per-state timers/guards. Illegal transitions are rejected (e.g. you cannot start an attack during `hitstun`).
- `actionable` = state ∈ {idle, walk, crouch} (and grounded). Only actionable fighters may start moves or guard.
- Gravity applies in `jumpRise/jumpFall/launched/juggled`; ground clamp at `GROUND_Y` triggers the landing transition (→ recovery or knockdown).

### §4 Inputs & default keybinds
Buttons: **J K L I** (4 attack buttons). Directions: `← →` walk, `↑` jump, `↓` crouch.
**Guard = hold the away-from-opponent direction (`back`) while grounded + actionable** (no
dedicated key; the dummy sets the guard flag directly via toggle in Phase 10).

Default keymap (remappable + persisted, Phase 10):
| Action | Key | Token |
|--------|-----|-------|
| Move toward | `D` / `→` | `→` |
| Move away (also Guard when grounded) | `A` / `←` | `←` |
| Jump | `W` / `↑` | `↑` |
| Crouch | `S` / `↓` | `↓` |
| Light (Jab) | `J` | `J` |
| Mid (Body Blow) | `K` | `K` |
| Low (Low Kick) | `L` | `L` |
| Heavy (Uppercut) | `I` | `I` |

One `InputFrame` is latched **per fighter per tick** (sampled by the impure input layer, read
by the sim). The sim keeps a ring buffer of the last `INPUT_BUFFER_LEN` frames per fighter.

### §5 The eight moves (`moves/moveData.ts`)
`level ∈ {high, mid, low}` · `type ∈ {normal, command, launcher, chain}` · frames are integer.
`knockback`/`launch` are units/tick. Defaults:

| # | Move | Input | level | type | startup | active | recovery | damage | hitstun | blockstun | knockback | launch |
|---|------|-------|-------|------|---------|--------|----------|--------|---------|-----------|-----------|--------|
| 1 | Jab | `J` | high | normal | 3 | 2 | 7 | 4 | 9 | 5 | 0.05 | — |
| 2 | Body Blow | `K` | mid | normal | 6 | 3 | 12 | 8 | 14 | 8 | 0.12 | — |
| 3 | Low Kick | `L` | low | normal | 7 | 3 | 14 | 6 | 12 | 7 | 0.08 | — |
| 4 | Uppercut | `I` | mid | launcher | 7 | 4 | 22 | 10 | — | 10 | 0.05 | 0.40 |
| 5 | Crouch Sweep | `↓+L` | low | command | 9 | 4 | 24 | 9 | — | 9 | 0.10 | — (→ knockdown) |
| 6 | Rising Uppercut | `↑+I` | high | launcher (command) | 9 | 5 | 26 | 12 | — | 11 | 0.05 | 0.55 |
| 7 | Wall Smash | `→+K` | mid | command | 11 | 4 | 28 | 12 | 16 | 10 | 0.45 | — (airborne KB → wall path) |
| 8 | Chain Finisher | `J,J,K` | mid | chain | 8 | 3 | 18 | 9 | 16 | 9 | 0.20 | — |

Notes:
- **Active-frame gating:** a move's hitbox is live only during `active` frames (inclusive start, exclusive end). `hasHitThisMove` resets on move start; a move connects at most once.
- **Launchers (4, 6):** on connect, set target → `launched` with upward `launch` velocity (hitstun replaced by air time governed by §3 gravity).
- **Crouch Sweep (5):** on connect → target `knockdown` (§8).
- **Wall Smash (7):** strong horizontal `knockback`; if it sends an **airborne** target into a wall → wall path (§9). Wires the wall-bounce demonstration.
- **Chain Finisher (8):** reachable only inside the Jab chain (`J,J,K`) via `chainsInto`; standalone `K` is Body Blow (§6 parser priority).

### §6 Command parser priority (`commandParser.ts`)
Resolve each attack press against the ring buffer in this order (first match wins):
1. **Chain window:** if within `CHAIN_WINDOW` of a chain-eligible move and the link matches → chain move (e.g. `J` then `J` then `K` → Chain Finisher).
2. **Command (direction+button):** `↓+L` → Crouch Sweep, `↑+I` → Rising Uppercut, `→+K` → Wall Smash.
3. **Normal:** bare button → Jab/Body Blow/Low Kick/Uppercut.

### §7 Connect table (does the strike reach the hurtbox, by attacker level × target stance)
| level | vs **standing** | vs **crouching** |
|-------|-----------------|-------------------|
| high | connect | **whiff** (ducked) |
| mid | connect | connect |
| low | connect | connect |

Hit detection (`hitDetection.ts`): sphere hitbox vs hurtbox AABB in sim space, gated to active
frames; if the connect table says whiff for the target's current stance, no event is emitted.

### §8 Guard matrix (target is in guard = back held + grounded + actionable)
| level | **standing** guard | **crouching** guard |
|-------|--------------------|---------------------|
| high | **blocked** | (whiffs anyway, §7) |
| mid | **blocked** | **blocked** |
| low | **hit** | **blocked** |

Resolution per active-frame contact:
1. Connect check (§7). Whiff → nothing.
2. Else if target guarding and its stance blocks this level (table above) → **blockstun**, no damage (no chip, MVP).
3. Else → **hit**: apply scaled damage (§10), then `hitstun` / `launch` / `knockback` / `knockdown` per the move.

### §9 Juggles, knockdown, wakeup, wall & ground bounce
- **Juggle (`juggle.ts`):** while target is `launched/juggled`, an air hit re-juggles with lift `launch * max(0.2, 1 − 0.15*juggleCount)` and increments `juggleCount`. At `juggleCount` reaching `MAX_JUGGLES` (5), the next would-be juggle becomes **knockdown** instead. `juggleCount` resets only on ground/recover, never on air state re-entry.
- **Knockdown (`knockdown.ts`):** `knockdown` (fall + down-time) → `wakeUp` → `idle`, all timer-driven; fighter is **invulnerable** through `knockdown` and `wakeUp`.
- **Walls:** invisible walls at `±ARENA_HALF_WIDTH`. An **airborne** wall contact with `!wallBounceUsed` → **`wallSplat`** (brief pause, inward bounce, combo continues) and sets `wallBounceUsed`. A second airborne wall contact in the same combo → **knockdown**.
- **Ground bounce:** `groundBounceUsed` flag, default off for MVP moves; tracked so a future ground-bounce move can fire once per combo (same once-flag pattern as the wall). No current move sets it — the flag exists for the wall/ground symmetry the combo manager reads, with the wall path as its second consumer.

### §10 Combo & damage scaling (`comboManager.ts`, `damageScaling.ts`)
- Combo state: `hits`, accumulated `damage`, `juggleCount`, `wallBounceUsed`, `groundBounceUsed`.
- Per hit, **compute then increment**: `finalDamage = round(base * max(0.25, 1 − hits*0.08))` using the **pre-hit** `hits`, then `hits++`.
- Combo resets on opponent `recover` or `groundContact` (emits `comboReset`).

### §11 Sim events (read by render/audio, never written back)
`step()` returns `state` carrying a per-tick `events: SimEvent[]` list: `hit`, `block`,
`launch`, `juggle`, `wallSplat`, `knockdown`, `wakeUp`, `comboReset`. Render (Phase 9) and
audio consume this list from their read of sim state; nothing in `src/sim` consumes feedback.

---

## Character Roster & Identity (presentation contract)

> **Load-bearing firewall rule:** a character is a **presentation skin only**. The sim
> (`src/sim/**`) has **no character identity** — it simulates two fighters over the one shared
> kit (§ Combat Specification). Identity (name, proportions, colors, accent shape) lives in
> `config/characters.ts`, read by `render/` + `ui/` **only**. Swapping characters never changes a
> single sim output. This is what makes "all characters" cheap: **one kit, N skins.**
>
> **Scope (decided): the full Tekken 5 roster, geometric, all wired this build** as data-driven
> configs over the shared procedural rig (Phase 2) + shared 8-move sim. Tekken 5 is the
> **art-direction + staging** reference (camera, lighting, stage depth, weight, UI) — **not** an
> asset source; no models are imported (Non-Goals). **Private/educational build; Tekken is Bandai
> Namco IP — not for distribution.**

### `CharacterConfig` schema (`config/characters.ts`)
```
CharacterConfig = {
  id:           string   // 'jin', 'kazuya', ...
  displayName:  string   // 'Jin Kazama'
  build:        { height, mass, limbScale, torsoScale }   // scalars on the shared rig
  primaryColor: hex      // body / outfit dominant
  accentColor:  hex      // signature accent + joint-glow / energyParticle color
  accent?:      AccentShape   // optional primitive add-on (hair spikes, mask, wings, ears, hat)
  stance?:      PoseOffsets   // optional idle-stance signature
}
```
`build` + colors drive `fighterView.ts`; `accentColor` feeds `jointFlash` + `energyParticles`
(§11 render). **No field reaches the sim.**

### Roster (≈32 base Tekken 5 + optional DR), authored from public reference art
**Base Tekken 5:** Jin, Kazuya, Heihachi, Paul, Marshall Law, Nina, Anna, King, Marduk, Bryan,
Bruce, Lei, Hwoarang, Baek, Xiaoyu, Asuka, Julia, Christie, Eddy, Steve, Lee, Feng, Raven,
Yoshimitsu, Wang, Ganryu, Devil Jin, Jack-5, Kuma, Panda, Mokujin, Roger Jr.
**Optional Dark Resurrection:** Lili, Dragunov, Armor King (+ Jinpachi as an unlock).
Build archetypes for rig scalars: *lean-tall striker · tall-heavy bruiser · small-fast agile ·
huge grappler · animal/non-human*. Per-character color/build/accent **seed values** are authored
in `config/characters.ts` during execution (code, not gated here — same status as the combat
tunables), referenced from the gallery: https://tekken.fandom.com/wiki/Tekken_5/Gallery .

> Non-human entries (Kuma/Panda bear, Mokujin wood dummy, Roger Jr. kangaroo, Jack-5 robot) use
> the **same rig** with build scalars + accent shapes — stylized geometric, not modeled.

---

## Platform Layer (Effect boundary contract)

> Effect-TS lives **only** in `src/platform/**` (FR-01 boundary lint). Two files, both run at
> startup/menu boundaries — never per tick. This contract makes the boundary concrete so an
> executor does not leak `Effect<…>` into `loop/sim/render` or reach for the wrong API.

- **Packages.** `effect@4.0.0-beta.78` (pinned, exact) + `@effect/platform-browser` on the **same**
  beta for the localStorage layer. No other `@effect/*` runtime package is needed; add
  `@effect/vitest` only if the platform tests adopt it. Keep every `@effect/*` version-aligned.
- **Runtime boundary (load-bearing).** Build the runtime once at `main.ts` (a `ManagedRuntime`
  from the platform layer, or a top-level `Effect.runPromise`). Platform Effects are *run there*,
  at startup/menu only; `loop`/`sim`/`render`/`ui` receive **resolved plain values**, never an
  `Effect`. This is what "Effect only at boundaries" means operationally — and why persistence
  can never stall a frame (P10 failure protocol).
- **`platform/persistence.ts` — settings + keybinds (FR-20).** Use `@effect/platform-browser`
  `BrowserKeyValueStore.layerLocalStorage` as the live `KeyValueStore`; in tests provide the
  in-memory layer (`KeyValueStore.layerMemory`, imported from `effect/unstable/persistence/KeyValueStore`
  in the beta line) so the round-trip test (AT-22) is pure — no jsdom/localStorage shim. Model
  `Settings`/`Keybinds` as a `Schema.Class`; **load** via `Schema.decodeUnknownEffect` (corrupt or
  absent store → typed `SchemaError` → fall back to defaults, never throw); **save** via
  `Schema.encode`.
- **`platform/assets.ts` — audio preload (FR-20).** Howler load is event-based (`onload` /
  `onloaderror`), so wrap each load in `Effect.async`, **not** `Effect.tryPromise`. A failure is a
  typed `AssetLoadError` (`Data.TaggedError`); retry with
  `Effect.retry(load, Schedule.exponential('100 millis').pipe(Schedule.intersect(Schedule.recurs(3))))`.
  A missing asset surfaces the typed error after the retries — exercised by the P10 failure test.
- **TS discipline (skill rule).** No `any`, no `as`; the localStorage boundary is *decoded* via
  Schema, not asserted. Prefer `Effect.fn` for reusable platform business logic.

> *Note: `KeyValueStore` import paths differ between the v4-beta line (`effect/unstable/...` +*
> *`@effect/platform-browser`) and v3 (`@effect/platform`); the Assumption #5 escape hatch accounts*
> *for that divergence.*

---

## Functional Requirements

> FR-XX numbered. Each cites the Combat Spec section and the phase that delivers it.

- **FR-01** (G1, P0) Two ESLint `no-restricted-imports` rules: `src/sim` bans `three/howler/effect/DOM/timers`; `src/{sim,loop,render}` bans `effect`.
- **FR-02** (G1/G7, P0) `renderer.ts` constructs `WebGPURenderer`; `main.ts` `await`s `renderer.init()`, logs the active backend (`renderer.backend.isWebGPUBackend` vs `renderer.backend.isWebGLBackend`), and only then starts the loop. WebGPU is the default backend; the WebGL2 fallback is reachable via the `forceWebGL: true` constructor option.
- **FR-03** (G2, P1) `sim/serialize.ts` snapshots `GameState` and produces a **stable hash over the typed-array/byte buffer** (not a JSON string — avoids float→string format drift). **Canonicalize before hashing:** map `-0 → 0` and keep `NaN` out of serialized fields (or canonicalize it), because raw byte hashing *is* sensitive to `-0`/`NaN` bit patterns — otherwise two semantically-equal states can hash unequal (breaks AT-05). Same-build replay (AT-02/AT-04) stays bit-deterministic either way.
- **FR-04** (G2, P1) Replay: a recorded `InputFrame` log replayed twice → identical hash.
- **FR-05** (G3, P1) `loop/gameLoop.ts` fixed-timestep accumulator with **dt clamp ≤ 250 ms** and a **max-substeps cap**; render gets interpolation `alpha`. Sim ticks/wall-second invariant under render throttle (±1).
- **FR-06** (G5, P2) `rig.ts` builds a shared-material `Object3D` humanoid (head/torso/pelvis/arms/forearms/hands/thighs/shins/feet + joint spheres). No per-limb material clones.
- **FR-07** (G5, P2) `fighterView.ts` maps a `FighterState` pose to a rig **and applies a `CharacterConfig` (build scalars + colors) to the shared rig**; idle/neutral stance; the two selected characters spawn at opposing X, both framed.
- **FR-08** (G4, P3) Input layer latches one `InputFrame`/fighter/tick; `inputBuffer.ts` ring buffer (`INPUT_BUFFER_LEN`).
- **FR-09** (G4, P3) Movement FSM (§3): idle/walk/crouch/jump with sim gravity + ground clamp; legal transitions pass, illegal rejected.
- **FR-10** (G4, P4) All eight moves load from `moveData.ts` with the §5 frame data; hitboxes live only on active frames; `hasHitThisMove` resets per move.
- **FR-11** (G4, P4) `commandParser.ts` resolves §6 priority: chain (`J,J,K`) > command (`↓+L`,`↑+I`,`→+K`) > normal.
- **FR-12** (G4, P4) `hitDetection.ts` sphere-vs-AABB on active frames with the §7 connect table; emits `hit` events; applies damage + hitstun.
- **FR-13** (G4, P5) Guard flag derivation (§8) + full guard matrix; blockstun on block, hitstun + knockback on hit.
- **FR-14** (G4, P6) Launchers set `launched`; sim gravity; `juggle.ts` lift decay + `MAX_JUGGLES`; air hits continue the juggle; cap → knockdown.
- **FR-15** (G4, P7) Sweep/Low trip → knockdown; `knockdown.ts` lifecycle + invulnerability; walls at `±ARENA_HALF_WIDTH`; `wallBounce.ts` once-per-combo `wallSplat` else knockdown; `groundBounceUsed` tracked; Wall Smash wired to the wall path.
- **FR-16** (G4, P8) `comboManager.ts` + `damageScaling.ts` per §10 (pre-hit compute, then increment); reset on recover/ground; surface to `uiStore`.
- **FR-17** (G5, P9) Render/audio feedback (jointFlash, **pooled** hitSpark/energyParticles, cameraShake, Howler impacts + autoplay unlock) consume `state.events` only; firewall `rg` clean.
- **FR-18** (G6, P10) HTML/CSS HUD via `zustand/vanilla`: health bars, timer placeholder, combo counter, damage, state debug, input display, optional FPS.
- **FR-19** (G6, P10) `training/dummy.ts` toggles (block / auto-recover / jump-state / reset) feed scripted `InputFrame`s into the sim.
- **FR-20** (G6, P10) `platform/persistence.ts` (Effect) load/save settings + keybinds to `localStorage`; `platform/assets.ts` (Effect) preloads audio with typed failure/retry; Effect only under `src/platform` (see § Platform Layer for the package, runtime-boundary, and Schema contract).
- **FR-21** (G7, P11) `arena.ts` Tekken-5 staging (enclosed walled arena, reflective/graded floor, deep hazy layered background, per-stage key+rim lighting + color grade) at low asset weight; side-3/4 tracking camera (dolly in/out + lateral track).
- **FR-22** (G7, P12) Desktop Chrome ≥ 60 FPS under full combat on the documented host; `step()` within the Phase 1 budget; Android-Chrome FPS via WebGL2 backend recorded (target 30–60).
- **FR-23** (G2/G4/G7, P12) Final Audit: one repo-wide cold run (build/tsc/eslint/full Vitest/replay) + Playwright Chrome smoke (real Chrome + WebGPU flags — see Phase 12).
- **FR-24** (G9, P2) `config/characters.ts` defines the `CharacterConfig` schema (§ Character Roster); `fighterView.ts` is config-driven — build scalars + colors applied to the shared rig. Identity never enters the sim (firewall).
- **FR-25** (G9, P11) The full Tekken-5 roster is authored as `CharacterConfig`s; `accentColor` maps to `jointFlash` + `energyParticles`; **every** character renders from config (render-sweep verified).
- **FR-26** (G6, P10) `ui/characterSelect.ts` portrait-grid select for **player + dummy** identity; confirm spawns the chosen pair; swap supported; selection carries no data into the sim.
- **FR-27** (G7, P11) Tekken-5 staging: side-3/4 **tracking camera** (dolly in/out + lateral track), key+rim lighting + per-stage color grade, enclosed walled arena + hazy layered background, chrome lifebars + portrait-grid select UI.

---

## Tests (per-phase, monotonic)

> Per the evidence rule, test work items are separate checklist items inside each phase.
> **Running total never decreases.** A refactor that invalidates a test replaces it same-commit.
> Counts below are targets (`≈`), tuned during execution; the **monotonic floor** is the contract.

| After phase | New unit tests — sim/replay/pure (≈) | Running total (floor) | Adds |
|-------------|--------------------------|------------------------|------|
| P0 | 1 | 1 | trivial smoke |
| P1 | 3 | 4 | serialize+hash, replay determinism, tick-parity |
| P2 | 1 | 5 | pose-mapping pure helper |
| P3 | 6 | 11 | FSM legal/illegal, gravity/ground, input-buffer latch |
| P4 | 12 | 23 | moves load, active-frame gating, frame data, §7 connect, §6 parser, chain |
| P5 | 10 | 33 | §8 guard matrix (3 levels × 2 stances × block), blockstun, knockback |
| P6 | 8 | 41 | launch, lift decay, `MAX_JUGGLES` cap → knockdown, air-continuation |
| P7 | 8 | 49 | sweep→knockdown, knockdown lifecycle, once-per-combo wall, 2nd-contact→knockdown |
| P8 | 7 | 56 | scaling across long combo, pre-hit increment order, reset conditions |
| P9 | 3 | 59 | sim `events[]` emission (sim-side only) |
| P10 | 6 | 65 | persistence round-trip, dummy-toggle, asset preload typed-failure/retry, character-select logic |
| P11 | 1 | 66 | character config: all roster entries load + valid |
| P12 | +1 e2e | 66 sim + 1 e2e | Playwright Chrome smoke |

**Coverage floor:** sim hot paths the perf/correctness criteria measure (frame data, hit
detection, FSM, combo/juggle math) carry **100% branch coverage**. Render/UI/audio follow risk.

---

## Error Correction

> Greenfield, so no carried-forward defects. These are the corrections this revision applies to
> the prior draft, surfaced (not silently patched), each owned by a phase.

- **EC-1 — `rng.ts` had no consumer.** A deterministic fighter with a scripted dummy has nothing random in MVP; an empty seeded-RNG module violates the Simplicity gate. **Fix:** `sim/rng.ts` is **deferred** — created only when a sim consumer appears. The Determinism constraint keeps "seeded RNG only" as the rule *if/when* randomness is introduced. (Owner: Repo Structure / P1.)
- **EC-2 — Phantom external spec.** Phases 3–8 referenced a non-existent `game-logic-prompt`. **Fix:** the full spec is inlined (§ Combat Specification); phases cite `§N` of this document. (Owner: all sim phases.)
- **EC-3 — WebGPU async-init race.** A hand-written loop can render before `WebGPURenderer` finishes async init. **Fix:** FR-02 / Hard Constraint — `await renderer.init()` before the loop. (Owner: P0/P1.)
- **EC-4 — Accumulator spiral-of-death.** Fixed-timestep with no dt clamp death-spirals after a backgrounded tab. **Fix:** FR-05 dt clamp + substep cap. (Owner: P1.)
- **EC-5 — Playwright can't reach WebGPU by default.** Headless bundled Chromium lacks WebGPU without flags. **Fix:** Phase 12 uses `channel: 'chrome'` + WebGPU launch flags and records the backend actually exercised. (Owner: P12.)
- **EC-6 — `groundBounceUsed` has no MVP consumer.** No current move sets it (§9), which is in tension with EC-1 (defer the unused `rng.ts`) and the Simplicity gate. **Ruling: keep it.** Unlike `rng.ts` — a whole module/seam — this is a single bookkeeping boolean in combo state that mirrors `wallBounceUsed` and rides the *same* once-per-combo reset path (the wall path is the live consumer of that pattern). It is tracked-only, sets nothing, and is asserted by no MVP test; adding a move that sets it is a deliberate, justified step under the Simplicity gate. (Owner: P7/P8.)
- **EC-7 — Wrong Three.js fallback option name.** The prior draft used `forceWebGL2`, which is not a `WebGPURenderer` option. **Fix:** the real constructor option is `forceWebGL: true` (forces the WebGL2-based `WebGLBackend`); WebGPU is the no-flag default; active backend is read via `renderer.backend.isWebGPUBackend` / `isWebGLBackend`. (Owner: P0 — Stack/FR-02/Audit Baseline updated.)
- **EC-8 — Zodiac concept replaced by the Tekken-5 roster.** The prior draft themed fighters as zodiac signs (Leo/Aries + sign stubs, others data-only). **Fix (decided):** the full Tekken 5 roster — geometric skins over the one shared kit — replaces the zodiac theme; § Character Roster & Identity is the new identity contract; Phase 11 retargets to Tekken-5 staging; `zodiacParticles`→`energyParticles`, `zodiacConfigs`→`config/characters.ts`. Because the sim stays **character-agnostic**, the change touches `config/`, `render/`, `ui/` only — **no sim/combat change** (the sim/combat correctness tests are untouched; the two added tests are render/UI/platform-side). (Owner: P2/P10/P11.)

---

## Phases

### Phase 0 — Baseline & skeleton

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** An empty Vite + TS + Three project builds, type-checks, lints, runs Vitest, and
clears the screen with `WebGPURenderer` (async init awaited, backend logged) in Chrome. Both
firewall lint rules pass on an empty `src`.

**Entry criteria:** Audit Baseline filled with real numbers; toolchain installed.

**Work:**
- [ ] Scaffold Vite + TypeScript (strict); `index.html` with a full-window canvas.
- [ ] Add Three.js; `renderer.ts` constructs `WebGPURenderer`; `main.ts` `await renderer.init()`, logs the active backend, then clears to a flat color (FR-02, EC-3).
- [ ] Confirm `forceWebGL: true` reaches the WebGL2 backend (logged).
- [ ] Add Vitest; one trivial passing test (test floor: 1).
- [ ] Add ESLint with the two `no-restricted-imports` rules (FR-01).
- [ ] Create the directory skeleton (no logic; `rng.ts` intentionally absent — EC-1).
- [ ] Run the one deliberate repo-wide cold sweep (build, tsc, eslint, vitest); record numbers and which backend Chrome selected.

**Exit gates:** Standing gates: Performance (N/A — capturing baseline), UX (N/A — blank screen).

**Evidence:** build/tsc/eslint/vitest output; bundle size; "backend = WebGPU" log line + forced-WebGL2 log line.

**Failure protocol:** Fix the command and retry. No revert.

---

### Phase 1 — Deterministic loop + render proof

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** A single box, owned by a trivial pure sim, advances at fixed 60 Hz regardless of
Chrome's render rate, is drawn with interpolation, framed by the side-view camera; a replayed
input log reproduces an identical state hash. The accumulator is dt-clamped.

**Entry criteria:** Phase 0 gates green.

**Work:**
- [ ] Minimal `GameState` (one entity: position + velocity) and `step()` in `sim/`.
- [ ] `sim/serialize.ts` snapshot + **stable hash over the byte buffer** (FR-03).
- [ ] Fixed-timestep accumulator loop in `loop/gameLoop.ts` with **dt clamp ≤ 250 ms + substep cap** (FR-05, EC-4); render gets `alpha`.
- [ ] `render/interpolate.ts`; draw the box lerped between prev/next snapshots.
- [ ] `camera.ts` side-view framing.
- [ ] Test: serialize→hash stability (1).
- [ ] Replay test: two replays of one input log → identical hash (FR-04; `tests/replay`) (1).
- [ ] Test: tick-count parity at full vs ~20 FPS throttled render within ±1 tick/s (FR-05) (1).
- [ ] **Measure & record:** empty-scene and single-box frame time, and `step()` cost (the baselines later phases protect).

**Exit gates (overrides only):**
- Tick-count parity across throttle within ±1 tick/sec (AT-04).
- Replay hash equality (AT-02); byte-buffer hash equal for equal states, `-0`/`NaN` canonicalized (AT-05).
- Accumulator dt-clamp + substep cap survive a simulated long stall — no death-spiral (AT-06).
- Render interpolation `alpha` drives draw between prev/next snapshots; no sim mutation in render (AT-07).
- Running test floor: **4**.

**Evidence:** replay test output with both hashes; tick-count parity log; frame-time + `step()` baselines.

**Failure protocol:** If sim speed tracks render rate, wall-time is leaking into the sim — suspect the clock crossing the firewall, not the tick math. If the sim hitches after a backgrounded tab, the dt clamp/substep cap is missing.

---

### Phase 2 — Fighter rig + procedural pose

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** Two geometric humanoid rigs (two selected roster characters) stand in the arena, framed by the
camera, glowing joints, posable from a `FighterState`.

**Entry criteria:** Phase 1 gates green.

**Work:**
- [ ] `rig.ts`: `Object3D` graph for head/torso/pelvis/arms/forearms/hands/thighs/shins/feet + joint spheres, named refs (FR-06).
- [ ] Shared materials (one body, one joint/glow) — no per-limb material clones.
- [ ] `config/characters.ts`: `CharacterConfig` schema + ≥2 seed configs (full roster authored in Phase 11) (FR-24).
- [ ] `fighterView.ts`: apply a `FighterState` pose to a rig **and a `CharacterConfig` (build scalars + colors) to the shared rig**; idle/neutral stance (FR-07, FR-24).
- [ ] Spawn two characters (config-driven) at opposing X; camera frames both (note dynamic framing as they approach the walls).
- [ ] Confirm joint glow renders under the WebGPU backend (and the WebGL2 backend initializes).
- [ ] Test: pure pose-mapping helper (FighterState → joint transforms) (1).

**Exit gates (overrides only):** UX — screenshot of both rigs, glowing joints, both framed. Running test floor: **5**.

**Evidence:** screenshot; two-rig draw-call count; frame-time delta vs Phase 1.

**Failure protocol:** Draw-call blowup → suspect per-part material instances; materials must be shared.

---

### Phase 3 — Input + movement state machine

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** The player walks, crouches, jumps via keyboard, driven by the sim FSM
(idle/walk/crouch/jump) with gravity and ground; input is captured into the ring buffer.
*(Sim spec: §3, §4.)*

**Entry criteria:** Phase 2 gates green.

**Work:**
- [ ] `input/inputTypes.ts` + a Chrome keyboard sampler (impure layer) latching one `InputFrame` per fighter per tick (FR-08).
- [ ] `input/inputBuffer.ts` ring buffer (`INPUT_BUFFER_LEN`) in the sim.
- [ ] `stateMachine.ts`: state enum (§3) + switch-based transition guards/timers.
- [ ] Movement in `fighterState.ts`: walk speeds, crouch, jump arc with sim gravity + ground clamp (FR-09).
- [ ] `commandParser.ts` scaffold reading the buffer (no moves yet).
- [ ] Drive walk/crouch/jump rig poses from sim state.
- [ ] Test: FSM transitions — legal accepted + illegal rejected (≈3) (FR-09).
- [ ] Test: gravity/ground-clamp jump arc (≈2).
- [ ] Test: input-buffer single-latch-per-tick (≈1) (FR-08).

**Exit gates (overrides only):** FSM tests green; UX — movement recording in Chrome. Running test floor: **11**.

**Evidence:** `vitest run tests/sim/fsm`; recording; frame-time delta.

**Failure protocol:** Dropped buttons → suspect sampling between ticks rather than one latched input frame per tick.

---

### Phase 4 — Move system + strikes + hit detection

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** The eight moves load as data; attacks run startup/active/recovery with correct
frame data; hitboxes are live only on active frames; a connecting strike applies damage +
hitstun (no knockback/launch yet). *(Sim spec: §5, §6, §7.)*

**Entry criteria:** Phase 3 gates green.

**Work:**
- [ ] `moves/moveTypes.ts` + `moves/moveData.ts` with all eight moves per §5 (frame data, level, type, hitbox, knockback, scaling, `chainsInto`) (FR-10).
- [ ] Attack states in the FSM (startup→active→recovery) with per-move frame counts; `hasHitThisMove` reset on move start.
- [ ] `commandParser.ts` §6 priority: chain (`J,J,K`), commands (`↓+L`,`↑+I`,`→+K`), normals (FR-11).
- [ ] `combat/hitDetection.ts`: sphere hitbox vs hurtbox AABB in sim space, gated to active frames; §7 connect table; emit `hit` events (FR-12).
- [ ] Apply damage + `hitstun` on connect; drop dummy health.
- [ ] Debug hitbox/hurtbox overlay (toggle) in render.
- [ ] Test: active-frame gating inclusive/exclusive bounds (≈3).
- [ ] Test: each move's frame data loads (≈2) (AT-08).
- [ ] Test: §7 level-vs-stance connect/whiff (≈3) (AT-09).
- [ ] Test: §6 command disambiguation incl. `J,J,K` chain (≈4) (AT-10).

**Exit gates (overrides only):** Move + command tests green (AT-08, AT-09, AT-10). Running test floor: **23**.

**Evidence:** `vitest run tests/sim/moves tests/sim/commands`; recording of each move connecting; overlay screenshot.

**Failure protocol:** A hit on startup/recovery is an off-by-one in the active window — check inclusive/exclusive bounds, not the hitbox.

---

### Phase 5 — Hit reactions: hitstun, block, blockstun, knockback

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** The dummy reacts correctly — hitstun, knockback, blockstun on guard, with the
§7 connect / §8 guard matrix correct. *(Sim spec: §7, §8.)*

**Entry criteria:** Phase 4 gates green.

**Work:**
- [ ] `guard` derived flag (§8: back held + grounded + actionable) and standing/crouching stance.
- [ ] §8 guard resolution table; `blockstun` on block, `hitstun` + `knockback` on hit (FR-13).
- [ ] Drive hitstun/blockstun/knockback poses + displacement in render.
- [ ] Dummy "block on/off" input path (full toggle UI in Phase 10).
- [ ] Test: full §8 guard matrix — high/mid/low × standing/crouching × guarding/not (≈6) (AT-11).
- [ ] Test: blockstun applied on block, no damage (≈2).
- [ ] Test: knockback magnitude/direction on hit (≈2) (AT-12).

**Exit gates (overrides only):** Block-matrix tests green (AT-11, AT-12). Running test floor: **33**.

**Evidence:** `vitest run tests/sim/block`; recording of blocked vs unblocked exchanges.

**Failure protocol:** Lows blockable standing → inverted guard-height comparison; verify hit-level enum ordering vs the guard check against §8.

---

### Phase 6 — Launchers + juggles

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** Uppercut / Rising Uppercut launch into `launched`; sim gravity pulls down; air
hits re-juggle with decaying lift; the 6th would-be juggle becomes knockdown. *(Sim spec: §9.)*

**Entry criteria:** Phase 5 gates green.

**Work:**
- [ ] `launched`/`juggled` states; apply `launch` velocity on launcher connect (FR-14).
- [ ] Sim gravity on airborne fighters; land → recovery/knockdown.
- [ ] `combat/juggle.ts`: lift decay `max(0.2, 1 − 0.15*juggleCount)`, `MAX_JUGGLES = 5`, then knockdown.
- [ ] Air hits keep the opponent aloft (continuation).
- [ ] Drive launched/juggled airborne poses in render.
- [ ] Test: launch velocity applied on connect (≈1) (AT-13).
- [ ] Test: lift decay per juggle (≈2).
- [ ] Test: `MAX_JUGGLES` cap → 6th becomes knockdown (≈3) (AT-14).
- [ ] Test: air-continuation keeps target aloft + `juggleCount` resets only on ground/recover (≈2) (AT-15).

**Exit gates (overrides only):** Juggle tests green (AT-13, AT-14, AT-15). Running test floor: **41**.

**Evidence:** `vitest run tests/sim/juggle`; recording of launch→juggle→knockdown.

**Failure protocol:** Endless juggles → the counter resets mid-air; suspect the reset firing on state re-entry rather than on ground/recover.

---

### Phase 7 — Sweep, knockdown, wakeup, wall + ground bounce

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** Sweeps knock down; knockdown→wakeUp→idle runs on timers; sim arena walls produce
a once-per-combo `wallSplat` (else knockdown); ground-bounce flag tracked. *(Sim spec: §9.)*

**Entry criteria:** Phase 6 gates green.

**Work:**
- [ ] Crouch Sweep (`↓+L`) and Low Kick trip → knockdown (FR-15).
- [ ] `combat/knockdown.ts`: fall → down-time → `wakeUp` → `idle` timers + poses; invulnerable through knockdown/wakeUp.
- [ ] Invisible walls at `±ARENA_HALF_WIDTH` in the sim.
- [ ] `combat/wallBounce.ts`: airborne wall contact, `!wallBounceUsed` → `wallSplat` (pause, inward bounce, continue) else knockdown.
- [ ] Track `groundBounceUsed`; wire Wall Smash (`→+K`) to the wall-bounce path (FR-15).
- [ ] Drive wallSplat/groundBounce/knockdown/wakeUp poses in render.
- [ ] Test: sweep/low → knockdown (≈2) (AT-16).
- [ ] Test: knockdown lifecycle + invulnerability window (≈2).
- [ ] Test: once-per-combo wall bounce, 2nd airborne contact → knockdown (≈4) (AT-17).

**Exit gates (overrides only):** Wall-bounce + knockdown tests green (AT-16, AT-17). Running test floor: **49**.

**Evidence:** `vitest run tests/sim/wallbounce`; recording of splat→continuation, then second contact→knockdown.

**Failure protocol:** Wall bounce fires twice → `wallBounceUsed` resets with the combo too eagerly; suspect the reset trigger, not the geometry.

---

### Phase 8 — Combo manager + damage scaling

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** The combo manager ties prior phases' hit events into live combo state (hits,
damage, juggle/wall/ground flags) with damage scaling, resetting on recover/ground.
*(Sim spec: §10.)*

**Entry criteria:** Phase 7 gates green.

**Work:**
- [ ] `combat/comboManager.ts`: increment hits, accumulate damage, track juggle/wall/ground usage (FR-16).
- [ ] `combat/damageScaling.ts`: `finalDamage = round(base * max(0.25, 1 − hits*0.08))` (pre-hit count, then increment).
- [ ] Reset combo on opponent recover or ground contact; emit `comboReset`.
- [ ] Surface combo hits + scaled damage to `uiStore`.
- [ ] Test: scaling across a long combo, exact values (≈3) (AT-18).
- [ ] Test: pre-hit-increment order (compute, then increment) (≈2).
- [ ] Test: reset on recover + on ground contact (≈2).

**Exit gates (overrides only):** Combo tests green (AT-18). Running test floor: **56**.

**Evidence:** `vitest run tests/sim/combo`; recording of a scaling combo that resets correctly.

**Failure protocol:** Damage doesn't taper → scaling reads a stale count; confirm increment order (compute, then increment).

---

### Phase 9 — Visual + audio feedback

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** Hits read and feel impactful in Chrome: limb flash, energy particles, sparks,
camera shake, impact sound — all driven by the sim's `SimEvent` list (§11), none feeding back
into the sim.

**Entry criteria:** Phase 8 gates green.

**Work:**
- [ ] `effects/jointFlash.ts` (flash struck limb/joint in the character's accent color).
- [ ] `effects/hitSpark.ts` + `effects/energyParticles.ts` — **pooled**, low cost.
- [ ] `effects/cameraShake.ts` — decaying offset, render-only.
- [ ] Commit placeholder/licensed sfx + ambience under `public/audio/` — real files so impacts (and the Phase 10 Effect preload + its failure/retry test) have something to load.
- [ ] `audio/audioManager.ts` (Howler): impact sounds, ambience, Chrome autoplay unlock on first input.
- [ ] Wire all to consume `state.events` (§11) from the render layer's read; never from the sim (FR-17).
- [ ] Firewall `rg` audit: no effects/audio under `src/sim`.
- [ ] Test: sim emits the correct `events[]` per tick for hit/block/launch/etc. (sim-side only) (≈3) (AT-19).

**Exit gates (overrides only):** UX — recording with effects + sound in Chrome; firewall `rg` clean; sim `events[]` emission correct, no sim feedback consumption (AT-19). Running test floor: **59**.

**Evidence:** recording; particle-on frame-time delta vs Phase 1; `rg` result.

**Failure protocol:** Frame-time spikes on heavy hits → particles not pooled; suspect per-hit geometry allocation, not the shader.

---

### Phase 10 — HUD + dummy training + persistence

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** A complete training HUD overlays the canvas; dummy toggles work; settings/keybinds
persist across reload via the Effect persistence boundary; assets preload via the Effect asset
boundary.

**Entry criteria:** Phase 9 gates green.

**Work:**
- [ ] HTML/CSS HUD (`ui/`): player + dummy health bars, round-timer placeholder, combo counter, damage number, current-state debug text, input display, optional FPS (FR-18).
- [ ] `state/uiStore.ts` (`zustand/vanilla`); subscribe HUD widgets.
- [ ] `training/dummy.ts`: block on/off, auto-recover on/off, jump-state on/off, reset position — scripted inputs into the sim (FR-19).
- [ ] `ui/characterSelect.ts` (+ css): Tekken-style portrait-grid select for **player + dummy**; confirm spawns the chosen pair; swap; last-selected pair persisted via `platform/persistence` (FR-26).
- [ ] `platform/persistence.ts` (Effect): load/save settings + keybinds to `localStorage` (FR-20).
- [ ] `platform/assets.ts` (Effect): preload audio with typed failures/retries (FR-20).
- [ ] Boundary `rg` audit: Effect only under `src/platform`.
- [ ] Verify a changed keybind survives reload.
- [ ] Test: persistence round-trip (settings/keybinds serialize → reload → equal) (≈2).
- [ ] Test: dummy-toggle input path produces the expected `InputFrame`s (≈2).
- [ ] Test: character-select logic — pick/confirm/swap yields the correct two-character render request; selection passes no data into the sim (≈1) (AT-30).
- [ ] Test: asset preload typed-failure + retry — a missing asset surfaces `AssetLoadError` after N retries; the in-memory `KeyValueStore` test layer keeps it pure (≈1) (FR-20).

**Exit gates (overrides only):** UX — character-select + toggle walkthrough + reload-persistence recording (AT-21, AT-22, AT-30); Effect-boundary `rg` clean (AT-03). Running test floor: **65**.

**Evidence:** recording; persistence reload clip; `rg "effect" src/sim src/loop src/render` → no matches.

**Failure protocol:** Persistence stalls a frame → an Effect program is running in the loop; confirm load/save runs at boundaries (startup/menu), not per tick.

---

### Phase 11 — Tekken-5 art pass: staging + full-roster identities

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** The scene reads as **Tekken-5 staging** at low asset weight, and **every** roster
character renders from its `CharacterConfig`. (Final-audit work moves to Phase 12 so a polish miss
can't pollute the audit gate.)

**Entry criteria:** Phase 10 gates green.

**Work:**
- [ ] `arena.ts`: Tekken-5-style **enclosed walled arena** — reflective/graded floor, deep hazy layered background, per-stage **key+rim lighting + color grade** (e.g. Moonlit-Wilderness cool blue) — low asset weight (FR-21, FR-27).
- [ ] Side-3/4 **tracking camera**: dolly in/out as fighters close/separate, lateral track, slight elevation (Tekken framing) (FR-27).
- [ ] Chrome lifebars + portrait-grid select styling pass (Tekken UI look) (FR-27).
- [ ] `config/characters.ts`: author identity configs (build scalars, primary/accent colors, optional accent shape) for the **full roster**; map `accentColor` → `jointFlash` + `energyParticles` (FR-25).
- [ ] Verify **every** roster character renders from config (select-sweep / spawn-sweep contact sheet).
- [ ] Test: all roster `CharacterConfig`s load + map to valid rig params (parametrized over the roster) (≈1) (AT-29).

**Exit gates (overrides only):** UX — Tekken-reference-vs-build screenshot pair + a full-roster render-sweep contact sheet (AT-29, AT-31). Running test floor: **66**.

**Evidence:** screenshot pair; full-roster render-sweep contact sheet; frame-time delta vs Phase 1 with staging on.

**Failure protocol:** Staging tanks frame time → suspect reflective-floor / background overdraw or per-stage lighting; cut sample count, background layers, or resolution before touching the rig.

---

### Phase 12 — Final Audit

**Checklist rule: mark `[x]` only when implementation is complete and validated with tests/evidence.**

**Objective:** Performance targets confirmed in Chrome; every Acceptance Target evidenced; one
clean repo-wide cold run + a real-Chrome Playwright smoke.

**Entry criteria:** Phase 11 gates green.

**Work:**
- [ ] Final perf pass in Chrome: desktop ≥ 60 FPS under full combat on the documented host (DevTools Performance trace); record Android-Chrome FPS via the WebGL2 backend (target 30–60) (FR-22).
- [ ] **Playwright smoke (FR-23, EC-5):** `playwright.config.ts` uses `channel: 'chrome'` (real Chrome, not bundled Chromium) with WebGPU launch flags (`--enable-unsafe-webgpu`, and on Linux `--use-angle=vulkan --enable-features=Vulkan --disable-vulkan-surface`; run headed or under `xvfb` if the CI GPU is unavailable). The smoke asserts a `GPUAdapter` was granted and **records the backend actually exercised**; if the harness GPU can't provide WebGPU, the smoke runs on the WebGL2 backend and that fact is logged (the gate is "smoke passes on the path the harness can reach", not "WebGPU in CI").
- [ ] One repo-wide cold run: build, tsc, eslint, full Vitest, replay.
- [ ] Walk every Acceptance Target; confirm current evidence.
- [ ] Write `artifacts/completion-summary.md`: shipped / deferred / residual risks.

**Exit gates (overrides only):**
- Desktop-Chrome FPS ≥ 60 under full-combat scene on the documented host (AT-23).
- Sim `step()` within the per-phase budget (AT-24).
- Every Acceptance Target `[x]` with evidence.
- Repo-wide cold run + Playwright smoke green (or each known gap documented).
- Running test floor: **66 sim + 1 e2e**.

**Evidence:** full cold-run output; Playwright result + backend-exercised line; final frame-time + sim-step numbers vs Phase 1, archived under `benchmarks/mirror-match-phase12-*.txt`; reference-vs-build screenshot pair; completion summary path.

**Failure protocol:** Missing 60 FPS → profile in Chrome DevTools before optimizing; suspect order is draw calls (shared materials?) → particle pooling → rig pose updates, not the backend choice. Playwright can't get WebGPU → it's the launch flags / channel, not the app.

---

## Acceptance Targets

> 20–36 testable assertions. Each cites its evidence source. Checked at the owning phase and
> re-confirmed at Final Audit.

**Architecture integrity**
- **AT-01** `rg "three|howler|effect|document|window" src/sim` → no matches; sim-firewall lint green. *(P0/ongoing)*
- **AT-02** Recorded input log replayed twice → identical state hash. *(`vitest run tests/replay`, P1)*
- **AT-03** `rg "effect" src/sim src/loop src/render` → no matches. *(P10)*
- **AT-04** Sim ticks/wall-second invariant under render throttle (full vs ~20 FPS) within ±1. *(instrumented log, P1)*
- **AT-05** State hash is computed over the byte buffer (no JSON-string float drift), with `-0` canonicalized to `0` and no `NaN` in serialized fields; two snapshots of equal state hash-equal. *(P1)*

**Loop & determinism**
- **AT-06** Accumulator clamps dt (≤ 250 ms) and caps substeps; a simulated long stall does not death-spiral. *(P1 test/log)*
- **AT-07** Render interpolation `alpha` drives draw between prev/next snapshots; no sim mutation in render. *(P1 review + recording)*

**Combat correctness (headless)**
- **AT-08** All eight moves load with §5 frame data; hits land only on active frames (inclusive/exclusive bounds correct). *(`vitest run tests/sim/moves`, P4)*
- **AT-09** §7 connect table correct (high whiffs crouch; mid/low connect both stances). *(P4)*
- **AT-10** §6 command disambiguation: `↓+L`→Crouch Sweep, `↑+I`→Rising Uppercut, `→+K`→Wall Smash, `J,J,K`→Chain Finisher. *(`vitest run tests/sim/commands`, P4)*
- **AT-11** §8 guard matrix correct across high/mid/low × standing/crouching. *(`vitest run tests/sim/block`, P5)*
- **AT-12** Blockstun on block (no damage); hitstun + knockback on hit, correct magnitude/direction. *(P5)*
- **AT-13** Launchers set `launched` with the §5 `launch` velocity; sim gravity returns them to ground. *(P6)*
- **AT-14** Juggle lift decays `max(0.2, 1 − 0.15*juggleCount)`; `MAX_JUGGLES = 5`; 6th would-be juggle → knockdown. *(`vitest run tests/sim/juggle`, P6)*
- **AT-15** `juggleCount` resets only on ground/recover, never on air state re-entry. *(P6)*
- **AT-16** Crouch Sweep / Low trip → knockdown; knockdown→wakeUp→idle timer lifecycle with invulnerability. *(P7)*
- **AT-17** Wall bounce fires once per combo (`wallSplat` + continuation); a second airborne wall contact → knockdown. *(`vitest run tests/sim/wallbounce`, P7)*
- **AT-18** Combo scaling matches `max(0.25, 1 − hits*0.08)` with pre-hit count, then increment; resets on recover/ground. *(`vitest run tests/sim/combo`, P8)*
- **AT-19** `step()` emits the correct §11 `events[]` per tick; no sim code consumes feedback. *(P9)*

**Observable behavior (Chrome)**
- **AT-20** Player walks/crouches/jumps and executes all eight moves; the dummy reacts with hitstun/blockstun/launch/knockdown appropriately. *(recording, P9/P10)*
- **AT-21** HUD shows live health, combo count, damage, state, input; dummy toggles (block, auto-recover, jump-state, reset) all work. *(recording + walkthrough, P10)*
- **AT-22** Settings/keybinds persist across reload. *(reload recording, P10)*

**Performance**
- **AT-23** Desktop Chrome ≥ 60 FPS under a full-combat scene on the documented host. *(in-app meter + DevTools trace, P12)*
- **AT-24** Sim `step()` within the Phase 1 budget. *(micro-bench per phase, P12)*
- **AT-25** Frame time holds the Phase 1 baseline within tolerance under full combat (benchmark line vs Phase 1). *(P12)*
- **AT-26** Android-Chrome FPS recorded via the WebGL2 backend (target 30–60). *(P12)*

**Baseline integrity**
- **AT-27** No dead code, unused params/imports/exports introduced; line delta accounted for. *(`git diff --shortstat` + review, ongoing)*
- **AT-28** Any removed test has a same-commit replacement or a reviewer-visible justification; running test floor met each phase. *(ongoing)*

**Character roster & staging**
- **AT-29** All roster `CharacterConfig`s load and map to valid rig params; **every** character renders from config. *(render-sweep contact sheet, P11)*
- **AT-30** Character-select picks player + dummy identity; the chosen pair renders; the **sim carries no character id** (firewall — `rg` the sim for character/roster terms → none). *(P10)*
- **AT-31** Tekken-5 staging present — side-3/4 tracking camera (dolly in/out + track), key+rim lighting + per-stage grade, walled arena + hazy depth, chrome lifebars + portrait-grid select. *(recording, P11)*

---

## Assumptions

1. **MVP is local training mode; the sim is built rollback-ready but no netcode ships.** If online is never planned, the determinism/serialization work is still cheap insurance; if it ever conflicts with delivery, revisit firewall strictness. (Veto-able.)
2. **Desktop Chrome is the contract.** WebGPU is guaranteed there; the WebGL2 backend is unverified beyond Android-Chrome smoke and is not a supported target on its own.
3. The 60 Hz fixed tick is the simulation contract; changing it re-derives all move frame data in code, not in this plan.
4. Howler latency is acceptable for impact feedback; swapping to a thin Web Audio wrapper later is a render/audio-layer change that does not touch the sim.
5. **Effect-TS v4 is beta.** We pin the exact version **`4.0.0-beta.78`** (no `^`/`~`) and accept that breaking changes may land before stable; upgrades are deliberate, tested events. Effect's own guidance recommends v3 for production — if v4-beta churn ever blocks a phase, the escape hatch is, **in order: (a) drop to plain `async`/`try`-`catch` in `platform/**` (smallest, fully contained), then (b) drop to Effect v3** only if a service/layer is worth preserving. Prefer (a): v4 (`effect-smol`) and v3 diverge enough (module layout, some constructor/export names) that a v3 downgrade is effectively a platform-layer rewrite, whereas plain-async touches the two `platform` files and nothing else. Either path stays inside `src/platform`, never the sim.
6. **Determinism = same-build replay on one machine.** JS double math is deterministic for identical operations in identical order on the same engine; that satisfies AT-02/AT-04. Cross-platform/cross-client determinism (true rollback) would require fixed-point and is a Non-Goal.
7. The combat numbers in § Combat Specification are **tunable defaults** chosen for consistency + testability, not balance-final values.
8. **Character identity is presentation-only; the sim is character-agnostic.** "All characters wired" = authoring N `CharacterConfig`s over one shared rig + one shared kit (§ Character Roster). It adds config data, render theming, and a select screen — **not** sim/combat surface. So roster size scales config + visual-verification work, not engine work.
9. **Tekken 5 is an art-direction/staging reference, not an asset source.** Build is **private/educational** — Tekken characters/likenesses are Bandai Namco IP and this is **not distributed**. The Fandom gallery and Sketchfab models are **visual reference** for authoring geometric configs, **not** imported assets. If distribution is ever wanted, swap to original-inspired characters (a `config/characters.ts` data change; no code/sim impact).

---

## Execution Notes

- **Skill stack for execution:** Gold Standard phased execution; mark `[x]` only on shipped + evidenced work; per-phase guardrail line is mandatory (already inlined).
- **Effect v4-beta hygiene:** pin the exact `effect@4.0.0-beta.78` in `package.json` (no `^`/`~`); keep any `@effect/*` package on the *same* beta; CI installs from the lockfile; an Effect upgrade is its own commit with the platform tests re-run. See § Platform Layer for the full package/runtime/Schema contract.
- **WebGPU bootstrap order is load-bearing:** every entry path (`main.ts`, any test harness that renders) `await`s `renderer.init()` and logs the backend before the loop. Never call `render()` ahead of init.
- **Playwright is real Chrome:** `channel: 'chrome'`, not bundled Chromium, with the Phase 12 WebGPU flags; record the backend the harness actually exercised rather than failing the gate on CI GPU limits.
- **Scope discipline:** tests/lint scoped to each phase's edits; the only repo-wide cold run is Phase 12.
- **`rng.ts` stays absent** until a sim consumer exists (EC-1); reintroducing it is a deliberate, justified step under the Simplicity gate.
- **Tunables live in `config/constants.ts`:** changing frame data, gravity, `MAX_JUGGLES`, arena width, or scaling constants is a code edit, not a plan revision.

---

## Reference Links

- WebGPU implementation status: https://github.com/gpuweb/gpuweb/wiki/Implementation-Status
- WebGPU in browsers (web.dev): https://web.dev/blog/webgpu-supported-major-browsers
- Three.js WebGPURenderer (async init): https://threejs.org/manual/en/webgpurenderer.html
- Three.js WebGPU examples: https://threejs.org/examples/?q=webgpu
- Fixed-timestep loop (Gaffer): https://gafferongames.com/post/fix_your_timestep/
- Effect-TS v4 beta: https://effect.website/blog/releases/effect/40-beta/
- Effect-TS: https://effect.website
- Howler.js: https://howlerjs.com/
- zustand vanilla store: https://zustand.docs.pmnd.rs/apis/create-store
- Playwright + headless WebGPU flags: https://michelkraemer.com/enable-gpu-for-slow-playwright-tests-in-headless-mode/
- Tekken 5 (roster + lore, Fandom): https://tekken.fandom.com/wiki/Tekken_5
- Tekken 5 art gallery (character reference): https://tekken.fandom.com/wiki/Tekken_5/Gallery
- Tekken 3D models (visual reference only — NOT imported): https://sketchfab.com/tags/tekken
- Tekken 5 stage reference (Tekken Warehouse): https://tekkenwarehouse.com/tekken5/stages/

---

## Appendix — Source drafts (history, not execution inputs)

- `mirror-match-foundation-plan.md` (superseded by this plan; not present in tree)
- `mirror-match-game-logic-prompt.md` (former external spec — **now inlined as § Combat Specification**; the external file is no longer a dependency)
