# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Der Planer Ultimate" — a German-language freight/logistics dispatcher simulation game. The **entire game is one file**: `index.html` (~31,400 lines, ~1.3 MB) with embedded CSS and JavaScript. No build step, no dependencies, no framework, no package manager. It runs by opening the file in a browser.

**Project history:** `git log` is the authoritative record of what has been built and why — every change is its own commit with a detailed message. To learn what was implemented recently (features, fixes, refactors), read `git log` rather than relying on any conversation summary.

## Running & verifying

- **Run locally:** open `index.html` directly in a browser, or serve it: `python -m http.server 8765`, then open `http://localhost:8765/index.html`. The Claude Code preview config is in `.claude/launch.json` (config name: `planer`).
- **There is no test suite and no linter.** After editing JavaScript, verify syntax by extracting both `<script>` blocks and running `node --check` on each (run from the repo directory):
  ```
  node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)];fs.writeFileSync('C:/tmp/_s0.js',m[0][1]);fs.writeFileSync('C:/tmp/_s1.js',m[1][1]);" && node --check C:/tmp/_s0.js && node --check C:/tmp/_s1.js
  ```
  `node --check` only catches syntax errors. To verify behaviour, load the game in a browser/preview and exercise the affected view.
- In-game diagnostics live in the `SelfTest` and `GameDebugger` modules.

## File layout of index.html

- ~1–721: `<head>`, CSS, page markup.
- **Header `<script>` (~722–939):** runs first. `HoverTooltip`, `GameClock` (the real-time day/clock loop), `EveningScreen` (the sleep screen). These attach to `window`.
- ~940–1793: more HTML markup (the `#view-*` view containers).
- **Main `<script>` (~1794–31406):** all game data, logic and rendering.
- A `DOMContentLoaded` handler at the very end boots the game.

Line numbers drift as the file is edited — locate things by grepping the stable name (e.g. `^const JobManager = {`), not by remembered line number.

## Architecture

**One global mutable `State` object holds the entire game.** It is built by `_planerDefaultState()` (~line 4641) and assigned to `const State` (~4757). Starting a new game does `Object.assign(State, _planerDefaultState())` (in the `Game` module).

⚠️ **Consequence:** any persistent state field MUST be declared in `_planerDefaultState()`. Fields written ad-hoc as `State.something` (common for `_`-prefixed runtime flags) are not reset on a new game and leak between playthroughs — only acceptable when every read guards with `|| default`.

**~95 module singletons.** Every subsystem is a global object literal `const ModuleName = { ... }`. There is no module system — everything shares one scope and modules call each other directly by name (`EventManager.addNews(...)`, `Utils.euro(...)`). Methods resolve at call time, so definition order rarely matters.

Key architectural anchors:
- **`Game`** (~25280) — lifecycle: new game, day rollover (`nextDay`), win/lose.
- **`GameClock`** (header script) — drives the in-game clock and speed.
- **`UI`** (~25581 to end of script, ~5,800 lines) — *all* views and rendering. Builds HTML strings into `#view-*` containers; `UI.showView(name)` switches view.
- **`EventManager`** (~23904) — `queueDecision(...)` (modal decisions) and `addNews(...)` (news feed).
- **`DATA`** (~1795) — static game data/config tables.
- **`Utils`** (~4595) — shared helpers: `euro`, `clamp`, `pick`, `randInt`.

Subsystem groups (grep `^const <Name> = {` to open one):
- Fleet/trucks: `FleetManager`, `TruckSpec`, `TrailerManager`, `MaintenancePlan`, `DealerManager`, `VehicleTuning`, `LeasingManager`
- Jobs/routing: `JobManager`, `RouteManager`, `DispoManager`, `RoadRouting`, `Pathfinding`, `SpotMarket`, `ContractSLA`
- Driving conditions: `DrivingTime`, `TrafficSystem`, `SundayBan`, `WeatherRoads`, `RoadClosures`, `BorderDelays`
- Staff/HR: `EmployeeManager`, `StaffDepth`, `StaffPersonality`, `DriverSkills`, `DriverCareer`, `StaffDevelopment`
- Finance: `FinanceManager`, `CashflowSystem`, `BankAccount`, `LoanManager`, `AnnualClose`
- Customers/market: `CustomerManager`, `MarketingManager`, `Competition`, `MarketCycles`, `MarketIntel`
- Private life ("Privatleben"): `MorningRoutine`, `PlayerManager`, `LifeEvents`, `GFOffice`, plus `EveningScreen` (header script); the view is `UI.renderPrivate`
- Events/crises: `AdvancedEvents`, `CrisisSystem`, `PoliticalEvents`, `SeasonCalendar`
- Onboarding/meta: `Tutorial`, `Onboarding`, `Campaign`, `SelfTest`, `GameDebugger`

## Conventions & gotchas

- **Stay single-file.** The game ships by opening `index.html`. Do not introduce external files, imports, or a build step.
- **Template-literal HTML.** Views are built as template-literal strings that use **literal `\n`** (backslash + n) for line breaks and **`\x3c`** in place of `<` (so an embedded `</script>` cannot break the page). When editing inside these strings, match those escapes exactly.
- The code is auto-formatted but **very dense** — many extremely long lines, especially in `UI`. Navigate with `Grep`; avoid reading huge ranges blindly.
- **Work one subsystem at a time.** The file is too large to hold entirely in context — scope each change to a single module/area and read it fresh rather than relying on memory.
- Game-design invariant: time off / vacation is a *recovery action only* and must **not** skip calendar days — skipping days froze the company (no salaries, jobs or events ran).
- **Help content ships with the feature.** Every player-visible change (feature, balance, mechanic rework) updates the in-game help in the SAME commit: FAQ, Einstieg, Systeme overview (incl. its `isActive` discovery detector), Tipps, and Glossar (`Glossary.TERMS`) — whichever are touched by the change. Verify numbers in help texts against the actual code, never from memory; stale help text is treated as a UI lie (see audit class A3/M31-M33).
