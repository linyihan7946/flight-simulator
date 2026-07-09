# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-page 3D flight simulator built entirely in one HTML file with inline CSS and JavaScript, using Three.js (vendored as `three.min.js`). No build system, no package manager, no tests. The UI is in Chinese (lang="zh").

Open `index.html` directly in a browser to run. There is no build step, dev server, or linting configured.

## Architecture

Everything lives in `index.html` (~3900 lines). The code is organized into these inline sections:

- **CSS** (lines 7-832): All styling inline in `<style>`. HUD, mobile controls, cockpit overlay, warnings, menus.
- **HTML** (lines 833+): `<script src="./three.min.js">` then one large `<script>` block containing all game logic.
- **Game state & menu** (~line 836): `gameActive`, `selectedAirportIndex`, `selectedAirplaneType`, dropdown population. `window.startGame()` / `window.backToMenu()` toggle between menu and simulation.
- **Scene setup** (~line 992): `THREE.Scene`, `PerspectiveCamera`, `WebGLRenderer` with shadow maps. `cameraMode` is `'chase'` or `'cockpit'`.
- **Airplane models** (`createAirplaneModel`, ~line 1223): Procedural Three.js geometry for 34 aircraft types. Each type has a config block inside this function defining fuselage dimensions, wing span, engine positions/scales, T-tail flag, hump (747), double-deck (A380), etc.
- **Physics configs** (`airplaneConfigs`, ~line 1181): Per-type `maxSpeed`, `minSpeed`, `accel`, `drag`, `pitchRate`, `rollRate`, `yawRate`, `liftFactor`, `enginesCount`. Cargo weight modifies these via `weightMul` in `resetGame()`.
- **Environment**: `createAirport` (3 airports with runways), `createMountain`/`createCity`/`createTree` for terrain. `isInRunwayArea()` enforces no-spawn/no-build zones near runways. Collision bodies tracked in a set.
- **Audio** (~line 2135): Web Audio API oscillators for beeps (`playBeep`, `playSquareBeep`) + `SpeechSynthesis` for voice warnings (`speak`). Stall, GPWS, gear, overspeed alert loops.
- **Warnings** (`updateWarnings` ~line 2287, `updateWarningHUD` ~line 2384): Stall detection, GPWS (terrain proximity), gear-down overspeed, bank angle warnings.
- **Autopilot & obstacle avoidance** (~line 2447): Altitude/heading hold. `computeObstacleAvoidance()` scans for terrain collisions and steers away.
- **Mobile controls** (~line 2563): Virtual joystick (pitch only, maps to W/S keys) + touch buttons for throttle, rudder, flaps, gear, etc. Uses pointer events with `touchHeldKeys` set to merge touch + keyboard state into the shared `keys` object.
- **Crash/explosion** (~line 2782): Particle system (`triggerExplosion`, `triggerBreakApart`, `updateExplosion`).
- **Main loop** (`animate`, ~line 2970): `requestAnimationFrame`. Order per frame:
  1. Air density calculation (drops to 0 between 90km-100km altitude — engines flame out, controls go dead)
  2. Throttle/fuel/engine thrust updates (per-engine, supports asymmetric failure)
  3. Asymmetric yaw moment from engine failure
  4. Aerodynamic control surfaces (pitch/roll/yaw scaled by air density)
  5. Stall logic, autopilot, auto-avoidance
  6. Position integration, ground collision
  7. `updateHUD()`, camera update, render
- **HUD** (`updateHUD`, ~line 3605): Speed (km/h + Mach), altitude, vertical speed, compass, gear status, reverse thrust status, engine status panel, warning lights.
- **State object** (`state`, reset in `resetGame` ~line 2916): `speed`, `fuel`, `throttle`, `engines[]`, `autopilot`, `autoAvoidance`, `gearDown`/`gearProgress`, `reverseThrust`, failure trigger modes. Replaced wholesale each reset.

## Key Controls (Keyboard)

| Key | Action |
|-----|--------|
| W/S | Pitch down/up |
| A/D | Roll left/right |
| Q/E | Yaw left/right |
| Shift/Ctrl | Throttle up/down |
| G | Toggle landing gear |
| C | Toggle cockpit/chase camera |
| P | Toggle autopilot |
| V | Toggle auto obstacle avoidance |
| F + 1-4 | Trigger engine failure (when armed) |
| R | Reset after crash |

## Patterns & Conventions

- **Adding a new aircraft**: Add entry to `airplaneOptions` array, add physics params to `airplaneConfigs`, add geometry branch inside `createAirplaneModel()` (pattern: set `r1/r2/len/wingTip/enginesConfig/tTail/noseX/tailX` then fall through to shared fuselage/wing/tail/engine builder).
- **Adding a new airport**: Add to `airportOptions` array, add config to `airports` array (with `startPos`, `startRotationY`, runway geometry).
- **Global scope**: Most game state is in global/module-scope variables. Functions exposed to HTML `onclick` via `window.xxx = ...`.
- **Touch + keyboard unification**: Both input paths write to the same `keys` object. `touchHeldKeys` tracks which keys are held by touch so keyboard release doesn't clear a touch-held key.
- **Three.js version**: Vendored `three.min.js` — check available APIs before using newer features (e.g. `thickness` on `MeshPhysicalMaterial` was already removed due to version incompatibility).
- **Language**: UI strings, comments, and commit messages are in Chinese.
