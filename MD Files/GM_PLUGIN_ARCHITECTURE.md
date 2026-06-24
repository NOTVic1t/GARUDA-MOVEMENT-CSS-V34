# GM_PLUGIN_ARCHITECTURE.md
## Garuda Movement — HNS Chase Ecosystem
### SourceMod Modular Plugin Architecture Specification
### Version 1.0 | June 2026

---

> **Document Status:** OFFICIAL PLUGIN ARCHITECTURE SPECIFICATION
> **References:**
> — GM_HNS_MASTER_SPEC_V1_FINAL.md (V1.0 FINAL — LOCKED)
> — GM_DATABASE_SPEC.md (V1.0 — LOCKED)
> **Classification:** Internal Architecture
> **Maintainer:** Lead Technical Architect, Garuda Movement
> **Target Framework:** SourceMod 1.11+
> **Target Game:** Counter-Strike: Source v34
> **Scope:** Complete plugin architecture for GM-HNS V1.0

---

## TABLE OF CONTENTS

1. [Architecture Philosophy](#1-architecture-philosophy)
2. [Repository Structure](#2-repository-structure)
3. [Module Registry](#3-module-registry)
4. [Plugin Load Order](#4-plugin-load-order)
5. [Include File Structure](#5-include-file-structure)
6. [Module Specifications](#6-module-specifications)
7. [Shared API Reference](#7-shared-api-reference)
8. [Forward and Event Definitions](#8-forward-and-event-definitions)
9. [Native Definitions](#9-native-definitions)
10. [Database Ownership Map](#10-database-ownership-map)
11. [Inter-Plugin Communication Protocol](#11-inter-plugin-communication-protocol)
12. [Configuration Architecture](#12-configuration-architecture)
13. [Logging Architecture](#13-logging-architecture)
14. [Failure Handling](#14-failure-handling)
15. [Discord Webhook Architecture](#15-discord-webhook-architecture)
16. [Build System](#16-build-system)
17. [Future Compatibility Provisions](#17-future-compatibility-provisions)

---

## 1. ARCHITECTURE PHILOSOPHY

### 1.1 Core Model: Modular Monorepo

GM-HNS follows a **modular monorepo** plugin architecture. All modules share:
- A single source code repository.
- A single shared include library (`gm_core`).
- A single database connection pool.
- A single configuration root.
- A single logging subsystem.

Each module compiles to an independent SourceMod plugin binary (`.smx`). Binaries load, unload, and hot-reload independently. No module's failure may crash the server, terminate the active round, or disconnect players.

### 1.2 Three-Layer Model

The architecture is organized in three conceptual layers:

```
┌─────────────────────────────────────────────────────┐
│  LAYER 3 — PRESENTATION                             │
│  gm_menu  |  gm_commands  |  gm_hud  |  gm_admin   │
├─────────────────────────────────────────────────────┤
│  LAYER 2 — FEATURE DOMAINS                          │
│  gm_account  |  gm_region    |  gm_team             │
│  gm_gameplay |  gm_movement  |  gm_antiexploit      │
│  gm_jumpstats|  gm_xp        |  gm_rank             │
│  gm_achievements | gm_cosmetics | gm_training       │
├─────────────────────────────────────────────────────┤
│  LAYER 1 — FOUNDATION                               │
│  gm_core (shared include library)                   │
│  DB Pool | Event Bus | Lang | Config | Logging       │
└─────────────────────────────────────────────────────┘
```

- **Layer 1 (Foundation):** `gm_core` — compiled into all modules. No standalone binary.
- **Layer 2 (Feature Domains):** Independent plugin binaries. Each owns a bounded set of game logic, database tables, and game state.
- **Layer 3 (Presentation):** Stateless routing and rendering modules. They call Layer 2 natives/events; they own no game state themselves.

### 1.3 Bounded Ownership

Every piece of game state has exactly one owner module. No two modules write to the same database table without explicit cross-module write authorization documented in Section 10. No module reads another module's in-memory state directly — it uses declared natives.

### 1.4 Communication Hierarchy

Modules communicate through exactly three channels, in order of preference:

1. **GM Event Bus** — preferred. Fire-and-forget. Asynchronous. Fully decoupled.
2. **Declared Natives** — used when a synchronous return value is needed. Sparingly applied.
3. **Shared Database** — used for persistent data exchange across sessions or servers. Read-only access to another module's tables is permitted via documented query patterns only.

Direct plugin handle calls, `GetPluginByFile()`, or any undeclared cross-plugin invocation are **prohibited**.

---

## 2. REPOSITORY STRUCTURE

```
garuda-movement/
│
├── addons/
│   └── sourcemod/
│       │
│       ├── scripting/
│       │   ├── include/
│       │   │   ├── gm_core.inc          ← Foundation library (compiled into all modules)
│       │   │   ├── gm_events.inc        ← All GM forward/event declarations
│       │   │   ├── gm_natives.inc       ← All GM native declarations
│       │   │   ├── gm_enums.inc         ← All shared enum definitions
│       │   │   ├── gm_constants.inc     ← All shared constant definitions
│       │   │   └── gm_structs.inc       ← All shared struct/methodmap declarations
│       │   │
│       │   ├── gm_account.sp
│       │   ├── gm_region.sp
│       │   ├── gm_movement.sp
│       │   ├── gm_team.sp
│       │   ├── gm_gameplay.sp
│       │   ├── gm_jumpstats.sp
│       │   ├── gm_xp.sp
│       │   ├── gm_rank.sp
│       │   ├── gm_achievements.sp
│       │   ├── gm_cosmetics.sp
│       │   ├── gm_training.sp
│       │   ├── gm_hud.sp
│       │   ├── gm_antiexploit.sp
│       │   ├── gm_menu.sp
│       │   ├── gm_commands.sp
│       │   └── gm_admin.sp
│       │
│       ├── plugins/
│       │   └── (compiled .smx binaries go here)
│       │
│       ├── configs/
│       │   ├── gm_config.cfg            ← Master configuration
│       │   ├── gm_gameplay.cfg
│       │   ├── gm_team.cfg
│       │   ├── gm_jumpstats.cfg
│       │   ├── gm_xp.cfg
│       │   ├── gm_rank.cfg
│       │   └── gm_training.cfg
│       │
│       ├── translations/
│       │   ├── gm_account.phrases.txt
│       │   ├── gm_gameplay.phrases.txt
│       │   ├── gm_jumpstats.phrases.txt
│       │   ├── gm_xp.phrases.txt
│       │   ├── gm_rank.phrases.txt
│       │   ├── gm_achievements.phrases.txt
│       │   ├── gm_cosmetics.phrases.txt
│       │   ├── gm_training.phrases.txt
│       │   ├── gm_hud.phrases.txt
│       │   ├── gm_menu.phrases.txt
│       │   ├── gm_commands.phrases.txt
│       │   └── gm_admin.phrases.txt
│       │
│       └── logs/
│           ├── gm_account.log
│           ├── gm_gameplay.log
│           ├── gm_jumpstats.log
│           ├── gm_xp.log
│           ├── gm_rank.log
│           ├── gm_achievements.log
│           ├── gm_cosmetics.log
│           ├── gm_training.log
│           ├── gm_antiexploit.log
│           └── gm_admin.log
│
├── maps/
│   └── metadata/
│       └── <mapname>.json               ← Per-map metadata files
│
└── tools/
    ├── build.sh                         ← Compilation script
    └── deploy.sh                        ← Deployment script
```

---

## 3. MODULE REGISTRY

Complete V1.0 module inventory with classification and dependency declaration.

| # | Module ID | Binary | Layer | Tier | Hard Dependencies |
|---|---|---|---|---|---|
| 01 | `gm_core` | `.inc` only | 1 | Foundation | None |
| 02 | `gm_account` | `gm_account.smx` | 2 | T1 Hot | `gm_core` |
| 03 | `gm_region` | `gm_region.smx` | 2 | T1 | `gm_core`, `gm_account` |
| 04 | `gm_movement` | `gm_movement.smx` | 2 | T1 Hot | `gm_core` |
| 05 | `gm_team` | `gm_team.smx` | 2 | T1 Hot | `gm_core`, `gm_account` |
| 06 | `gm_gameplay` | `gm_gameplay.smx` | 2 | T1 Hot | `gm_core`, `gm_team`, `gm_movement` |
| 07 | `gm_jumpstats` | `gm_jumpstats.smx` | 2 | T2 Warm | `gm_core`, `gm_account`, `gm_movement` |
| 08 | `gm_xp` | `gm_xp.smx` | 2 | T2 Warm | `gm_core`, `gm_account` |
| 09 | `gm_rank` | `gm_rank.smx` | 2 | T2 Warm | `gm_core`, `gm_account`, `gm_xp` |
| 10 | `gm_achievements` | `gm_achievements.smx` | 2 | T2 Warm | `gm_core`, `gm_account`, `gm_xp` |
| 11 | `gm_cosmetics` | `gm_cosmetics.smx` | 2 | T2 | `gm_core`, `gm_account` |
| 12 | `gm_training` | `gm_training.smx` | 2 | T2 | `gm_core`, `gm_account`, `gm_jumpstats`, `gm_xp` |
| 13 | `gm_hud` | `gm_hud.smx` | 3 | T1 Hot | `gm_core`, `gm_jumpstats`, `gm_movement` |
| 14 | `gm_antiexploit` | `gm_antiexploit.smx` | 2 | T1 Hot | `gm_core`, `gm_movement` |
| 15 | `gm_menu` | `gm_menu.smx` | 3 | T2 | `gm_core` + all L2 modules |
| 16 | `gm_commands` | `gm_commands.smx` | 3 | T2 | `gm_core` + all L2 modules |
| 17 | `gm_admin` | `gm_admin.smx` | 3 | T3 | `gm_core` + all modules |

**Tier definitions** match GM_DATABASE_SPEC.md Section 1.2: T1=Hot (every tick or every connect), T2=Warm (round-frequency writes), T3=Cold (infrequent admin actions).

---

## 4. PLUGIN LOAD ORDER

SourceMod loads plugins in the order declared in `addons/sourcemod/configs/plugins.ini`. The following order is **mandatory** and enforced by the build system.

```
; ============================================================
; GARUDA MOVEMENT — plugins.ini
; Load order is dependency-ordered. DO NOT reorder.
; ============================================================

; Layer 1 — Foundation (include only, no entry needed)
; gm_core.inc is compiled into all modules below.

; Layer 2 — Feature Domains (dependency order)
garuda/gm_account          ; Identity, sessions, preferences
garuda/gm_region           ; GeoIP region assignment
garuda/gm_movement         ; Movement validation, speed cap
garuda/gm_team             ; Team ratio, CT Queue, auto-balance
garuda/gm_gameplay         ; Round rules, FrostNades, countdown
garuda/gm_jumpstats        ; Jump detection, validation, leaderboard
garuda/gm_xp               ; XP awards, level management
garuda/gm_rank             ; Rank score, tier, position
garuda/gm_achievements     ; Achievement evaluation and rewards
garuda/gm_cosmetics        ; Cosmetic catalog, equip, rendering
garuda/gm_training         ; Training mode, checkpoints, challenges
garuda/gm_antiexploit      ; Server-side cheat detection

; Layer 3 — Presentation (load last; depend on all L2)
garuda/gm_hud              ; HUD overlays
garuda/gm_menu             ; Menu rendering and navigation
garuda/gm_commands         ; Command registration and routing
garuda/gm_admin            ; Admin commands and audit logging
```

### 4.1 Load Order Rationale

| Position | Module | Reason |
|---|---|---|
| 1st | `gm_account` | Every other module needs `account_id`. Must be ready before any player event. |
| 2nd | `gm_region` | Assigned at first connect; must be ready before `gm_account` fires `GM_OnPlayerConnect`. |
| 3rd | `gm_movement` | Required by both `gm_team` (movement validation reference) and `gm_gameplay` (speed cap). |
| 4th | `gm_team` | Required by `gm_gameplay` for team query natives. |
| 5th | `gm_gameplay` | Must load before round events fire. Depends on `gm_team` and `gm_movement`. |
| 6th | `gm_jumpstats` | Subscribes to `GM_OnCountdownGo` to reset per-round tracking. |
| 7th | `gm_xp` | Subscribes to `GM_OnRoundEnd` and `GM_OnJumpRecorded`. Depends on `gm_account`. |
| 8th | `gm_rank` | Subscribes to `GM_OnXPAwarded` and `GM_OnJumpRecorded`. Depends on `gm_xp`. |
| 9th | `gm_achievements` | Subscribes to multiple events from all earlier modules. Must load after them all. |
| 10th | `gm_cosmetics` | Depends on `gm_account` for ownership data. No event ordering dependency. |
| 11th | `gm_training` | Depends on `gm_jumpstats` and `gm_xp` natives. |
| 12th | `gm_antiexploit` | Subscribes to movement events fired by `gm_movement`. |
| 13th | `gm_hud` | Presentation. Depends on `gm_jumpstats` and `gm_movement` data natives. |
| 14th | `gm_menu` | Presentation. Calls all L2 natives for data display. |
| 15th | `gm_commands` | Presentation. Routes to L2 and L3 modules. |
| 16th (last) | `gm_admin` | Presentation. Must load after all modules it administers. |

### 4.2 Dependency Failure at Load Time

If a required dependency module fails to load, the dependent module must:
1. Log a CRITICAL error to its own log file.
2. Set itself to a degraded-mode state.
3. **Not** call `SetFailState()` — the server must continue running.

Exception: `gm_account` failure is unrecoverable. If `gm_account` fails to load, the server cannot safely associate any player with an identity. In this case only, `gm_account` failing should be treated as a Severity-1 incident requiring immediate human intervention, and the remaining modules should refuse to serve player requests until `gm_account` is restored.

---

## 5. INCLUDE FILE STRUCTURE

### 5.1 `gm_core.inc` — Master Include

Every `.sp` file includes `gm_core.inc` as its first include after SourceMod standard includes.

`gm_core.inc` is a meta-include that pulls in all sub-includes in the correct order:

```
gm_core.inc
  ├── #include <sourcemod>
  ├── #include <sdktools>
  ├── #include <cstrike>        ← CS:S specific
  ├── #include "gm_enums.inc"
  ├── #include "gm_constants.inc"
  ├── #include "gm_structs.inc"
  ├── #include "gm_events.inc"
  └── #include "gm_natives.inc"
```

No module includes sub-files directly. All imports go through `gm_core.inc`.

### 5.2 `gm_enums.inc` — Shared Enumerations

Defines all typed enum values used across modules. Every TINYINT UNSIGNED enum from `GM_DATABASE_SPEC.md` Appendix B has a corresponding SourcePawn enum defined here.

Enumerations defined in `gm_enums.inc`:

| Enum Name | Values | Maps To DB Column |
|---|---|---|
| `GMAccountType` | Guest=0, LinkedSteam=1, LinkedDiscord=2, LinkedBoth=3 | `accounts.account_type` |
| `GMAdminLevel` | Player=0, Admin=1, Owner=2 | `accounts.admin_level` |
| `GMJumpType` | LJ=1, CJ=2, MCJ=3, LDJ=4, DBhop=5 | `jumpstat_records.jump_type` |
| `GMRoundType` | Standard=0, Training=1 | `rounds.round_type` |
| `GMWinnerTeam` | Draw=0, Hiders=1, Chasers=2 | `rounds.winner_team` |
| `GMParticipantTeam` | Hider=1, Chaser=2 | `round_participants.team` |
| `GMXPSourceType` | Gameplay=1, JumpStat=2, TrainingChallenge=3, Achievement=4 | `xp_transactions.source_type` |
| `GMAchievementCategory` | Survival=1, Chaser=2, Movement=3, Progression=4, Training=5, Hidden=6 | `achievement_catalog.category` |
| `GMAchievementTier` | Bronze=1, Silver=2, Gold=3, Platinum=4 | `achievement_catalog.tier` |
| `GMCosmeticCategory` | Knife=1, Trail=2, Title=3, Tag=4 | `cosmetic_catalog.category` |
| `GMCosmeticAcquisition` | LevelReward=1, AchievementReward=2, AdminGrant=3, AutoRank=4 | `cosmetic_catalog.acquisition_type` |
| `GMRankAxis` | Gameplay=1, Movement=2 | `rank_scores.axis` |
| `GMRankTier` | Newcomer=1, Amateur=2, Skilled=3, Expert=4, Master=5, Grandmaster=6, Legend=7, Garuda=8 | `rank_scores.tier` |
| `GMFriendStatus` | Pending=0, Accepted=1, Declined=2, Blocked=3 | `friends.status` (reserved) |
| `GMDBStatus` | OK=0, Degraded=1, Offline=2 | Internal pool health |
| `GMLogLevel` | Debug=0, Info=1, Warn=2, Error=3, Critical=4 | Internal logging |

### 5.3 `gm_constants.inc` — Shared Constants

Defines all numeric and string constants shared across modules.

Constants defined in `gm_constants.inc`:

| Constant | Value | Used By |
|---|---|---|
| `GM_MAX_PLAYERS` | 32 | All |
| `GM_RECOMMENDED_PLAYERS` | 24 | `gm_team` |
| `GM_TICK_RATE` | 66 | `gm_jumpstats`, `gm_movement` |
| `GM_CT_FREEZE_DURATION` | 5 | `gm_gameplay` |
| `GM_COUNTDOWN_START` | 5 | `gm_gameplay` |
| `GM_HIDER_RATIO` | 0.70 | `gm_team` |
| `GM_CHASER_RATIO` | 0.30 | `gm_team` |
| `GM_FROSTNADE_COUNT` | 2 | `gm_gameplay` |
| `GM_FROSTNADE_DURATION` | 3.0 | `gm_gameplay` |
| `GM_FROSTNADE_RADIUS` | 150.0 | `gm_gameplay` |
| `GM_HIDER_SPAWN_GRACE` | 20 | `gm_gameplay` |
| `GM_CHASER_SPAWN_GRACE` | 10 | `gm_gameplay` |
| `GM_TRAINING_VOTE_DURATION` | 30 | `gm_training` |
| `GM_TRAINING_VOTE_COOLDOWN` | 300 | `gm_training` |
| `GM_LEVEL_MAX` | 100 | `gm_xp` |
| `GM_LEVEL_XP_BASE` | 500 | `gm_xp` |
| `GM_LEVEL_XP_MULTIPLIER` | 1.12 | `gm_xp` |
| `GM_DB_POOL_MIN` | 4 | `gm_core` |
| `GM_DB_POOL_MAX` | 16 | `gm_core` |
| `GM_DB_QUERY_TIMEOUT` | 5.0 | `gm_core` |
| `GM_DB_TX_TIMEOUT` | 10.0 | `gm_core` |
| `GM_WEBHOOK_RETRY_MAX` | 3 | `gm_admin` |
| `GM_KNIFE_DEFAULT` | "knife_default" | `gm_cosmetics` |
| `GM_TRAIL_DEFAULT` | "trail_off" | `gm_cosmetics` |
| `GM_JUMP_TYPES_V1` | 5 | `gm_jumpstats` |

### 5.4 `gm_structs.inc` — Shared Structs and Methodmaps

Defines all structured data types used in native function signatures and event payloads.

Key structs defined:

**`GMPlayerCache`** — In-memory player state. One instance per connected client slot.

| Field | Type | Source |
|---|---|---|
| `account_id` | int | `accounts` |
| `is_loaded` | bool | Internal — true after all connect-time queries complete |
| `is_banned` | bool | `accounts` |
| `username` | char[64] | `accounts` |
| `region_code` | char[12] | `accounts` |
| `admin_level` | GMAdminLevel | `accounts` |
| `account_type` | GMAccountType | `accounts` |
| `language` | char[5] | `player_settings` |
| `total_xp` | int | `player_levels` |
| `current_level` | int | `player_levels` |
| `xp_to_next_level` | int | `player_levels` |
| `gameplay_score` | int | `rank_scores` |
| `gameplay_tier` | GMRankTier | `rank_scores` |
| `gameplay_position` | int | `rank_scores` |
| `movement_score` | int | `rank_scores` |
| `movement_tier` | GMRankTier | `rank_scores` |
| `equipped_knife` | char[32] | `player_settings` |
| `equipped_trail` | char[32] | `player_settings` |
| `equipped_title` | char[32] | `player_settings` |
| `hud_jumpstats` | bool | `player_settings` |
| `hud_speed` | bool | `player_settings` |
| `hud_sync` | bool | `player_settings` |
| `hud_strafe` | bool | `player_settings` |
| `chat_notifications` | bool | `player_settings` |
| `current_team` | GMParticipantTeam | `gm_team` runtime |
| `is_alive` | bool | `gm_gameplay` runtime |
| `current_round_eliminations` | int | `gm_gameplay` runtime |
| `time_alive_this_round` | float | `gm_gameplay` runtime |
| `frostnades_remaining` | int | `gm_gameplay` runtime |

**`GMJumpData`** — A fully measured in-progress or completed jump.

| Field | Type | Description |
|---|---|---|
| `jump_type` | GMJumpType | Jump type enum |
| `distance` | float | Horizontal distance in HU |
| `pre_speed` | float | Velocity at jump input |
| `land_speed` | float | Velocity at landing |
| `strafe_count` | int | Strafe key presses during airtime |
| `sync_pct` | float | Sync percentage 0.0–100.0 |
| `airtime_ticks` | int | Ticks from liftoff to landing |
| `map_name` | char[64] | Map where jump was made |
| `is_valid` | bool | Passed server validation |
| `is_pb` | bool | Is a new personal best |
| `is_server_record` | bool | Is a new server record |

**`GMRoundState`** — Live state of the current round.

| Field | Type | Description |
|---|---|---|
| `round_id` | int | DB round_id |
| `round_type` | GMRoundType | Standard or Training |
| `map_name` | char[64] | Current map |
| `is_active` | bool | Round in progress |
| `is_freeze_time` | bool | CT freeze period |
| `is_countdown` | bool | Countdown sequence active |
| `countdown_value` | int | Current countdown number |
| `hider_count` | int | Live hider count |
| `chaser_count` | int | Live chaser count |
| `round_start_time` | float | Game time at round start |

---

## 6. MODULE SPECIFICATIONS

### 6.1 `gm_core` — Foundation Library

**Type:** Include library (`.inc`). Not a standalone plugin.
**Compiled into:** Every module.
**Database tables owned:** None. Provides access to all.

**Responsibilities:**

- Provide the shared database connection pool and async query interface.
- Provide the GM Event Bus (forward registration and dispatch).
- Provide the Language/Translation resolver.
- Provide the Configuration manager.
- Provide the centralized Logging interface.
- Provide the `GMPlayerCache` array (indexed by client slot, 1–`GM_MAX_PLAYERS`).
- Define all include sub-files (`gm_enums.inc`, `gm_constants.inc`, `gm_structs.inc`, `gm_events.inc`, `gm_natives.inc`).

**Subsystems:**

| Subsystem | Interface Prefix | Description |
|---|---|---|
| Database Pool | `GM_DB_` | Async query, transaction, escape, status |
| Event Bus | `GM_Event_` | Forward fire and subscribe |
| Language | `GM_Lang_` | String formatting with locale resolution |
| Config | `GM_Config_` | Key-value config access, hot-reload |
| Logging | `GM_Log_` | Levelled logging per module |
| Player Cache | `GM_Cache_` | Read/write `GMPlayerCache` per client |

Full API documented in Section 7.

---

### 6.2 `gm_account` — Account and Session Module

**Binary:** `gm_account.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1 Hot
**Database tables owned:** `accounts`, `sessions`, `player_stats`, `player_levels`, `player_settings`

**Responsibilities:**

- Detect player connection (Steam and NonSteam/LSGI).
- Resolve or create `account_id` on every connect.
- Enforce ban check at connect time. Kick banned players with localized ban reason.
- Execute all connect-time database queries (Q-01 through Q-07 per DB Spec Section 12.3).
- Populate the `GMPlayerCache` entry for the connecting player.
- Open a `sessions` record on connect; close it on disconnect.
- Update `accounts.last_seen` and `accounts.total_playtime_seconds` on disconnect.
- Expose player identity data to all other modules via declared natives.
- Handle the `!profile` command by providing raw data to `gm_menu`.
- Guard `GM_OnPlayerConnect` — fire only after `GMPlayerCache` is fully populated.
- On disconnect: write final session data, clear `GMPlayerCache` slot.

**State managed in memory:**

- Full `GMPlayerCache` array (one entry per connected client).
- Session ID per connected client.

**Fires events:**
- `GM_OnPlayerConnect` — after all connect-time DB queries complete and cache is populated.
- `GM_OnPlayerDisconnect` — before cache is cleared.

**Consumes events:** None. This module is the identity root.

---

### 6.3 `gm_region` — Regional Assignment Module

**Binary:** `gm_region.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1
**Database tables owned:** None. Reads and writes `accounts.region_code` only during first connect.

**Responsibilities:**

- Perform GeoIP lookup on first connect (when `accounts.region_code` is NULL or default).
- Write the resolved `region_code` to the `accounts` table.
- Update `GMPlayerCache.region_code` after resolution.
- Expose `GM_Region_GetClientRegion(client)` native.
- Load the GeoIP database file at plugin startup.
- Reload the GeoIP database on `!a reloadconfig`.

**State managed in memory:**

- GeoIP database handle.
- Region code per connected client (cached from `GMPlayerCache`).

**Fires events:** None.

**Consumes events:**
- `GM_OnPlayerConnect` — triggers GeoIP lookup if region not yet assigned.

---

### 6.4 `gm_movement` — Movement Validation Module

**Binary:** `gm_movement.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1 Hot
**Database tables owned:** None. Pure runtime state.

**Responsibilities:**

- Hook `OnPlayerRunCmd` to monitor player input every tick.
- Enforce AutoBhop disable: detect held jump key and suppress additional jump events.
- Enforce the ground speed cap (configured via `gm_gameplay.cfg`).
- Track per-tick player velocity, position, and jump state.
- Detect jump liftoff and landing events.
- Compute per-tick strafe sync data.
- Expose current player velocity, strafe sync, and jump state via natives.
- Fire `GM_OnPlayerJumpLiftoff` and `GM_OnPlayerJumpLanded` for `gm_jumpstats` consumption.
- Detect Head Boost, Double Boost, Human Ladder triggers by monitoring relative player positions.
- Fire `GM_OnBoostDetected` when a boost interaction is validated.
- No database access. Pure runtime tick-level state.

**State managed in memory (per client, per tick):**

- Current velocity (x, y, z).
- Current position (x, y, z).
- Jump state (grounded, airborne, landing).
- Consecutive jump tick counter (for held-key autobhop detection).
- Strafe direction history (last N ticks).
- Sync percentage accumulator for current airtime.

**Fires events:**
- `GM_OnPlayerJumpLiftoff` — client, pre_speed, position.
- `GM_OnPlayerJumpLanded` — client, land_speed, position, airtime_ticks.
- `GM_OnBoostDetected` — client (boostee), booster_client, boost_type.

**Consumes events:**
- `GM_OnRoundStart` — reset per-client jump tracking.
- `GM_OnPlayerConnect` — initialize per-client movement state.
- `GM_OnPlayerDisconnect` — clear per-client movement state.

---

### 6.5 `gm_team` — Team System Module

**Binary:** `gm_team.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1 Hot
**Database tables owned:** None. Runtime state only.

**Responsibilities:**

- Maintain the 70%/30% Hider/Chaser ratio at all times.
- Manage the CT Queue (ordered list of clients waiting for Chaser role).
- Process `!team ct` and `!team t` requests (queued for next round).
- Execute auto-balance at round end if composition deviates beyond threshold.
- Enforce persistent team assignment across rounds.
- Restore team assignment for reconnecting players within 60-second reconnect window.
- Fire `GM_OnTeamAssigned` when a player is placed on a team.
- Expose team query natives to `gm_gameplay`, `gm_hud`, and `gm_menu`.

**State managed in memory:**

- Current team per client (`GMPlayerCache.current_team`).
- CT Queue: ordered array of `account_id` values.
- Reconnect cache: `account_id` → team mapping with expiry timestamps.
- CT decline count per client per session.

**Fires events:**
- `GM_OnTeamAssigned` — client, team (GMParticipantTeam), was_auto_balanced (bool).
- `GM_OnCTQueueUpdated` — queue length.

**Consumes events:**
- `GM_OnPlayerConnect` — initial team assignment.
- `GM_OnPlayerDisconnect` — remove from CT Queue, update reconnect cache.
- `GM_OnRoundEnd` — trigger auto-balance evaluation, process queued team change requests.

---

### 6.6 `gm_gameplay` — Round Rules Module

**Binary:** `gm_gameplay.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1 Hot
**Database tables owned:** `rounds`, `round_participants`

**Responsibilities:**

- Hook all CSS round events (`round_start`, `round_end`, `player_death`, `player_spawn`).
- Implement CT Freeze Time (5 seconds, locked).
- Implement the 5..4..3..2..1..GO countdown sequence.
- Manage FrostNade distribution to Hiders at round start (2 per Hider, T-only).
- Prevent CTs from acquiring or holding FrostNades (validate on pickup).
- Enforce Hider spawn zone exit timer (warn at 15s, slay at 20s post-round-start).
- Enforce Chaser spawn zone re-entry restriction (10s limit).
- Detect round win conditions and fire `GM_OnRoundEnd`.
- Insert a `rounds` record on round start; update it on round end.
- Batch-insert `round_participants` records at round end for all participants.
- Load per-map metadata from `.json` files and apply `default_round_time`.
- Manage the Training Mode round lifecycle (no stats, no XP, no team rules).

**State managed in memory:**

- `GMRoundState` singleton.
- Per-client: spawn zone timer, frostnades remaining, is_alive, time_alive.
- Map metadata (loaded at map change).
- Round-end participant snapshot (collected during round, written at round end).

**Fires events:**
- `GM_OnRoundStart` — round_id, map_name, round_type.
- `GM_OnCountdownGo` — round_id.
- `GM_OnRoundEnd` — round_id, winner_team.
- `GM_OnPlayerEliminated` — victim_id, attacker_id, round_id.
- `GM_OnPlayerSurvived` — account_id, round_id, time_alive_seconds.
- `GM_OnFrostNadeDetonated` — thrower_id, targets_hit, round_id.
- `GM_OnMapLoaded` — map_name, metadata (GMMapMetadata struct).

**Consumes events:**
- `GM_OnPlayerConnect` — initialize runtime client state.
- `GM_OnPlayerDisconnect` — handle mid-round disconnect.
- `GM_OnTrainingStart` — switch to training round mode.
- `GM_OnTrainingEnd` — return to standard round mode.

---

### 6.7 `gm_jumpstats` — JumpStats Module

**Binary:** `gm_jumpstats.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2 Warm
**Database tables owned:** `jumpstat_records`, `jumpstat_tick_data`

**Responsibilities:**

- Subscribe to `GM_OnPlayerJumpLiftoff` and `GM_OnPlayerJumpLanded` from `gm_movement`.
- Classify jump type from liftoff context (ground state, ladder state, pre-strafe velocity).
- Accumulate per-tick strafe count and sync data during airtime.
- Record position and velocity per tick to the `jumpstat_tick_data` buffer (for Phase 3).
- On landing: run all validation rules (velocity, trajectory, surface, scroll-jump integrity).
- Commit validated jumps to `jumpstat_records` asynchronously.
- Update `is_pb` flag on previous PB record; set `is_pb = 1` on new record if PB.
- Check if distance exceeds current server record; update `is_server_record` flags.
- Fire `GM_OnJumpRecorded` after database commit with full `GMJumpData` payload.
- Fire `GM_OnServerRecordBroken` if applicable (consumed by `gm_admin` for webhook).
- Serve leaderboard data queries from `gm_menu` via native.
- Suppress jump recording during Training Mode (`GMRoundState.round_type == Training`).
- Track training-mode jumps separately (in-memory only; not committed to main leaderboard).

**State managed in memory:**

- Per-client: jump tracking state (liftoff_position, tick buffer, strafe accumulator).
- Per-client: current PB per jump type (loaded at connect from DB).
- Current server record per jump type (loaded at plugin startup, updated on new SR).
- Training-mode jump buffer per client (not persisted).

**Fires events:**
- `GM_OnJumpRecorded` — account_id, GMJumpData, record_id.
- `GM_OnServerRecordBroken` — account_id, jump_type, new_distance, previous_distance, record_id.

**Consumes events:**
- `GM_OnPlayerJumpLiftoff` — begin jump tracking.
- `GM_OnPlayerJumpLanded` — finalize measurement, validate, commit.
- `GM_OnPlayerConnect` — load client's PBs per jump type.
- `GM_OnPlayerDisconnect` — clear client jump tracking state.
- `GM_OnCountdownGo` — reset per-round jump state.
- `GM_OnTrainingStart` / `GM_OnTrainingEnd` — toggle training mode flag.

---

### 6.8 `gm_xp` — XP and Level Module

**Binary:** `gm_xp.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2 Warm
**Database tables owned:** `xp_transactions`, `player_levels`

**Responsibilities:**

- Compute XP awards at round end per participant (survival bonus, elimination bonus, last survivor bonus, first blood bonus, round win bonus, participation floor).
- Validate that round type is Standard before awarding gameplay XP. Training Mode rounds award zero XP.
- Award JumpStat XP on `GM_OnJumpRecorded` (PB = 20 XP, SR = 150 XP).
- Award Achievement XP when `GM_OnAchievementUnlocked` fires.
- Insert an `xp_transactions` record for every XP award.
- Update `player_levels.total_xp`, `player_levels.current_level`, `player_levels.xp_at_current_level`, `player_levels.xp_to_next_level` atomically.
- Update `GMPlayerCache.total_xp`, `current_level`, `xp_to_next_level` in memory.
- Detect level-up and fire `GM_OnLevelUp`.
- Fire `GM_OnXPAwarded` after every XP transaction.
- Expose level data and XP history natives.

**State managed in memory:**

- Per-client: `total_xp`, `current_level`, `xp_at_current_level`, `xp_to_next_level` (mirrored from `GMPlayerCache`).

**Fires events:**
- `GM_OnXPAwarded` — account_id, amount, GMXPSourceType.
- `GM_OnLevelUp` — account_id, old_level, new_level.

**Consumes events:**
- `GM_OnRoundEnd` — compute and award per-participant gameplay XP.
- `GM_OnJumpRecorded` — award JumpStat XP.
- `GM_OnAchievementUnlocked` — award achievement XP.
- `GM_OnPlayerConnect` — load XP and level state (from `GMPlayerCache`, already populated by `gm_account`).

---

### 6.9 `gm_rank` — Ranking Module

**Binary:** `gm_rank.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2 Warm
**Database tables owned:** `rank_scores`, `rank_history`

**Responsibilities:**

- Compute Gameplay Rank score delta at round end for all participants (survival weight, elimination weight, round win bonus, early elimination penalty — weights from `gm_rank.cfg`).
- Update `rank_scores` for all participants asynchronously at round end.
- Compute Movement Rank score when a player's JumpStat PB changes (composite of 5 jump types weighted per spec).
- Recompute `server_position` for affected players after each score update.
- Determine rank tier from score thresholds (from `gm_rank.cfg`).
- Detect tier changes and fire `GM_OnRankChanged`.
- Assign or remove TOP 10 / TOP 3 tags dynamically after server position recompute.
- Fire `GM_OnTagAutoAssigned` when a player enters or exits the top 3/10.
- Write weekly `rank_history` snapshots (driven by a repeating timer, fires every Sunday 00:00 UTC).
- Expose rank data natives for `gm_menu` and `gm_hud`.
- Skip Gameplay Rank update for Training Mode rounds.

**State managed in memory:**

- Per-client: `gameplay_score`, `gameplay_tier`, `gameplay_position`, `movement_score`, `movement_tier` (mirrored from `GMPlayerCache`).
- Server position recompute is async; in-memory value may lag by one round.

**Fires events:**
- `GM_OnRankChanged` — account_id, GMRankAxis, old_tier, new_tier.
- `GM_OnTagAutoAssigned` — account_id, tag_id ("tag_top3" or "tag_top10"), is_assigned (bool).

**Consumes events:**
- `GM_OnRoundEnd` — trigger Gameplay Rank update.
- `GM_OnJumpRecorded` — trigger Movement Rank update (only on PB: `GMJumpData.is_pb == true`).
- `GM_OnPlayerConnect` — load rank state (from `GMPlayerCache`, already populated by `gm_account`).

---

### 6.10 `gm_achievements` — Achievement Module

**Binary:** `gm_achievements.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2 Warm
**Database tables owned:** `player_achievements`

**Responsibilities:**

- Load the full `achievement_catalog` into memory at plugin startup.
- Load each connected player's `player_achievements` records at connect time.
- Maintain per-client progress counters for all condition types (rounds survived, eliminations, jump distances, levels, etc.).
- Evaluate achievement conditions asynchronously at round end, on JumpStat commit, and on session events.
- On condition met: insert `player_achievements` record (or increment `times_completed` for repeatable achievements).
- Fire `GM_OnAchievementUnlocked` — consumed by `gm_xp` for XP reward and by `gm_cosmetics` for cosmetic reward.
- Display achievement unlock notification to the player via `GM_Lang_Format` + chat print.
- Server-wide announcement for Platinum tier achievement unlocks.

**State managed in memory:**

- Achievement catalog: full array of all achievement definitions.
- Per-client: owned achievement ID set (for unlock gate checks).
- Per-client: progress counters per tracked condition type.

**Fires events:**
- `GM_OnAchievementUnlocked` — account_id, achievement_id, tier (GMAchievementTier).

**Consumes events:**
- `GM_OnPlayerConnect` — load owned achievements and initialize progress counters.
- `GM_OnPlayerDisconnect` — clear client achievement state.
- `GM_OnRoundEnd` — evaluate survival/elimination/participation conditions.
- `GM_OnJumpRecorded` — evaluate movement conditions.
- `GM_OnLevelUp` — evaluate level-based conditions.
- `GM_OnRankChanged` — evaluate rank-based conditions.
- `GM_OnFrostNadeDetonated` — evaluate FrostNade deployment conditions.

---

### 6.11 `gm_cosmetics` — Cosmetics Module

**Binary:** `gm_cosmetics.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2
**Database tables owned:** `player_cosmetics`, `cosmetic_catalog`

**Responsibilities:**

- Load the full `cosmetic_catalog` into memory at plugin startup.
- Load each connected player's owned cosmetics (`player_cosmetics`) at connect.
- Apply the equipped knife skin via CSS model override hooks on player spawn.
- Apply the equipped trail via particle effect spawn during movement (hooks `OnPlayerRunCmd` for trail position update).
- Apply the equipped title to the player's name display in HUD and scoreboard.
- Apply the `equipped_title` from `player_settings`.
- Apply auto-rank tags (TOP 10 / TOP 3) dynamically — sourced from `GM_OnTagAutoAssigned` event.
- Apply permanent admin-granted tags (FOUNDER, BETA) from `player_cosmetics` ownership records.
- Process cosmetic grant requests from `gm_achievements` (on `GM_OnAchievementUnlocked` with cosmetic reward).
- Process cosmetic grant requests from `gm_xp` (on `GM_OnLevelUp` for level rewards).
- Process admin-issued cosmetic grants via `GM_Cosmetics_GrantCosmetic` native.
- Update `player_settings.equipped_knife`, `equipped_trail`, `equipped_title` on equip changes.
- Expose ownership check and equip natives.

**State managed in memory:**

- Cosmetic catalog: full array of all cosmetic definitions.
- Per-client: owned cosmetic ID set.
- Per-client: currently equipped knife, trail, title, active tag.

**Fires events:**
- `GM_OnCosmeticGranted` — account_id, cosmetic_id, source (GMCosmeticAcquisition).
- `GM_OnCosmeticEquipped` — account_id, cosmetic_id, category (GMCosmeticCategory).

**Consumes events:**
- `GM_OnPlayerConnect` — load owned cosmetics, apply equipped items.
- `GM_OnPlayerDisconnect` — clear client cosmetic state.
- `GM_OnAchievementUnlocked` — check for cosmetic reward; grant if applicable.
- `GM_OnLevelUp` — check for level reward cosmetic; grant if applicable.
- `GM_OnTagAutoAssigned` — apply or remove TOP 10 / TOP 3 tag display.

---

### 6.12 `gm_training` — Training Mode Module

**Binary:** `gm_training.smx`
**Layer:** 2 — Feature Domain
**Tier:** T2
**Database tables owned:** None (training stats are in-memory only).

**Responsibilities:**

- Handle `!training` command: if solo player → activate immediately; if multiple players → initiate server vote.
- Manage the vote lifecycle: broadcast vote prompt, collect votes, evaluate result, apply cooldown on failure.
- Admin override: activate training mode without vote on `!a training start`.
- Fire `GM_OnTrainingStart` and `GM_OnTrainingEnd` to notify all other modules.
- Implement `!cp` (checkpoint save) and `!tp` (checkpoint teleport) per client.
- Implement `!play` to end training mode and return to live HNS.
- Manage Training Mode sub-modes (Free Movement, JumpStats Practice, Strafing Workshop, Guided Challenges).
- Evaluate Guided Challenge pass/fail conditions.
- Award challenge completion XP via `GM_XP_AwardXP` native (one-time per account per challenge).
- Track challenge completion state per client.
- Maintain per-client checkpoint position.

**State managed in memory:**

- Training mode active flag.
- Vote state: in-progress, vote counts, expiry timer.
- Per-client: checkpoint position (x, y, z, angle).
- Per-client: active training sub-mode.
- Per-client: challenge completion flags (for in-session deduplication; DB is authoritative).

**Fires events:**
- `GM_OnTrainingVoteStart` — initiator_id.
- `GM_OnTrainingStart` — (no payload).
- `GM_OnTrainingEnd` — (no payload).
- `GM_OnCheckpointSaved` — client, position.
- `GM_OnChallengeCompleted` — account_id, challenge_id.

**Consumes events:**
- `GM_OnPlayerConnect` — load challenge completion state.
- `GM_OnPlayerDisconnect` — clear checkpoint and vote state.
- `GM_OnRoundEnd` — training mode does not fire this; `gm_gameplay` manages round lifecycle.

---

### 6.13 `gm_hud` — HUD Overlay Module

**Binary:** `gm_hud.smx`
**Layer:** 3 — Presentation
**Tier:** T1 Hot
**Database tables owned:** None. Presentation only.

**Responsibilities:**

- Render the JumpStats HUD overlay (last jump data: type, distance, pre_speed, sync_pct, strafes).
- Render the Speed HUD overlay (current horizontal velocity).
- Render the Sync HUD overlay (real-time sync % for current airtime).
- Render the Strafe HUD overlay (per-strafe velocity gain/loss).
- Render the countdown sequence (5..4..3..2..1..GO!) as a fullscreen HUD panel.
- Render the round timer HUD.
- Respect per-client HUD preferences from `GMPlayerCache`.
- Update HUD at configurable tick rate (not every frame — configurable interval, default 0.1s).
- All HUD values are read from `GMPlayerCache` and `gm_movement` / `gm_jumpstats` natives. No DB access.

**State managed in memory:**

- Per-client: last jump HUD data (from `GM_OnJumpRecorded`).
- HUD update timer handle.

**Fires events:** None.

**Consumes events:**
- `GM_OnJumpRecorded` — update last jump HUD cache.
- `GM_OnCountdownGo` — end countdown display; begin round timer.
- `GM_OnRoundStart` — begin countdown display.
- `GM_OnRoundEnd` — clear round timer.
- `GM_OnPlayerConnect` — initialize client HUD state.
- `GM_OnPlayerDisconnect` — clear client HUD state.

---

### 6.14 `gm_antiexploit` — Anti-Exploit Module

**Binary:** `gm_antiexploit.smx`
**Layer:** 2 — Feature Domain
**Tier:** T1 Hot
**Database tables owned:** None.

**Responsibilities:**

- Subscribe to per-tick position and velocity data from `gm_movement`.
- Validate per-tick velocity delta against the physics-possible maximum (based on `sv_airaccelerate` and `sv_gravity`).
- Detect teleports: position delta exceeding theoretical per-tick maximum.
- Detect scroll-jump integrity violations: jump input rate exceeding human-possible threshold.
- Detect FrostNade count manipulation: validate via server-authoritative `frostnades_remaining`.
- On anomaly detection: log to `gm_antiexploit.log`, flag the round's stats for discard, optionally fire `GM_OnExploitDetected`.
- Provide `!bug` command handling: capture player position and round context, insert `bug_reports` record.

**State managed in memory:**

- Per-client: position history (last N ticks).
- Per-client: velocity history (last N ticks).
- Per-client: jump input rate (ticks since last jump).
- Per-client: anomaly flag (if flagged, round stats are discarded).

**Fires events:**
- `GM_OnExploitDetected` — account_id, exploit_type (string), severity (int), round_id.

**Consumes events:**
- `GM_OnPlayerJumpLiftoff` — check velocity at liftoff.
- `GM_OnPlayerJumpLanded` — validate trajectory.
- `GM_OnPlayerConnect` — initialize client tracking state.
- `GM_OnPlayerDisconnect` — clear client tracking state.
- `GM_OnRoundStart` — reset per-round anomaly flags.

---

### 6.15 `gm_menu` — Menu Module

**Binary:** `gm_menu.smx`
**Layer:** 3 — Presentation
**Tier:** T2
**Database tables owned:** None. Stateless presentation.

**Responsibilities:**

- Render all player-facing menus defined in GM_HNS_MASTER_SPEC_V1_FINAL.md Section 13.
- Route menu item selections to the appropriate L2 module native or command.
- Maintain no game state. All data is fetched from L2 natives at menu open time.
- All strings rendered through `GM_Lang_Format`.
- Handle paginated leaderboard display (page state held in per-client menu handle).

**Menus owned:**

| Menu ID | Trigger | Data Source |
|---|---|---|
| Main Menu | `!menu` | Static |
| Profile Menu | `!profile` | `gm_account`, `gm_xp`, `gm_rank`, `gm_jumpstats` natives |
| Statistics Menu | `!stats` | `gm_account` natives |
| Rankings Menu | `!rank` / `!lb` | `gm_rank`, `gm_jumpstats` natives |
| JumpStats LB | `!lb` | `gm_jumpstats` native (DB query) |
| Personal Bests | `!pbs` | `gm_jumpstats` native |
| Achievements | `!achievements` | `gm_achievements` native |
| Cosmetics Menu | `!cosmetics` | `gm_cosmetics` natives |
| Training Menu | `!training` | `gm_training` native |
| Settings Menu | `!settings` | `GMPlayerCache` via `gm_account` native |
| HUD Sub-Menu | `!hud` | `GMPlayerCache` via `gm_account` native |

**Fires events:** None.

**Consumes events:**
- `GM_OnPlayerDisconnect` — close any open menu handles for the disconnecting client.

---

### 6.16 `gm_commands` — Command Router Module

**Binary:** `gm_commands.smx`
**Layer:** 3 — Presentation
**Tier:** T2
**Database tables owned:** None.

**Responsibilities:**

- Register all player-facing commands (`!menu`, `!js`, `!lb`, `!cp`, `!tp`, `!play`, `!training`, `!team ct`, `!team t`, `!queue`, `!cosmetics`, `!knife`, `!trail`, `!title`, `!profile`, `!stats`, `!rank`, `!level`, `!achievements`, `!pbs`, `!sr`, `!settings`, `!lang`, `!hud`, `!rules`, `!discord`, `!bug`, `!help`).
- Map each command to the appropriate module native, menu open call, or event.
- Validate command arguments.
- Check command permission level against `GMPlayerCache.admin_level`.
- Return localized error messages for invalid arguments or permission failures.
- Commands do not contain logic — they are pure routing.

**Command → Target Map (Representative)**

| Command | Routes To |
|---|---|
| `!menu` | `gm_menu` — open main menu |
| `!cp` | `gm_training` — native `GM_Training_SaveCheckpoint` |
| `!tp` | `gm_training` — native `GM_Training_TeleportToCheckpoint` |
| `!play` | `gm_training` — native `GM_Training_EndTraining` |
| `!training` | `gm_training` — native `GM_Training_RequestStart` |
| `!team ct` | `gm_team` — native `GM_Team_JoinCTQueue` |
| `!team t` | `gm_team` — native `GM_Team_RequestHider` |
| `!queue` | `gm_menu` — open CT queue display |
| `!lb` | `gm_menu` — open JumpStats leaderboard menu |
| `!pbs` | `gm_menu` — open personal bests panel |
| `!sr` | `gm_menu` — open server records panel |
| `!profile` | `gm_menu` — open profile menu |
| `!stats` | `gm_menu` — open statistics menu |
| `!rank` | `gm_menu` — open rank display |
| `!level` | `gm_menu` — open level/XP display |
| `!achievements` | `gm_menu` — open achievement browser |
| `!cosmetics` | `gm_menu` — open cosmetics menu |
| `!knife` | `gm_menu` — open knife sub-menu |
| `!trail` | `gm_menu` — open trail sub-menu |
| `!title` | `gm_menu` — open title sub-menu |
| `!settings` | `gm_menu` — open settings menu |
| `!lang` | `gm_menu` — open language sub-menu |
| `!hud` | `gm_menu` — open HUD settings sub-menu |
| `!rules` | `gm_menu` — open rules panel |
| `!discord` | Chat print of Discord URL (localized) |
| `!bug` | `gm_antiexploit` — native `GM_AntiExploit_SubmitBugReport` |
| `!help` | `gm_menu` — open help panel |

**Fires events:** None.

**Consumes events:** None.

---

### 6.17 `gm_admin` — Admin Module

**Binary:** `gm_admin.smx`
**Layer:** 3 — Presentation
**Tier:** T3 Cold
**Database tables owned:** `admin_log`

**Responsibilities:**

- Register all admin commands (`!a kick`, `!a ban`, `!a unban`, `!a mute`, `!a slay`, `!a slayteam`, `!a forceseeker`, `!a setregion`, `!a givexp`, `!a givecosmetic`, `!a granttag`, `!a invalidatejs`, `!a training start/stop`, `!a auditlog`, `!a reloadconfig`, `!a reloadmap`, `!a dbstatus`, `!a reloadplugin`).
- Validate admin level for every command (`GMAdminLevel.Admin` or `GMAdminLevel.Owner`).
- Execute commands by calling target module natives.
- Write an `admin_log` record for every executed admin action.
- Manage outbound Discord webhooks (ban notifications, CRITICAL error alerts).
- Implement `!a dbstatus` by calling `GM_DB_Status()` and formatting the result.
- Implement `!a reloadplugin` by issuing SourceMod plugin reload commands.

**State managed in memory:**

- Webhook retry queue (in-memory list of pending webhook payloads with attempt counts).

**Fires events:** None.

**Consumes events:**
- `GM_OnServerRecordBroken` — fire Discord webhook to `#records` channel.
- `GM_OnExploitDetected` — log to admin log; escalate CRITICAL level to Discord webhook.

---

## 7. SHARED API REFERENCE

### 7.1 Database Pool API (`GM_DB_*`)

All database operations in all modules go through this API. Direct `SQL_Query` or `db.Query` calls on a module-maintained handle are prohibited.

| Function Signature | Description | Returns |
|---|---|---|
| `GM_DB_QueryAsync(const char[] query, SQLQueryCallback callback, any data, float timeout)` | Fire an async SQL query. Callback fires on completion or timeout. | void |
| `GM_DB_TransactionAsync(Transaction txn, SQLTxnSuccessCallback onSuccess, SQLTxnFailureCallback onFailure, any data, float timeout)` | Fire an async multi-query transaction. | void |
| `GM_DB_Escape(const char[] input, char[] output, int maxlen)` | SQL-safe string escaping. Must be called for all string inputs in query strings. | void |
| `GM_DB_Status()` | Returns current pool health. | GMDBStatus |
| `GM_DB_GetPoolSize()` | Returns number of active connections in pool. | int |

**Query callback contract:**

All `SQLQueryCallback` implementations must:
1. Check for `db == null` (connection failure) before accessing the result.
2. Check for `results == null` before iterating.
3. Never block the callback for more than 1 frame of processing.
4. Log errors at WARN level minimum.

---

### 7.2 Language API (`GM_Lang_*`)

| Function Signature | Description | Returns |
|---|---|---|
| `GM_Lang_Format(int client, const char[] key, char[] output, int maxlen, any ...)` | Format a localized string for the given client. Resolves client's locale from `GMPlayerCache`. Fallback: `id` → raw key. | void |
| `GM_Lang_GetLocale(int client)` | Returns the client's locale code string ('id' or 'en'). | char[] |
| `GM_Lang_PrintToChat(int client, const char[] key, any ...)` | Format and print directly to client chat. Handles `\x01` prefix for color reset. | void |
| `GM_Lang_PrintToAll(const char[] key, any ...)` | Format and print to all clients. Each client receives in their own locale. | void |

---

### 7.3 Configuration API (`GM_Config_*`)

| Function Signature | Description | Returns |
|---|---|---|
| `GM_Config_GetInt(const char[] key, int default_val)` | Read integer config value. Returns `default_val` if key not found. | int |
| `GM_Config_GetFloat(const char[] key, float default_val)` | Read float config value. | float |
| `GM_Config_GetString(const char[] key, char[] output, int maxlen, const char[] default_val)` | Read string config value. | void |
| `GM_Config_Reload()` | Hot-reload all config files. Called by `!a reloadconfig`. | void |

---

### 7.4 Player Cache API (`GM_Cache_*`)

| Function Signature | Description | Returns |
|---|---|---|
| `GM_Cache_Get(int client)` | Returns a reference to the `GMPlayerCache` entry for this client. | GMPlayerCache |
| `GM_Cache_IsLoaded(int client)` | Returns true if the connect-time DB queries for this client are complete. | bool |
| `GM_Cache_GetAccountId(int client)` | Returns `account_id` for this client. Returns -1 if not loaded. | int |

All modules check `GM_Cache_IsLoaded(client)` before operating on player data.

---

### 7.5 Logging API (`GM_Log_*`)

| Function Signature | Description |
|---|---|
| `GM_Log_Debug(const char[] module, const char[] msg, any ...)` | Debug-level log. Written only if debug mode enabled in `gm_config.cfg`. |
| `GM_Log_Info(const char[] module, const char[] msg, any ...)` | Info-level log. Standard operational messages. |
| `GM_Log_Warn(const char[] module, const char[] msg, any ...)` | Warning-level log. Non-fatal anomalies. |
| `GM_Log_Error(const char[] module, const char[] msg, any ...)` | Error-level log. Recoverable failures. |
| `GM_Log_Critical(const char[] module, const char[] msg, any ...)` | Critical-level log. Fires Discord webhook to `#admin-alerts`. |

Each module passes its own module name string as the first argument. The log API writes to `logs/gm_<module>.log`.

---

## 8. FORWARD AND EVENT DEFINITIONS

All GM forwards are declared in `gm_events.inc`. Modules subscribe by implementing the forward. No module may fire a forward not declared in this file.

### 8.1 Naming Convention

All GM forwards follow the pattern:

```
GM_On<Subject><Verb>(<parameters>)
```

### 8.2 Complete Forward List — V1.0

#### Identity and Session Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnPlayerConnect` | `gm_account` | `int client, int account_id` | Fires after all connect-time DB queries complete. Cache is fully populated. |
| `GM_OnPlayerDisconnect` | `gm_account` | `int client, int account_id, int session_id` | Fires before cache is cleared. Final state is still readable. |

#### Round Lifecycle Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnRoundStart` | `gm_gameplay` | `int round_id, const char[] map_name, GMRoundType round_type` | Fires at round start (after CSS `round_start` event). |
| `GM_OnCountdownGo` | `gm_gameplay` | `int round_id` | Fires when the countdown reaches GO! CT freeze lifted. |
| `GM_OnRoundEnd` | `gm_gameplay` | `int round_id, GMWinnerTeam winner` | Fires at round end (after CSS `round_end` event). |
| `GM_OnMapLoaded` | `gm_gameplay` | `const char[] map_name` | Fires after map metadata is loaded. |

#### Player Game Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnPlayerEliminated` | `gm_gameplay` | `int victim_client, int attacker_client, int round_id` | Fires on player death during active round. Parameters use client slot, not account_id. |
| `GM_OnPlayerSurvived` | `gm_gameplay` | `int client, int account_id, int round_id, int time_alive_seconds` | Fires for each Hider who survives to round end. |
| `GM_OnFrostNadeDetonated` | `gm_gameplay` | `int thrower_client, int targets_hit, int round_id` | Fires when a FrostNade detonates and hits at least one Chaser. |

#### Team Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnTeamAssigned` | `gm_team` | `int client, GMParticipantTeam team, bool was_auto_balanced` | Fires when a player is assigned to a team. |
| `GM_OnCTQueueUpdated` | `gm_team` | `int queue_length` | Fires when CT queue state changes. |

#### Movement Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnPlayerJumpLiftoff` | `gm_movement` | `int client, float pre_speed, float pos[3]` | Fires on jump liftoff detection. |
| `GM_OnPlayerJumpLanded` | `gm_movement` | `int client, float land_speed, float pos[3], int airtime_ticks` | Fires on landing detection. |
| `GM_OnBoostDetected` | `gm_movement` | `int boostee_client, int booster_client, const char[] boost_type` | Fires when a boost interaction is validated. `boost_type`: "head", "double", "ladder". |

#### JumpStats Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnJumpRecorded` | `gm_jumpstats` | `int client, int account_id, GMJumpData jump, int record_id` | Fires after a validated jump is committed to DB. |
| `GM_OnServerRecordBroken` | `gm_jumpstats` | `int account_id, GMJumpType jump_type, float new_distance, float old_distance, int record_id` | Fires when a new server record is set. |

#### Progression Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnXPAwarded` | `gm_xp` | `int account_id, int amount, GMXPSourceType source_type` | Fires after every XP transaction. |
| `GM_OnLevelUp` | `gm_xp` | `int client, int account_id, int old_level, int new_level` | Fires when a player levels up. |
| `GM_OnRankChanged` | `gm_rank` | `int client, int account_id, GMRankAxis axis, GMRankTier old_tier, GMRankTier new_tier` | Fires when a player's rank tier changes. |
| `GM_OnTagAutoAssigned` | `gm_rank` | `int account_id, const char[] tag_id, bool is_assigned` | Fires when TOP 10 / TOP 3 tag is applied or removed. |
| `GM_OnAchievementUnlocked` | `gm_achievements` | `int client, int account_id, const char[] achievement_id, GMAchievementTier tier` | Fires when an achievement is completed. |

#### Cosmetic Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnCosmeticGranted` | `gm_cosmetics` | `int account_id, const char[] cosmetic_id, GMCosmeticAcquisition source` | Fires when a cosmetic is added to a player's owned inventory. |
| `GM_OnCosmeticEquipped` | `gm_cosmetics` | `int client, int account_id, const char[] cosmetic_id, GMCosmeticCategory category` | Fires when a player equips a cosmetic. |

#### Training Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnTrainingVoteStart` | `gm_training` | `int initiator_client` | Fires when a Training Mode vote is initiated. |
| `GM_OnTrainingStart` | `gm_training` | (none) | Fires when Training Mode activates. |
| `GM_OnTrainingEnd` | `gm_training` | (none) | Fires when Training Mode ends. |
| `GM_OnCheckpointSaved` | `gm_training` | `int client, float pos[3]` | Fires when a player saves a checkpoint. |
| `GM_OnChallengeCompleted` | `gm_training` | `int client, int account_id, const char[] challenge_id` | Fires when a guided challenge is completed. |

#### Anti-Exploit Events

| Forward | Fired By | Parameters | Description |
|---|---|---|---|
| `GM_OnExploitDetected` | `gm_antiexploit` | `int client, int account_id, const char[] exploit_type, int severity, int round_id` | Fires when a server-side anomaly is detected. |

---

## 9. NATIVE DEFINITIONS

All GM natives are declared in `gm_natives.inc`. Natives are synchronous cross-module function calls that return a value. They are used only when an event cannot satisfy the need (i.e., when a return value is required in the same frame).

### 9.1 Naming Convention

```
GM_<ModuleShortName>_<FunctionName>(<parameters>)
```

### 9.2 Account Natives (`gm_account`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Account_GetAccountId` | `int client` | `int` | Returns `account_id` for client. -1 if not loaded. |
| `GM_Account_GetAdminLevel` | `int client` | `GMAdminLevel` | Returns admin level. |
| `GM_Account_GetRegionCode` | `int client, char[] output, int maxlen` | `void` | Writes region code string. |
| `GM_Account_GetUsername` | `int client, char[] output, int maxlen` | `void` | Writes current username. |
| `GM_Account_IsLoaded` | `int client` | `bool` | True if connect-time queries are complete. |
| `GM_Account_IsBanned` | `int account_id` | `bool` | Check ban status (DB query, async — for admin use). |
| `GM_Account_BanPlayer` | `int account_id, int duration_minutes, const char[] reason, int admin_account_id` | `void` | Apply a ban. Writes `accounts` table. |
| `GM_Account_UnbanPlayer` | `int account_id, int admin_account_id` | `void` | Remove a ban. |
| `GM_Account_SetPreference` | `int client, const char[] key, const char[] value` | `void` | Update a player setting in DB and cache. |
| `GM_Account_GetTotalPlaytime` | `int client` | `int` | Returns total playtime in seconds from cache. |

### 9.3 Region Natives (`gm_region`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Region_GetClientRegion` | `int client, char[] output, int maxlen` | `void` | Writes client's region code. |
| `GM_Region_SetClientRegion` | `int account_id, const char[] region_code` | `void` | Override region (admin use). Writes DB and cache. |

### 9.4 Movement Natives (`gm_movement`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Movement_GetClientVelocity` | `int client, float vel[3]` | `void` | Returns client's current velocity vector. |
| `GM_Movement_GetClientSpeed` | `int client` | `float` | Returns current horizontal speed in HU/s. |
| `GM_Movement_GetClientJumpState` | `int client` | `int` | Returns jump state enum (grounded/airborne/landing). |
| `GM_Movement_GetCurrentSyncPct` | `int client` | `float` | Returns sync % for current ongoing airtime. |
| `GM_Movement_IsGrounded` | `int client` | `bool` | True if client is on a valid ground surface. |

### 9.5 Team Natives (`gm_team`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Team_GetClientTeam` | `int client` | `GMParticipantTeam` | Returns client's current HNS team. |
| `GM_Team_GetHiderCount` | (none) | `int` | Live count of Hiders. |
| `GM_Team_GetChaserCount` | (none) | `int` | Live count of Chasers. |
| `GM_Team_GetCTQueuePosition` | `int client` | `int` | Returns client's position in CT Queue. 0 if not queued. |
| `GM_Team_JoinCTQueue` | `int client` | `void` | Add client to CT Queue. |
| `GM_Team_RequestHider` | `int client` | `void` | Request Hider assignment. |
| `GM_Team_ForceChaser` | `int client, int admin_account_id` | `void` | Admin force to Chaser next round. |

### 9.6 Gameplay Natives (`gm_gameplay`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Gameplay_GetRoundState` | (none) | `GMRoundState` | Returns current round state struct. |
| `GM_Gameplay_GetRoundId` | (none) | `int` | Returns current `round_id`. |
| `GM_Gameplay_IsTrainingMode` | (none) | `bool` | True if Training Mode is active. |
| `GM_Gameplay_GetClientFrostnadesRemaining` | `int client` | `int` | Returns FrostNades remaining for this client this round. |
| `GM_Gameplay_SlayClient` | `int client, int admin_account_id` | `void` | Admin slay. |
| `GM_Gameplay_SlayTeam` | `GMParticipantTeam team, int admin_account_id` | `void` | Admin slay team. |

### 9.7 JumpStats Natives (`gm_jumpstats`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_JumpStats_GetClientPB` | `int client, GMJumpType jump_type` | `float` | Returns client's PB distance for a jump type. 0.0 if none. |
| `GM_JumpStats_GetServerRecord` | `GMJumpType jump_type` | `float` | Returns current server record distance. |
| `GM_JumpStats_GetServerRecordHolder` | `GMJumpType jump_type` | `int` | Returns `account_id` of current SR holder. |
| `GM_JumpStats_GetLeaderboard` | `GMJumpType jump_type, int page, int page_size, SQLQueryCallback callback` | `void` | Async leaderboard query. Result delivered via callback. |
| `GM_JumpStats_InvalidateRecord` | `int record_id, int admin_account_id` | `void` | Admin invalidation. Writes `is_valid = 0`. |
| `GM_JumpStats_GetLastJump` | `int client, GMJumpData result` | `bool` | Returns last validated jump data for client. False if none this session. |

### 9.8 XP Natives (`gm_xp`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_XP_GetClientXP` | `int client` | `int` | Returns client's `total_xp` from cache. |
| `GM_XP_GetClientLevel` | `int client` | `int` | Returns client's `current_level` from cache. |
| `GM_XP_GetXPToNextLevel` | `int client` | `int` | Returns `xp_to_next_level` from cache. |
| `GM_XP_AwardXP` | `int client, int amount, GMXPSourceType source, const char[] reference` | `void` | Award XP. Writes to DB; updates cache; fires `GM_OnXPAwarded`. |
| `GM_XP_AdminGiveXP` | `int account_id, int amount, int admin_account_id` | `void` | Admin XP grant. Bypasses round-type check. |

### 9.9 Rank Natives (`gm_rank`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Rank_GetClientScore` | `int client, GMRankAxis axis` | `int` | Returns client's rank score on an axis from cache. |
| `GM_Rank_GetClientTier` | `int client, GMRankAxis axis` | `GMRankTier` | Returns client's rank tier from cache. |
| `GM_Rank_GetClientPosition` | `int client, GMRankAxis axis` | `int` | Returns client's server position from cache. |
| `GM_Rank_GetLeaderboard` | `GMRankAxis axis, int page, int page_size, SQLQueryCallback callback` | `void` | Async rank leaderboard query. |
| `GM_Rank_HasAutoTag` | `int client, const char[] tag_id` | `bool` | True if client currently has the specified auto-rank tag active. |

### 9.10 Achievement Natives (`gm_achievements`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Achievement_HasEarned` | `int client, const char[] achievement_id` | `bool` | True if client has earned this achievement. |
| `GM_Achievement_GetClientList` | `int client, ArrayList results` | `void` | Populates ArrayList with earned achievement IDs. |
| `GM_Achievement_GetCatalogSize` | (none) | `int` | Returns total achievement count. |

### 9.11 Cosmetics Natives (`gm_cosmetics`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Cosmetics_Owns` | `int client, const char[] cosmetic_id` | `bool` | True if client owns this cosmetic. |
| `GM_Cosmetics_Grant` | `int account_id, const char[] cosmetic_id, GMCosmeticAcquisition source, int admin_account_id` | `void` | Grant a cosmetic. Writes DB; fires event. |
| `GM_Cosmetics_Equip` | `int client, const char[] cosmetic_id` | `bool` | Equip a cosmetic. Returns false if not owned. |
| `GM_Cosmetics_GetEquipped` | `int client, GMCosmeticCategory category, char[] output, int maxlen` | `void` | Returns ID of equipped cosmetic in category. |
| `GM_Cosmetics_GetActiveTag` | `int client, char[] output, int maxlen` | `void` | Returns the currently displayed tag ID (auto or permanent). |

### 9.12 Training Natives (`gm_training`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_Training_IsActive` | (none) | `bool` | True if Training Mode is currently active. |
| `GM_Training_RequestStart` | `int client` | `void` | Client requests training (solo activate or vote). |
| `GM_Training_EndTraining` | `int client` | `void` | End Training Mode and return to live. |
| `GM_Training_SaveCheckpoint` | `int client` | `void` | Save checkpoint at client's current position. |
| `GM_Training_TeleportToCheckpoint` | `int client` | `bool` | Teleport client to checkpoint. Returns false if none saved. |

### 9.13 Anti-Exploit Natives (`gm_antiexploit`)

| Native | Parameters | Returns | Description |
|---|---|---|---|
| `GM_AntiExploit_IsClientFlagged` | `int client` | `bool` | True if client has a pending anomaly flag this round. |
| `GM_AntiExploit_SubmitBugReport` | `int client, const char[] description` | `void` | Insert a `bug_reports` record with client position + round context. |

---

## 10. DATABASE OWNERSHIP MAP

Each table has exactly one **owner module** — the only module permitted to INSERT, UPDATE, or DELETE rows in that table. Other modules may SELECT from tables they do not own only via documented cross-module read patterns.

| Table | Owner Module | Cross-Module Reads Permitted By |
|---|---|---|
| `accounts` | `gm_account` | All modules (via `GM_Cache_Get` or `GM_Account_*` natives; not direct SELECT in non-owner modules) |
| `player_stats` | `gm_account` | `gm_menu` (profile display), `gm_achievements` (condition eval) |
| `player_levels` | `gm_xp` | `gm_menu`, `gm_achievements`, `gm_cosmetics` (level reward check) |
| `player_settings` | `gm_account` | All modules read via `GMPlayerCache`; `gm_cosmetics` writes equipped fields |
| `sessions` | `gm_account` | `gm_admin` (admin audit) |
| `servers` | `gm_admin` | All modules (read-only; loaded at startup) |
| `rounds` | `gm_gameplay` | `gm_admin`, `gm_antiexploit` |
| `round_participants` | `gm_gameplay` | `gm_admin`, `gm_achievements` |
| `xp_transactions` | `gm_xp` | `gm_admin` (audit log view) |
| `jumpstat_records` | `gm_jumpstats` | `gm_menu`, `gm_rank` (movement score calc), `gm_achievements` |
| `jumpstat_tick_data` | `gm_jumpstats` | None in V1.0 (Phase 3 replay reader TBD) |
| `achievement_catalog` | `gm_achievements` | All modules (read-only static catalog; loaded at startup) |
| `player_achievements` | `gm_achievements` | `gm_menu`, `gm_cosmetics` |
| `cosmetic_catalog` | `gm_cosmetics` | All modules (read-only static catalog; loaded at startup) |
| `player_cosmetics` | `gm_cosmetics` | `gm_menu`, `gm_achievements` |
| `rank_scores` | `gm_rank` | `gm_menu`, `gm_hud` (via native) |
| `rank_history` | `gm_rank` | `gm_admin` |
| `admin_log` | `gm_admin` | `gm_admin` only |
| `bug_reports` | `gm_antiexploit` | `gm_admin` |
| `account_merge_log` | Reserved | Phase 2 |
| `linked_accounts` | Reserved | Phase 2 |
| `friends` | Reserved | Phase 2 |

**Cross-module SELECT rule:** A non-owner module that needs data from another module's table must:
1. First check if the data is available via a native (preferred).
2. If not available via native, issue a direct SELECT — with no INSERT/UPDATE/DELETE — and document the cross-read in this table.
3. Never issue a write to a table it does not own.

---

## 11. INTER-PLUGIN COMMUNICATION PROTOCOL

### 11.1 Channel Priority

| Channel | Use Case | Synchronous? | Return Value? |
|---|---|---|---|
| GM Event Bus (Forwards) | Notifying multiple modules of a state change | No (fire-and-forget) | No |
| GM Natives | Querying a module's in-memory state | Yes | Yes |
| Shared Database | Persisted data exchange between sessions | No (async) | Via callback |

### 11.2 Event Bus Rules

- Events fire after the triggering action is complete. `GM_OnRoundEnd` fires after `gm_gameplay` has finalized the round state and begun its DB writes.
- Events are not cancellable. They are notification-only. A module cannot prevent an event from having already occurred.
- Event handlers must complete within the same frame. No blocking operations (file I/O, synchronous DB calls) inside an event handler.
- Event handlers must not fire other events directly. If follow-on events are needed, they are scheduled via a 0.0-second timer or are triggered by the follow-on module's own async callback.

### 11.3 Native Rules

- Natives are read-biased. They primarily read in-memory cache (`GMPlayerCache`) and return computed values.
- Natives that write to the database (`GM_XP_AwardXP`, `GM_Cosmetics_Grant`, `GM_Account_BanPlayer`) fire their own events after the DB write completes — not synchronously.
- A native must never call another module's native within its own implementation (circular dependency risk). If a native needs data from another module, it reads from `GMPlayerCache` only.
- Natives must not block the game thread. If a native's work is async, it accepts a callback parameter.

### 11.4 Communication Matrix

The following matrix documents which modules call which other modules' APIs. `E` = consumes event, `N` = calls native, `—` = no direct communication.

| Caller → | account | region | movement | team | gameplay | jumpstats | xp | rank | achievements | cosmetics | training | hud | antiexploit | menu | commands | admin |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **account** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **region** | N | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **movement** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **team** | N | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **gameplay** | N | — | N | N | — | — | — | — | — | — | — | — | — | — | — | — |
| **jumpstats** | N | — | N (E) | — | N | — | — | — | — | — | — | — | N | — | — | — |
| **xp** | N | — | — | — | N | N (E) | — | — | — | — | — | — | — | — | — | — |
| **rank** | N | — | — | — | N | N (E) | N (E) | — | — | — | — | — | — | — | — | — |
| **achievements** | N | — | — | — | N (E) | N (E) | N (E) | N (E) | — | — | — | — | — | — | — | — |
| **cosmetics** | N | — | — | — | — | — | — | N (E) | N (E) | — | — | — | — | — | — | — |
| **training** | N | — | — | — | N | N | N | — | — | — | — | — | — | — | — | — |
| **hud** | N | — | N | N | N | N | N | N | — | N | N | — | — | — | — | — |
| **antiexploit** | N | — | N (E) | — | N | — | — | — | — | — | — | — | — | — | — | — |
| **menu** | N | — | — | N | N | N | N | N | N | N | N | — | — | — | — | — |
| **commands** | N | — | — | N | — | N | — | — | — | N | N | — | N | N (routes) | — | — |
| **admin** | N | N | — | N | N | N | N | — | — | N | N | — | — | — | — | — |

---

## 12. CONFIGURATION ARCHITECTURE

### 12.1 Configuration File Ownership

| File | Owner Module | Hot-Reloadable | Description |
|---|---|---|---|
| `gm_config.cfg` | `gm_core` | Yes | Master server config. DB credentials, webhook URLs, debug mode, server_id. |
| `gm_gameplay.cfg` | `gm_gameplay` | Yes | Round timer, CT freeze duration, spawn grace timers, FrostNade config. |
| `gm_team.cfg` | `gm_team` | Yes | Hider/Chaser ratio, auto-balance threshold, CT queue settings. |
| `gm_jumpstats.cfg` | `gm_jumpstats` | Yes | Validation thresholds per jump type, scroll-jump integrity threshold, tick recording toggle. |
| `gm_xp.cfg` | `gm_xp` | Yes | XP award amounts per event type. |
| `gm_rank.cfg` | `gm_rank` | Yes | Score weights per event type, rank tier thresholds per axis. |
| `gm_training.cfg` | `gm_training` | Yes | Vote duration, vote cooldown, challenge definitions (JSON-format). |
| `gm_maps/<mapname>.json` | `gm_gameplay` | On map change | Per-map metadata. |

### 12.2 Master Config (`gm_config.cfg`) Keys

| Key | Type | Description |
|---|---|---|
| `gm_db_host` | String | Database host |
| `gm_db_port` | Int | Database port (default: 3306) |
| `gm_db_name` | String | Database name |
| `gm_db_user` | String | Database user |
| `gm_db_pass` | String | Database password |
| `gm_db_pool_min` | Int | Min connections (default: 4) |
| `gm_db_pool_max` | Int | Max connections (default: 16) |
| `gm_server_id` | Int | This server's `servers.server_id` |
| `gm_debug_mode` | Bool | Enable DEBUG log level (default: 0) |
| `gm_webhook_records` | String | Discord webhook URL for `#records` |
| `gm_webhook_admin` | String | Discord webhook URL for `#admin-alerts` |
| `gm_webhook_banlog` | String | Discord webhook URL for `#admin-log` |
| `gm_default_language` | String | Default locale for new accounts (default: "id") |

### 12.3 Hot-Reload Procedure

`!a reloadconfig` triggers `GM_Config_Reload()`, which:
1. Re-reads all `gm_*.cfg` files from disk.
2. Calls each module's `OnConfigReloaded` forward (fired by `gm_core`).
3. Modules update their in-memory config values in the `OnConfigReloaded` handler.
4. Map metadata is NOT reloaded by `!a reloadconfig`; use `!a reloadmap` instead.

---

## 13. LOGGING ARCHITECTURE

### 13.1 Log Files

Each module writes to its own dedicated log file. No module writes to another module's log.

| Module | Log File | Default Level |
|---|---|---|
| `gm_account` | `logs/gm_account.log` | INFO |
| `gm_gameplay` | `logs/gm_gameplay.log` | INFO |
| `gm_movement` | `logs/gm_movement.log` | WARN |
| `gm_team` | `logs/gm_team.log` | INFO |
| `gm_jumpstats` | `logs/gm_jumpstats.log` | INFO |
| `gm_xp` | `logs/gm_xp.log` | INFO |
| `gm_rank` | `logs/gm_rank.log` | INFO |
| `gm_achievements` | `logs/gm_achievements.log` | INFO |
| `gm_cosmetics` | `logs/gm_cosmetics.log` | WARN |
| `gm_training` | `logs/gm_training.log` | INFO |
| `gm_antiexploit` | `logs/gm_antiexploit.log` | INFO |
| `gm_admin` | `logs/gm_admin.log` | INFO |

### 13.2 Log Entry Format

```
[YYYY-MM-DD HH:MM:SS.mmm UTC] [LEVEL] [module_name] message
```

Example:
```
[2026-06-15 14:32:07.441 UTC] [INFO] [gm_account] Player connected. account_id=1042 steam_id=76561198012345678 username=GarudaPlayer
[2026-06-15 14:32:08.019 UTC] [INFO] [gm_jumpstats] New PB recorded. account_id=1042 jump_type=LJ distance=251.42 sync=87.3%
[2026-06-15 14:32:08.021 UTC] [CRITICAL] [gm_jumpstats] DB write failed for record. account_id=1042 error=Connection timeout
```

### 13.3 CRITICAL Escalation

`GM_Log_Critical()` performs two actions:
1. Writes to the module's log file.
2. Enqueues a Discord webhook payload to `gm_admin` for delivery to `#admin-alerts`.

CRITICAL events are defined as:
- Database connection failure.
- Database write failure for player progression data (XP, level, rank).
- JumpStat record loss after validation.
- Plugin load dependency failure.
- Any unhandled exception caught by a try/catch equivalent.

---

## 14. FAILURE HANDLING

### 14.1 Failure Severity Classification

| Severity | Definition | Response |
|---|---|---|
| S1 — Critical | Server or data at risk. Progression data loss possible. | Log CRITICAL, Discord alert, human intervention required. |
| S2 — Error | Feature unavailable. Player impacted but server stable. | Log ERROR, graceful degradation, auto-retry if applicable. |
| S3 — Warning | Anomaly detected. System still functioning. | Log WARN, self-correcting if possible. |
| S4 — Info | Expected operational event. | Log INFO. No action needed. |

### 14.2 Module Failure Scenarios and Responses

#### `gm_account` — Connect-time DB query fails

| Condition | Response |
|---|---|
| Steam identity lookup returns no rows (new player) | Create new account. Expected path. |
| DB query times out | Retry once (1-second delay). On second failure: kick player with localized "server busy" message. Log ERROR. |
| DB pool reports `GMDBStatus.Offline` | All connecting players are kicked with localized message. Server remains running. Log CRITICAL. Discord alert. |

#### `gm_account` — Disconnect-time write fails

| Condition | Response |
|---|---|
| Session close write fails | Log ERROR. Playtime for this session is lost. Do NOT retry (player is gone). |

#### `gm_jumpstats` — Jump validation/commit fails

| Condition | Response |
|---|---|
| Validation fails | Jump silently discarded. Player is not notified. No log entry (expected path). |
| DB write fails after validation | Log ERROR. Jump is lost for this commit. In-memory PB is not updated. Player is notified via chat that their jump could not be saved. Retry is not attempted (jump moment has passed). |
| Server Record detection query fails | Log ERROR. `is_server_record` flag not updated. The jump record exists but is not marked as SR. Admin can correct via `!a invalidatejs` and re-evaluation tool. |

#### `gm_xp` — XP write fails

| Condition | Response |
|---|---|
| `xp_transactions` INSERT fails | Log ERROR. XP is not awarded. Fire `GM_Log_Critical` if this is the second consecutive failure. |
| `player_levels` UPDATE fails | Log ERROR. In-memory cache is already updated. Next session connect will re-read correct DB value. Warn player via chat that XP may not have saved. |

#### `gm_gameplay` — Round participant write fails

| Condition | Response |
|---|---|
| `round_participants` batch INSERT fails | Log ERROR. Stats for this round are lost for all participants. XP awards proceed regardless (from in-memory data). |
| `rounds` UPDATE at round end fails | Log ERROR. Round record remains incomplete (`ended_at = NULL`). Scheduled cleanup task closes stale rounds daily. |

#### `gm_rank` — Rank score update fails

| Condition | Response |
|---|---|
| `rank_scores` UPDATE fails | Log ERROR. In-memory score is already computed correctly. On next connect, the DB value will be stale. Admin can trigger manual rank recompute via future admin tool. |
| Server position recompute query fails | Log WARN. `server_position` in cache and DB remains at last known value. TOP 10 / TOP 3 tag assignment deferred to next round end. |

#### `gm_cosmetics` — Rendering hooks fail

| Condition | Response |
|---|---|
| Knife model override fails | Log WARN. Player sees default knife. No gameplay impact. |
| Trail particle spawn fails | Log WARN. No trail visible. No gameplay impact. |

#### Any module — Plugin fails to load

| Condition | Response |
|---|---|
| Non-critical module fails to load (`gm_cosmetics`, `gm_training`, `gm_hud`, `gm_rank`, `gm_achievements`) | Log CRITICAL to `gm_admin.log`. Discord alert. Server remains fully playable. Affected feature is unavailable until plugin is reloaded. |
| Critical module fails to load (`gm_account`, `gm_gameplay`, `gm_movement`, `gm_team`) | Log CRITICAL. Discord alert. Server should be restarted. `gm_account` failure requires immediate operator response. |

### 14.3 Database Pool Degradation

| Pool Status | `GM_DB_Status()` Returns | Behavior |
|---|---|---|
| All connections healthy | `GMDBStatus.OK` | Normal operation. |
| Some connections dropped, pool rebuilding | `GMDBStatus.Degraded` | Non-critical writes (cosmetics, HUD prefs) are queued or dropped. Gameplay writes (XP, stats) retry once. Log WARN per dropped write. |
| All connections lost | `GMDBStatus.Offline` | All DB writes dropped. All connecting players kicked. Log CRITICAL. Discord alert. |

The DB pool attempts automatic reconnection every 5 seconds when in Degraded or Offline state.

### 14.4 Graceful Degradation Contract

Every module must handle the case where `GM_Cache_IsLoaded(client)` returns `false`. All gameplay logic must check this before acting on player data. If a player's data is not yet loaded (connect-time queries still pending), the module must either:
- Wait for `GM_OnPlayerConnect` to fire (preferred — subscribe to the event).
- Return a safe default value from its native.
- Skip the action and retry on the next event cycle.

A module must never operate on unloaded player data.

---

## 15. DISCORD WEBHOOK ARCHITECTURE

### 15.1 Webhook Manager

Discord webhook delivery is managed exclusively by `gm_admin`. No other module sends webhooks directly. Other modules trigger webhooks by consuming events that `gm_admin` subscribes to, or by calling the webhook enqueue function:

`GM_Admin_EnqueueWebhook(const char[] webhook_url, const char[] payload_json)`

### 15.2 Delivery Mechanism

SourceMod's `SteamWorks` or `cURL` extension is used for outbound HTTP POST. The implementation must:
- Send the request asynchronously (non-blocking).
- On HTTP error or timeout: increment retry count, schedule retry with exponential backoff.
- Maximum 3 retry attempts per payload.
- After 3 failures: drop payload, log WARN.

### 15.3 Webhook Payloads — V1.0

#### Server Record Broken
- **Trigger:** `GM_OnServerRecordBroken`
- **Channel:** `#records`
- **Content:** Player name, jump type label, new distance (2dp), previous distance (2dp), map name, timestamp.

#### CRITICAL Error
- **Trigger:** `GM_Log_Critical()` call from any module.
- **Channel:** `#admin-alerts`
- **Content:** Module name, error message, server name, timestamp.

#### Player Banned
- **Trigger:** `GM_Account_BanPlayer()` native call.
- **Channel:** `#admin-log`
- **Content:** Admin name, target player name, reason, duration (or "Permanent"), timestamp.

### 15.4 Webhook URL Configuration

Webhook URLs are stored in `gm_config.cfg` under keys `gm_webhook_records`, `gm_webhook_admin`, `gm_webhook_banlog`. Empty string disables that webhook. Configuration is hot-reloadable.

---

## 16. BUILD SYSTEM

### 16.1 Compilation

All modules are compiled with SourceMod's `spcomp` compiler. The `tools/build.sh` script compiles all `.sp` files in the correct dependency order.

**Compilation flags (recommended):**

| Flag | Purpose |
|---|---|
| `-O2` | Optimization level 2 |
| `-v2` | Verbose error output |
| `-i scripting/include` | Include path for all `gm_*.inc` files |
| `-D GM_VERSION="1.0"` | Version constant injected at compile time |

### 16.2 Build Order

`build.sh` compiles modules in the same order as the plugin load order (Section 4). This ensures include files referenced by multiple modules are validated against a consistent include tree.

### 16.3 Versioning

Each compiled `.smx` binary embeds the build version via the `GM_VERSION` constant defined at compile time. Admin tool `!a dbstatus` reports the version of each loaded module.

Version format: `MAJOR.MINOR.PATCH` — e.g., `1.0.0`.

### 16.4 Deployment

`tools/deploy.sh` copies compiled `.smx` files to the server's `addons/sourcemod/plugins/` directory and triggers a `sm plugins reload` for each module in the correct load order.

---

## 17. FUTURE COMPATIBILITY PROVISIONS

### 17.1 Phase 2 — Discord Bot and API

The plugin architecture makes zero assumptions about the external API layer. The database is the interface. No plugin exposes a socket, port, or HTTP endpoint. The Phase 2 REST API connects directly to the database in read mode — no plugin modification is required.

### 17.2 Phase 2 — Friends and Account Merge

The `friends` and `account_merge_log` tables are reserved in the schema. When Phase 2 ships:
- A new module `gm_social.smx` will own these tables.
- It will be inserted into the load order between `gm_account` and `gm_region`.
- No existing module requires modification.

### 17.3 Phase 3 — Additional Jump Types

`gm_jumpstats` is already parameterized by `GMJumpType` enum. Adding new jump types requires:
- Adding values to `GMJumpType` in `gm_enums.inc`.
- Adding new constants to `gm_jumpstats.cfg`.
- No structural changes to any other module.

### 17.4 Phase 3 — Jump Replay System

`jumpstat_tick_data` is written from V1.0 launch. A future `gm_replay.smx` module will:
- Read tick data via DB query (no new native required from `gm_jumpstats`).
- Render replay using client-side position/angle overrides.
- Insert itself between `gm_hud` and `gm_menu` in the load order.

### 17.5 Phase 4 — Multi-Server Federation

Every write-heavy table carries `server_id`. When federation is introduced:
- Leaderboards become cross-server queries against the shared database.
- `gm_jumpstats` and `gm_rank` gain a new `GM_*_GetGlobalLeaderboard` native variant.
- The existing `GM_*_GetLeaderboard` natives continue serving server-local results.
- No breaking changes to existing modules.

### 17.6 New Language Support

`GM_Lang_*` functions resolve locale from a string key (`"id"`, `"en"`). Adding a new locale requires:
- Adding `.phrases.txt` entries for the new locale code.
- Adding the new code to the language validation in `gm_account` (for `!lang` selection).
- No code changes to any module.

---

## APPENDIX A: MODULE QUICK REFERENCE

| Module | Binary | Primary DB Tables | Fires Events | Consumes Events | Owns Natives |
|---|---|---|---|---|---|
| `gm_core` | `.inc` | None | — | — | GM_DB_*, GM_Lang_*, GM_Config_*, GM_Cache_*, GM_Log_* |
| `gm_account` | `.smx` | accounts, sessions, player_stats, player_levels, player_settings | Connect, Disconnect | None | GM_Account_* |
| `gm_region` | `.smx` | (accounts.region_code) | None | Connect | GM_Region_* |
| `gm_movement` | `.smx` | None | Liftoff, Landed, Boost | Connect, Disconnect, RoundStart | GM_Movement_* |
| `gm_team` | `.smx` | None | TeamAssigned, CTQueueUpdated | Connect, Disconnect, RoundEnd | GM_Team_* |
| `gm_gameplay` | `.smx` | rounds, round_participants | RoundStart, CountdownGo, RoundEnd, Eliminated, Survived, FrostNade, MapLoaded | Connect, Disconnect, TrainingStart/End | GM_Gameplay_* |
| `gm_jumpstats` | `.smx` | jumpstat_records, jumpstat_tick_data | JumpRecorded, ServerRecordBroken | Liftoff, Landed, Connect, Disconnect, CountdownGo, TrainingStart/End | GM_JumpStats_* |
| `gm_xp` | `.smx` | xp_transactions, player_levels | XPAwarded, LevelUp | RoundEnd, JumpRecorded, AchievementUnlocked | GM_XP_* |
| `gm_rank` | `.smx` | rank_scores, rank_history | RankChanged, TagAutoAssigned | RoundEnd, JumpRecorded, Connect | GM_Rank_* |
| `gm_achievements` | `.smx` | player_achievements | AchievementUnlocked | Connect, Disconnect, RoundEnd, JumpRecorded, LevelUp, RankChanged, FrostNade | GM_Achievement_* |
| `gm_cosmetics` | `.smx` | player_cosmetics, cosmetic_catalog | CosmeticGranted, CosmeticEquipped | Connect, Disconnect, AchievementUnlocked, LevelUp, TagAutoAssigned | GM_Cosmetics_* |
| `gm_training` | `.smx` | None | VoteStart, TrainingStart, TrainingEnd, CheckpointSaved, ChallengeCompleted | Connect, Disconnect | GM_Training_* |
| `gm_hud` | `.smx` | None | None | JumpRecorded, CountdownGo, RoundStart, RoundEnd, Connect, Disconnect | None |
| `gm_antiexploit` | `.smx` | bug_reports | ExploitDetected | Liftoff, Landed, Connect, Disconnect, RoundStart | GM_AntiExploit_* |
| `gm_menu` | `.smx` | None | None | Disconnect | None |
| `gm_commands` | `.smx` | None | None | None | None |
| `gm_admin` | `.smx` | admin_log | None | ServerRecordBroken, ExploitDetected | None |

---

## APPENDIX B: VERSION HISTORY

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0 | June 2026 | Lead Technical Architect | Initial production plugin architecture specification for GM-HNS V1.0 |

---

*End of GM_PLUGIN_ARCHITECTURE.md*

---

> **This document is the official V1.0 plugin architecture specification for the Garuda Movement HNS Ecosystem.**
> All implementation decisions must reference and conform to this specification.
> This document is subordinate to GM_HNS_MASTER_SPEC_V1_FINAL.md on all gameplay decisions
> and to GM_DATABASE_SPEC.md on all data model decisions.
> Architecture deviations require a formal amendment with version increment.
