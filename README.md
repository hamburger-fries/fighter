# Mirror Match (Chrome Fighter) — Final Build Summary

**Mirror Match** is a greenfield 2.5D training-mode fighter targeting **desktop Chrome** at **60 FPS** with:

- **TypeScript (strict) + Vite + Bun**
- **Three.js WebGPU renderer (primary)** with **WebGL2 fallback** scoped to Android-Chrome coverage
- A **deterministic, fixed-timestep (60 Hz), headless simulation** in `src/sim/`
- **Bare Three.js rendering** (no React/R3F), HTML/CSS HUD, Howler audio
- **Effect-TS only in platform boundaries** (`src/platform/`)
- **Vitest** for sim correctness + replay determinism, and **Playwright** for final Chrome smoke validation

## Core Architectural Contract

- `src/sim/` is pure and serializable: no renderer, DOM, audio, Effect, timers, or network.
- Simulation is frame-driven (`step(state, inputs)`), deterministic, and replay-hash stable.
- Render/UI/audio are one-way consumers of sim state; they never mutate combat state directly.

## Delivery Goal

Ship a playable single-player training mode where the player controls **Leo** against a configurable **Aries** dummy, with all eight moves and combat systems (hit/block reactions, launch/juggle, sweep/knockdown/wakeup, wall bounce, combo scaling) behaving correctly in headless tests and in Chrome runtime.

## Plan Shape

The plan is execution-gated and evidence-driven across **Phase 0 → Phase 11 + Final Audit**:

1. Baseline/tooling and WebGPU bootstrap
2. Deterministic loop + replay proof
3. Rig, input/FSM, move system, hit detection
4. Block/hitstun/knockback, launch/juggle, sweep/knockdown, wall/ground bounce
5. Combo manager + scaling
6. Visual/audio feedback, HUD, dummy controls, persistence
7. Zodiac arena polish + final performance and full audit

A phase is complete only when its required tests, static checks, performance checks, and Chrome-visible evidence pass and are recorded.

## Out of Scope (MVP)

- Online/netcode
- Broad cross-browser support beyond Chrome target
- Full AI opponent (dummy is scripted)
- Heavy physics/animation pipelines and non-essential content systems

---