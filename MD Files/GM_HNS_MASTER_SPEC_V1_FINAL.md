# GM_HNS_MASTER_SPEC_V1_FINAL.md
## Garuda Movement — HNS Chase Ecosystem
### Counter-Strike: Source v34 | Master Technical Specification
### Version 1.0 FINAL | June 2026

---

> **Document Status:** OFFICIAL V1.0 PROJECT CONTRACT
> **Supersedes:** GM_HNS_MASTER_SPEC.md (v1.0.0 Draft)
> **Classification:** Internal Architecture & Design
> **Maintainer:** Lead Technical Architect, Garuda Movement
> **Scope:** Production-Grade Modular HNS Chase Ecosystem — CSS v34 — SEA Region

---

## ARCHITECTURE REVIEW SUMMARY

This document supersedes the original GM_HNS_MASTER_SPEC.md. A full architecture review was conducted against the approved V1.0 project decisions. The following categories of changes were applied before this specification was written:

| Change Type | Count | Summary |
|---|---|---|
| Removed | 15 | Out-of-scope features, P2W mechanics, unsupported cosmetics |
| Revised | 14 | Aligned to approved decisions (ratio, movement, account, cosmetics) |
| Added | 6 | Missing approved features (FrostNades, boosts, CT Queue, knife system, NonSteam support, vote system) |

All changes are binding. This document is the single source of truth for V1.0.

---

## TABLE OF CONTENTS

1. [Project Vision](#1-project-vision)
2. [Design Principles](#2-design-principles)
3. [Gameplay Rules](#3-gameplay-rules)
4. [Team System](#4-team-system)
5. [Training System](#5-training-system)
6. [JumpStats System](#6-jumpstats-system)
7. [XP and Level System](#7-xp-and-level-system)
8. [Achievement System](#8-achievement-system)
9. [Cosmetics System](#9-cosmetics-system)
10. [Account System](#10-account-system)
11. [Regional System](#11-regional-system)
12. [Commands](#12-commands)
13. [Menus](#13-menus)
14. [Language System](#14-language-system)
15. [Ranking System](#15-ranking-system)
16. [Database Requirements](#16-database-requirements)
17. [Plugin Architecture](#17-plugin-architecture)
18. [API Requirements](#18-api-requirements)
19. [Future Roadmap](#19-future-roadmap)

---

## 1. PROJECT VISION

### 1.1 Mission Statement

The Garuda Movement HNS Ecosystem (GM-HNS) is a production-grade, modular server-side framework for Counter-Strike: Source v34, engineered to deliver a competitive, skill-driven Hide and Seek Chase experience for the Southeast Asian movement community.

The project is not a simple plugin. It is a fully integrated ecosystem: a living system of independent modules united by a shared data layer, a coherent design philosophy, and a long-term commitment to fair, skill-based gameplay.

The core identity of GM-HNS is **movement skill without mechanical assistance**. There is no AutoBhop. Mastery is earned through practice, not automation.

### 1.2 Vision Statement

To build Southeast Asia's most technically sound and community-respected HNS platform on Counter-Strike: Source — attracting competitive players and movement enthusiasts through rigorous skill-based gameplay, transparent progression, and equal treatment of every player regardless of account status.

### 1.3 Target Audience

| Segment | Description |
|---|---|
| Movement Specialists | Players focused on scroll-jump, strafing, and air movement mastery |
| HNS Competitors | Players invested in ranked survival and chase gameplay |
| Achievement Hunters | Players driven by long-term progression goals |
| Community Regulars | Players seeking a reliable, fair, daily play environment |
| SEA Movement Community | Indonesian, Malaysian, Singaporean, Filipino, and Thai HNS communities |

### 1.4 Competitive Benchmarks

The following servers are studied as reference implementations for technical quality and community features:

- **Cybershoke HNS** — Reference for JumpStats precision, ranking architecture, and plugin stability.
- **CS-Dream** — Reference for movement training systems and community retention mechanics.

GM-HNS does not aim to clone these servers. It aims to build a technically superior, SEA-native platform with its own identity.

### 1.5 Core Differentiators

- **No AutoBhop.** Movement mastery is the skill gate, not a toggle.
- **HNS Chase mechanics** — FrostNades, Head Boost, Double Boost, Human Ladder, Duck Under, Underblock.
- **Steam + NonSteam support** — Accessible to the full SEA CSS player base.
- **Equal gameplay for all** — No paid or tiered gameplay advantages. Every player competes identically.
- **Modular plugin architecture** — Each feature domain deploys and fails independently.
- **Dual-language interface** — Indonesian (canonical) and English (mandatory fallback).
- **SEA-first regional architecture** — Designed from the ground up for Southeast Asian communities.

---

## 2. DESIGN PRINCIPLES

### 2.1 Foundational Principles

The following principles are binding on all V1.0 implementation decisions. Any system or feature that violates a principle must be escalated for architectural review before proceeding.

#### P-01: Modularity First
Every feature domain is a discrete plugin module. No module is coupled to another by hard dependency at the code level. Inter-module communication occurs exclusively through the defined event bus and shared core library.

#### P-02: Data Integrity Above All
Player data — XP, ranks, stats, achievements — is sacred. No operation that modifies player data may proceed without a recoverable write path. All writes are transactional. Data loss is classified as a Severity-1 incident.

#### P-03: Performance is a Feature
The server runs on CSS v34 at 66 tick. All modules must operate within a CPU budget that does not degrade server tickrate. Expensive operations (database writes, stat computation) are performed asynchronously. No module may block the game thread.

#### P-04: Transparency to the Player
Every system the player interacts with must be explainable in-game. Ranking formulas, XP rewards, and cooldown timers are discoverable via menus or commands. Hidden mechanics are prohibited.

#### P-05: Fairness and Anti-Exploit
Gameplay-affecting systems (movement validation, JumpStats recording, XP rewards) include server-side verification. Client-authoritative data is never trusted. All stat records are auditable.

#### P-06: Graceful Degradation
If a module fails (database timeout, plugin error), the server must remain playable. Non-critical features (cosmetics, stat recording) failing must not terminate the round or disconnect players.

#### P-07: Localization by Default
All player-facing strings are defined in the Language System. No hardcoded strings of any language exist in any module. The Indonesian locale is canonical; English is the mandatory fallback.

#### P-08: Extensibility over Completeness
Ship fewer systems done correctly than many systems done poorly. Every V1.0 system is designed to accept future extension without requiring schema migrations or plugin rewrites.

#### P-09: Equal Gameplay — No Exceptions
No account tier, donor status, or external condition may grant any player a gameplay advantage over another. Movement speed, damage, FrostNade count, weapon access, and XP rates are identical for all players. This principle is absolute and non-negotiable.

### 2.2 Technical Constraints

| Constraint | Value |
|---|---|
| Engine | Counter-Strike: Source v34 (HL2MP base) |
| Server Framework | SourceMod 1.11+ |
| Tick Rate | 66 tick |
| Recommended Player Slots | 24 |
| Hard Maximum Player Slots | 32 |
| Database Layer | MySQL 5.7+ or MariaDB 10.4+ |
| Min Supported Client | CSS v34 (Steam and NonSteam) |
| AutoBhop | DISABLED — server-enforced |

---

## 3. GAMEPLAY RULES

### 3.1 Game Mode Overview

GM-HNS operates in **HNS Chase** mode. This is a pursuit-and-evasion game mode built on the Counter-Strike: Source round framework.

- **Terrorists (T)** are designated **Hiders**.
- **Counter-Terrorists (CT)** are designated **Chasers**.

The central skill mechanic is **scroll-jump movement**. There is no AutoBhop. Players who master scroll jumping, strafing, and the HNS Chase-specific mechanics (Head Boost, Double Boost, Human Ladder, Duck Under, Underblock) gain competitive advantage through practice, not configuration.

### 3.2 Movement Philosophy

AutoBhop is **permanently disabled** server-side. This is a locked project decision and is not configurable by admins or players.

Approved movement techniques:

| Technique | Description |
|---|---|
| Scroll Jump | Timed jump input via scroll wheel or manual key |
| Strafing | Air direction control via A/D keys synchronized with mouse |
| Pre-strafe | Ground acceleration through directional input before jumping |
| Head Boost | Using a teammate's head as a launch surface to gain extra height |
| Double Boost | Two teammates stacked, providing a two-level height boost |
| Human Ladder | A team of players stacked vertically to reach elevated positions |
| Duck Under | Crouching under a moving or stationary obstacle at speed |
| Underblock | Exploiting map geometry to pass under surfaces at speed |

### 3.3 Win Conditions

| Outcome | Condition | Result |
|---|---|---|
| Hider Victory | At least one Hider survives to round end | T team wins |
| Chaser Victory | All Hiders are eliminated before round end | CT team wins |
| Draw | No Hiders remain but Chasers made zero eliminations | No score |

### 3.4 Round Structure

#### 3.4.1 Pre-Round Phase (CT Freeze Time)
- Duration: **5 seconds** (locked).
- Hiders are free to move immediately at round start.
- Chasers (CT) are frozen in spawn for the full 5 seconds.
- No weapon fire permitted during freeze.
- HUD displays: round timer, map name, player count, team counts.

#### 3.4.2 Countdown Sequence
At the end of the CT freeze period, a server-wide countdown broadcasts:

```
5 ... 4 ... 3 ... 2 ... 1 ... GO!
```

- Countdown is displayed in HUD and played as an audio cue.
- At "GO!", CT freeze is lifted and Chasers may move.
- The countdown is non-interruptible and not configurable per player.

#### 3.4.3 Active Phase
- Duration: configurable per map (default: 3 minutes 30 seconds).
- Full HNS Chase gameplay.
- Round timer visible to all players in HUD.
- Death is immediate and non-respawnable within the round.
- FrostNades are distributed to Hiders at round start (see Section 3.6).

#### 3.4.4 Post-Round Phase
- Duration: 5 seconds.
- Round result panel displayed: winner team, last Hider alive (if any), top Chaser (by eliminations), survival duration.
- Stat snapshots committed to the database asynchronously.
- XP and achievement evaluations triggered.
- Team rebalance evaluation (see Section 4.3).

### 3.5 Hider Rules

- **Weapons:** Knife only. No firearms permitted.
- **FrostNades:** 2 per round, T-only (see Section 3.6).
- **Spawn Zone:** Hiders must vacate T spawn within 20 seconds of round start. Violation: warning at 15 seconds, auto-slay at 20 seconds.
- **Movement:** All approved techniques in Section 3.2 permitted.
- **Objective:** Survive until the round timer expires.

### 3.6 FrostNades

FrostNades are a core HNS Chase mechanic that allow Hiders to slow or freeze pursuing Chasers.

| Property | Value |
|---|---|
| Team | Hiders (T) only |
| Count per Round | 2 per Hider |
| CT Access | Prohibited — Chasers cannot acquire or use FrostNades |
| Effect | Slows affected CT players for a configurable duration (default: 3 seconds) |
| Radius | Configurable (default: 150 Hammer Units) |
| Distribution | Automatic at round start via loadout assignment |
| Carry-Over | FrostNades do not carry over between rounds |

FrostNade count is fixed and identical for all Hiders. This count is not modifiable by account tier, VIP status, or any other condition.

### 3.7 Chaser Rules

- **Weapons:** Standard CT loadout. Configurable per server (default: all CT weapons permitted).
- **Spawn Restriction:** Frozen for 5 seconds (CT Freeze Time). May not fire during freeze.
- **CT Spawn Exit:** Chasers may not re-enter CT spawn after leaving it for longer than 10 seconds. Violation: warning, then forced teleport.
- **FrostNades:** Cannot be held or used by Chasers under any condition.
- **Objective:** Eliminate all Hiders before the round timer expires.

### 3.8 Map Requirements

Maps admitted to the GM-HNS rotation must satisfy:

- No accessible out-of-bounds geometry for either team.
- Distinct, non-overlapping T and CT spawn zones.
- Minimum three navigable paths between spawn zones.
- Geometry that meaningfully supports Head Boost, Double Boost, and Human Ladder techniques.
- A `.json` metadata file (see Section 11.3 for schema).

### 3.9 Anti-Exploit Policy

- **Speed Validation:** Server-side velocity validation against the engine movement cap every tick.
- **Teleport Detection:** Player position delta validated per tick. Anomalies discard the affected round's stats and flag the account for review.
- **JumpStat Integrity:** All JumpStat records are server-computed. Client-reported values are never used.
- **FrostNade Exploit Prevention:** The FrostNade distribution system is server-authoritative. Clients cannot modify nade count or acquire Chasers' FrostNades.
- **Exploit Reporting:** Players may use `!bug` to flag map exploits. Reports are logged with position data, round ID, and timestamp.

---

## 4. TEAM SYSTEM

### 4.1 Overview

The Team System manages player assignment, dynamic team balancing, the CT Queue System, and team-specific rule enforcement.

### 4.2 Team Definitions

| Team | CSS Team | HNS Role | Target Population |
|---|---|---|---|
| Hiders | Terrorist (T) | Evade and survive | ~70% of players |
| Chasers | Counter-Terrorist (CT) | Hunt and eliminate | ~30% of players |

### 4.3 Team Ratio and Dynamic Balancing

The target team composition is **70% Hiders / 30% Chasers**, computed from the current online player count and rounded to the nearest whole player.

| Online Players | Target Hiders | Target Chasers |
|---|---|---|
| 4 | 3 | 1 |
| 6 | 4 | 2 |
| 10 | 7 | 3 |
| 16 | 11 | 5 |
| 24 | 17 | 7 |
| 32 | 22 | 10 |

#### 4.3.1 Auto Team Balance
- Evaluated at the end of each round.
- If actual composition deviates from target by more than 1 player, balance triggers.
- Candidates for swap are selected from players with the lowest contribution score (Chasers: fewest eliminations; Hiders: shortest survival time) in the completed round.
- A player cannot be auto-swapped on consecutive rounds.
- Auto-balance never overrides the CT Queue System (Section 4.4).

#### 4.3.2 No Instant Team Swap
- Players cannot switch teams mid-round.
- Team swap commands (`!team ct`, `!team t`) queue the player for assignment at the next round start.
- A player who requests CT will be placed in the CT Queue (Section 4.4) and assigned when a CT slot is available.

#### 4.3.3 Persistent Teams
- Team assignment persists across rounds until a player explicitly requests a change or is moved by auto-balance.
- Players who disconnect and reconnect within 60 seconds are restored to their previous team assignment.

### 4.4 CT Queue System

The CT Queue ensures fair rotation of the Chaser role across a session.

- The queue is maintained per-session (resets on disconnect).
- A player joining the queue is added to the back.
- Players are assigned CT from the front of the queue when a CT slot becomes available (due to player departure or team balance adjustment).
- A player may decline a CT assignment once per session without losing their queue position.
- On second consecutive decline, the player is moved to the back of the queue.
- The queue is visible to the player via `!queue` command.

### 4.5 Team Commands

| Command | Effect |
|---|---|
| `!team ct` | Join CT Queue for Chaser role at next round |
| `!team t` | Request Hider (T) assignment at next round |

### 4.6 Spectator and AFK Management

- Players may spectate freely between rounds.
- Spectators cannot communicate via team-restricted chat.
- AFK detection triggers after 3 consecutive rounds without input.
- AFK players receive a confirmation prompt. Non-response within 60 seconds results in kick.

---

## 5. TRAINING SYSTEM

### 5.1 Overview

The Training System is an in-server mode allowing players to practice movement mechanics, study JumpStat technique, and complete skill challenges. Training Mode does **not** affect ranked stats, JumpStat server records, or XP — except for one-time XP rewards from Guided Challenges (Section 5.4).

### 5.2 Training Mode Activation

#### 5.2.1 Solo Activation
If the player requesting Training Mode is the **only player on the server**, Training Mode activates immediately upon `!training`.

#### 5.2.2 Multi-Player Vote
If other players are online, Training Mode requires a server vote.

- The requesting player initiates the vote with `!training`.
- A vote prompt is shown to all online players.
- Vote passes if: ≥50% of online players vote yes, within 30 seconds.
- If the vote fails, a 5-minute cooldown applies before another vote can be initiated.
- If the vote passes, Training Mode activates at the start of the next round.

#### 5.2.3 Admin Override
Admins may activate Training Mode without a vote using `!a training start`.

### 5.3 Training Commands

| Command | Function |
|---|---|
| `!training` | Initiate Training Mode (solo) or start vote (multi-player) |
| `!play` | Exit Training Mode and return to live HNS Chase |
| `!cp` | Save a checkpoint at the player's current position |
| `!tp` | Teleport to the player's most recently saved checkpoint |

Checkpoints are per-player, per-session. A player may hold one checkpoint at a time. Saving a new checkpoint overwrites the previous one.

### 5.4 Training HUD Overlays

The following HUD overlays are available during Training Mode. Each is individually togglable via the Settings Menu.

| Overlay | Information Displayed |
|---|---|
| Speed HUD | Current horizontal velocity in Hammer Units per second |
| Sync HUD | Real-time strafe sync percentage for the current jump |
| Strafe HUD | Per-strafe gain/loss in velocity units |

All overlays are disabled by default and must be enabled by the player via `!hud` or the Settings Menu.

### 5.5 Training Modes

#### 5.5.1 Free Movement Mode
- No round timer.
- No opponents.
- AutoBhop remains **disabled** — scroll jump only, consistent with live server.
- JumpStats recorded to a **training-only leaderboard** (separate from server records).
- Speed, Sync, and Strafe HUDs available.

#### 5.5.2 JumpStats Practice
- Curated map set with marked jump surfaces and measured reference distances embedded in map geometry.
- Each approved jump type (Section 6.2) has a dedicated practice zone.
- Personal bests tracked against the training leaderboard only.
- Does not contribute to the server JumpStats leaderboard.

#### 5.5.3 Strafing Workshop
- Map set with defined strafing corridors and accuracy target zones.
- Strafing efficiency score computed per corridor (sync % + velocity gain).
- Scores stored to the player's training record.
- Recommended for players below Rank Tier III (see Section 15).

#### 5.5.4 Guided Challenges
Structured skill challenges with pass/fail evaluation. Each challenge is completable once per account for its XP reward. Challenge progress survives server restarts.

| Tier | Example Challenge | One-Time XP Reward |
|---|---|---|
| Beginner | Complete a 5-jump scroll-jump chain without landing | 100 XP |
| Intermediate | Achieve 240 u/s average velocity over a 10-second run | 300 XP |
| Advanced | Land a Longjump of 240+ units | 500 XP |
| Elite | Land a Longjump of 260+ units | 1,000 XP |

Completing all challenges in a tier unlocks a Training Badge Title (see Section 9.4).

### 5.6 Training Exclusions

The following systems are **not part of V1.0 Training** and are explicitly out of scope:

- AI-driven training opponents.
- Bot-based Hider/Chaser simulation.
- Seeker AI.
- Machine learning movement analysis.

---

## 6. JUMPSTATS SYSTEM

### 6.1 Overview

The JumpStats System records, validates, and ranks player movement feats with precision and full transparency. It is the core technical prestige system of GM-HNS — the definitive measure of a player's movement mastery.

All measurements are server-computed. No client-provided measurement is used or trusted.

### 6.2 Approved Jump Types — V1.0

Only the following five jump types are tracked in V1.0. No other jump types are recorded or displayed.

| Jump Type | Code | Description |
|---|---|---|
| Longjump | LJ | Ground-to-ground jump from standing pre-strafe |
| Countjump | CJ | Double-tap jump off crouch, measured on landing |
| Multi-Countjump | MCJ | Consecutive CJ chain, measured per landing |
| Ladder Jump | LDJ | Jump initiated from ladder-acquired momentum |
| Double Bhop | DBhop | Two consecutive scroll-jump bhops, measured across both |

Jump types not in this list (BLJ, DJ, WJ, HJ, SJ, MBH) are **not recorded in V1.0** and are deferred to the roadmap.

### 6.3 Measurement Specification

All measurements are in Hammer Units (HU). 1 HU ≈ 1.905 cm in-game.

| Metric | Description |
|---|---|
| Distance | Horizontal ground-to-ground displacement in HU |
| Pre-Jump Speed | Velocity at the frame of jump input |
| Landing Speed | Velocity at the frame of landing detection |
| Strafes | Count of directional strafe key presses during airtime |
| Sync % | Percentage of airtime where mouse direction and strafe key align |
| Airtime | Duration in server ticks from liftoff to landing |

### 6.4 Validation Rules

All jumps must pass server-side validation before being committed.

#### 6.4.1 Velocity Validation
- Pre-jump velocity must not exceed the server ground speed cap.
- Per-tick velocity delta must conform to the air acceleration model for the server's configured `sv_airaccelerate`.
- Any tick with an impossible velocity delta discards the entire jump.

#### 6.4.2 Trajectory Validation
- The jump arc (position per tick) must be consistent with a ballistic trajectory given measured velocity and `sv_gravity`.
- A teleport (position delta exceeding the theoretical per-tick maximum) discards the jump.

#### 6.4.3 Surface Validation
- Launch surface must be a valid ground plane.
- Landing surface must be within declared valid landing zones for the jump type (per map metadata).

#### 6.4.4 Scroll-Jump Integrity
- The jump input must be a single discrete jump event (scroll or key press).
- Consecutive jump inputs at a rate impossible for human input (configurable threshold) discard the jump and flag the account.

### 6.5 Record Tiers

| Tier | Label | Scope | Color |
|---|---|---|---|
| Server Record | SR | Best on this server, all time | Gold |
| Personal Best | PB | Player's best for that jump type | Blue |
| Good | — | Top 25% of all recorded entries | Green |
| Normal | — | Within standard statistical range | White |

V1.0 does not implement regional or global records across multiple servers. Server Record (SR) is the highest prestige tier available at launch.

### 6.6 JumpStats Leaderboards — V1.0

#### 6.6.1 Server Leaderboard
- All-time best per jump type on this server.
- Top 100 per jump type, viewable in-game via `!lb`.

#### 6.6.2 Personal Stats Panel
- Accessible via `!js` or `!jumpstats`.
- Displays: all-time PBs per jump type, rank on server leaderboard, 10 most recent jumps.

### 6.7 Record Announcements

| Event | Scope | Channel |
|---|---|---|
| Server Record broken | Server-wide chat announcement | In-game chat + HUD flash |
| Personal Best improved | Player-visible only | Chat notice to that player |

### 6.8 JumpStats HUD Overlay

During live gameplay and Training Mode, a JumpStats HUD overlay (togglable) displays the last recorded jump:

```
Last Jump: LJ | 248.33 u | Pre: 279 u/s | Sync: 84% | Strafes: 4
```

Toggled via `!hud` command or Settings Menu.

### 6.9 Position Tick Recording

During every validated jump, the server records player position per tick for the duration of the jump (liftoff to landing). This data is stored to the `jumpstat_tick_data` table and is reserved for the Jump Replay System (Phase 3 Roadmap, Section 19). The data is collected from V1.0 launch to ensure a backlog is available when replay is implemented.

---

## 7. XP AND LEVEL SYSTEM

### 7.1 Overview

The XP and Level System provides a linear progression layer that rewards consistent participation. XP is a single currency earned through gameplay, training challenge completions, and achievement rewards. There are no XP multipliers, boosts, or advantages tied to account type.

### 7.2 XP Sources

#### 7.2.1 Gameplay XP

| Event | Condition | Base XP |
|---|---|---|
| Round Survival | Hider survives to round end | 50 XP |
| Elimination | Chaser eliminates a Hider | 30 XP |
| Last Survivor | The final surviving Hider | 75 XP bonus |
| First Blood | First elimination of the round (CT) | 15 XP bonus |
| Round Win (team) | Member of the winning team | 10 XP |
| Round Participation | Alive at any point in the round | 10 XP (floor) |

#### 7.2.2 JumpStats XP

| Event | Condition | XP |
|---|---|---|
| Personal Best | Improved PB in any approved jump type | 20 XP |
| Server Record | Set a new server record in any jump type | 150 XP |

#### 7.2.3 Training XP
One-time XP rewards from Guided Challenge completions only (see Section 5.4). No repeatable training XP.

#### 7.2.4 Achievement XP
Defined per achievement in the Achievement Catalog (Section 8.3).

### 7.3 XP Multipliers — V1.0

There are **no XP multipliers in V1.0**. All players earn XP at identical rates. This upholds Principle P-09.

The following items from the original specification are **removed**:
- VIP +20% XP boost.
- Party Bonus XP.
- Daily login streak multipliers.
- Weekend / Double XP events.
- Underdog team XP bonus.

### 7.4 Level Thresholds

Levels use a progressive curve. The formula is implementation-defined but must satisfy:

- Level 1 → Level 2: approximately 500 XP.
- Each subsequent level requires approximately 12% more XP than the previous.
- Level cap: **100** (extendable in future roadmap).
- No prestige system exists in V1.0.

| Level Range | Label | Badge Color |
|---|---|---|
| 1–10 | Newcomer | Grey |
| 11–25 | Player | Green |
| 26–50 | Veteran | Blue |
| 51–75 | Elite | Purple |
| 76–99 | Legend | Gold |
| 100 | Garuda | Animated |

### 7.5 Level Rewards

Each level delivers at least one reward from the following approved types:

- Cosmetic item unlock (knife skin, trail color).
- Title unlock (from approved list, Section 9.4).
- Tag unlock (from approved list, Section 9.5).
- Extended in-game stat panel access.

Level 100 unlocks the **Garuda** title and the animated Level 100 badge. No gameplay advantage is associated with any level reward.

### 7.6 XP Audit Log

Every XP transaction is logged with: player `account_id`, XP amount, source type, source reference, round ID (if applicable), and timestamp. Logs are retained for 180 days and are queryable by admins.

---

## 8. ACHIEVEMENT SYSTEM

### 8.1 Overview

The Achievement System defines a catalog of player goals that reward engagement across gameplay, movement, progression, and community. Achievements are non-repeatable unless explicitly marked repeatable in their definition.

### 8.2 Achievement Structure

| Field | Type | Description |
|---|---|---|
| `achievement_id` | String slug | Unique identifier |
| `name` | Localization key | Display name |
| `description` | Localization key | Condition description |
| `category` | Enum | See Section 8.3 |
| `tier` | Enum | Bronze / Silver / Gold / Platinum |
| `xp_reward` | Integer | XP on completion |
| `cosmetic_reward` | Optional string | Cosmetic item unlocked |
| `condition` | Structured object | Machine-readable condition |
| `repeatable` | Boolean | Can be re-earned |
| `hidden` | Boolean | Hidden until earned |

### 8.3 Achievement Catalog — V1.0

#### 8.3.1 Survival Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `surv_first` | First Steps | Survive your first round | Bronze | 50 |
| `surv_10` | Persistent | Survive 10 rounds total | Bronze | 100 |
| `surv_100` | Tenacious | Survive 100 rounds total | Silver | 500 |
| `surv_1000` | Immortal | Survive 1,000 rounds total | Gold | 2,000 |
| `surv_streak_5` | Five Alive | Survive 5 consecutive rounds | Silver | 300 |
| `surv_streak_20` | Ghost Protocol | Survive 20 consecutive rounds | Gold | 1,000 |
| `surv_last_10` | Last Man x10 | Be the last Hider alive 10 times | Silver | 500 |
| `surv_frost_stop` | Frost Defense | Successfully slow a Chaser with a FrostNade 50 times | Bronze | 200 |

#### 8.3.2 Chaser Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `chase_first` | First Hunt | Get your first elimination as Chaser | Bronze | 50 |
| `chase_50` | Tracker | Eliminate 50 Hiders total | Bronze | 200 |
| `chase_500` | Hunter | Eliminate 500 Hiders total | Silver | 1,000 |
| `chase_5000` | Apex | Eliminate 5,000 Hiders total | Gold | 5,000 |
| `chase_sweep` | Clean Sweep | Chaser team eliminates all Hiders in one round | Silver | 400 |
| `chase_speed` | Blink | Eliminate a Hider within 10 seconds of GO! | Silver | 300 |

#### 8.3.3 Movement Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `mv_first_lj` | First Leap | Record any Longjump | Bronze | 50 |
| `mv_lj_220` | Strider | Land a Longjump of 220+ units | Bronze | 100 |
| `mv_lj_240` | Jumper | Land a Longjump of 240+ units | Silver | 300 |
| `mv_lj_260` | Flier | Land a Longjump of 260+ units | Gold | 1,000 |
| `mv_cj_first` | Crouch King | Record your first Countjump | Bronze | 75 |
| `mv_dbhop_first` | Double Tap | Record your first Double Bhop | Bronze | 75 |
| `mv_ldj_first` | Ladder King | Record your first Ladder Jump | Bronze | 75 |
| `mv_all_types` | Polymath | Record a PB in all 5 approved jump types | Gold | 2,000 |
| `mv_sr` | Server Best | Hold any Server Record | Platinum | 3,000 |

#### 8.3.4 Progression Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `prog_level_10` | Rising | Reach Level 10 | Bronze | 200 |
| `prog_level_50` | Established | Reach Level 50 | Silver | 1,000 |
| `prog_level_100` | Garuda | Reach Level 100 | Platinum | 10,000 |
| `prog_rank_t10` | Elite Class | Reach the top 10 on the server ranking | Gold | 2,000 |
| `prog_rank_1` | The Best | Reach Rank #1 on any leaderboard | Platinum | 5,000 |

#### 8.3.5 Training Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `train_beginner` | First Steps | Complete all Beginner challenges | Bronze | 200 |
| `train_intermediate` | Student | Complete all Intermediate challenges | Silver | 500 |
| `train_advanced` | Practitioner | Complete all Advanced challenges | Gold | 1,500 |
| `train_elite` | Master | Complete all Elite challenges | Platinum | 5,000 |

#### 8.3.6 Hidden Achievements

A minimum of 10 hidden achievements are defined in the launch catalog. They are not listed to players until earned. Each carries a minimum Gold tier reward. Examples include:

- Playing at a specific server time milestone (e.g., 100th round on the server's history).
- Surviving while all other Hiders were eliminated within 30 seconds of GO!.
- Executing a Human Ladder with 3+ players in a single round (server-detected via position analysis).

### 8.4 Achievement Evaluation

- Evaluated server-side at round end, on JumpStat commit, and on session events.
- Evaluation is asynchronous and does not block the game thread.
- On completion: chat notification + HUD flash displayed to the earning player.
- Server-wide announcement for Platinum tier achievements only.

---

## 9. COSMETICS SYSTEM

### 9.1 Overview

The Cosmetics System manages all player-visible personalization items. Cosmetics are strictly non-gameplay-affecting. No cosmetic may confer any movement advantage, visibility advantage, or stat benefit.

### 9.2 Approved Cosmetic Categories — V1.0

The following categories are approved for V1.0. All other categories from the original specification are removed.

| Category | Description |
|---|---|
| Knife Skin | Visual skin applied to the player's melee weapon |
| Trail | Particle trail following the player during movement |
| Title | Text displayed below the player's name in HUD and scoreboard |
| Tag | Prefix tag displayed in chat alongside the player's name |

**Removed from V1.0 (explicitly out of scope):**
- Player/Character Model Skins
- Nameplates
- Sprays
- Kill Sounds
- Colored Nicknames (name color overrides)
- Animated Nameplates

### 9.3 Knife Skins

| Knife | ID |
|---|---|
| Default | `knife_default` |
| Karambit | `knife_karambit` |
| Butterfly | `knife_butterfly` |
| M9 Bayonet | `knife_m9` |
| Bayonet | `knife_bayonet` |
| Flip Knife | `knife_flip` |
| Gut Knife | `knife_gut` |

All knife skins are visual-only. No knife skin modifies damage, swing speed, or hitbox.

### 9.4 Trails

| Trail | ID |
|---|---|
| Off (None) | `trail_off` |
| Red | `trail_red` |
| Blue | `trail_blue` |
| Green | `trail_green` |
| Yellow | `trail_yellow` |
| Purple | `trail_purple` |
| White | `trail_white` |
| Cyan | `trail_cyan` |
| Orange | `trail_orange` |
| Pink | `trail_pink` |
| Gold | `trail_gold` |
| Rainbow | `trail_rainbow` |

Trails are visible to all players on the server. Trail Off is the default; no trail is rendered unless the player equips one. Trail rendering does not affect hitboxes or player detection.

### 9.5 Titles

Titles are text strings displayed below a player's name in the HUD and scoreboard.

| Title | ID | Unlock Condition |
|---|---|---|
| Escapist | `title_escapist` | Survive 500 rounds total |
| Chaser | `title_chaser` | Eliminate 500 Hiders total |
| Frost Master | `title_frost_master` | Successfully deploy 200 FrostNades |
| Jumper | `title_jumper` | Record a PB in all 5 jump types |
| Veteran | `title_veteran` | Reach Level 50 |

Training Tier Badges (Beginner / Intermediate / Advanced / Elite) function as additional titles unlockable through Guided Challenge completion (Section 5.4).

### 9.6 Tags

Tags are short prefix labels displayed in all chat contexts alongside the player's name.

| Tag | ID | Unlock Condition |
|---|---|---|
| TOP 10 | `tag_top10` | Currently ranked in the top 10 on the server ranking — dynamically assigned |
| TOP 3 | `tag_top3` | Currently ranked in the top 3 on the server ranking — dynamically assigned |
| FOUNDER | `tag_founder` | Admin-granted to founding community members |
| BETA | `tag_beta` | Admin-granted to beta testers |

`TOP 10` and `TOP 3` tags are **automatically applied and removed** based on current server ranking. Players do not manually equip these tags; they are system-assigned. If a player falls out of the top 10, the tag is automatically removed.

`FOUNDER` and `BETA` are permanent once granted and cannot be removed by ranking changes.

### 9.7 Cosmetic Acquisition

| Source | Category | Method |
|---|---|---|
| Level Rewards | Knife, Trail | Automatically unlocked at specific levels |
| Achievement Rewards | Knife, Trail, Title | Unlocked on achievement completion |
| Server Record | Tag (auto) | TOP 3 / TOP 10 — system assigned |
| Admin Grant | Tag | FOUNDER / BETA |

Cosmetics are **never purchasable for real money**. No monetization mechanism exists in V1.0.

### 9.8 Cosmetic Equipping

Players manage equipped cosmetics via `!cosmetics` menu. One item per category may be equipped at a time. Preferences are stored to the Account System and persist across sessions.

---

## 10. ACCOUNT SYSTEM

### 10.1 Overview

The Account System manages player identity, session state, preferences, and cosmetics. The canonical player identity key is `account_id` (UUID), not SteamID64. This design supports both Steam and NonSteam (LSGI-based) players.

### 10.2 Account Types

| Type | Description | Canonical Identity |
|---|---|---|
| Guest | Automatic, no external link. NonSteam players who have not linked an account. | `account_id` (UUID) only |
| Linked Steam | Steam identity linked to the `account_id` | `account_id` + `steam_id_64` |
| Linked Discord | Discord identity linked to the `account_id` | `account_id` + `discord_id` |

A single `account_id` may link both a Steam identity and a Discord identity simultaneously.

#### 10.2.1 Steam + NonSteam Support
GM-HNS V1.0 fully supports both Steam-authenticated and NonSteam (LSGI) players. NonSteam players are issued a Guest account with a generated `account_id` on first connection. Their identity is persisted by a combination of LSGI identifier and IP-hash binding (configurable). NonSteam players may link a Discord account to improve identity persistence.

#### 10.2.2 No Account Tiers with Gameplay Advantages
V1.0 does not implement VIP, VIP+, or any paid/donated account tier. All players are functionally equal. Admin and Owner roles exist for server management purposes only and confer no gameplay advantages.

#### 10.2.3 Account Merge (Reserved)
The Account Merge feature — allowing two accounts to be merged (e.g., a Guest account and a newly linked Steam account) — is reserved for Phase 2. The schema must accommodate this without requiring a migration.

### 10.3 Account Record Fields

| Field | Type | Description |
|---|---|---|
| `account_id` | UUID (PK) | Canonical internal identity |
| `steam_id_64` | String (nullable) | Steam identity, null for Guest |
| `discord_id` | String (nullable) | Discord identity, null if unlinked |
| `lsgi_identifier` | String (nullable) | NonSteam LSGI identifier |
| `username` | String | Last known display name |
| `account_type` | Enum | guest / linked_steam / linked_discord |
| `created_at` | Timestamp (UTC) | First connection |
| `last_seen` | Timestamp (UTC) | Most recent connection |
| `region` | Enum | Assigned region (Section 11) |
| `preferred_language` | Enum | `id` or `en` |
| `total_playtime_seconds` | Integer | Lifetime playtime |
| `is_banned` | Boolean | Account ban status |
| `ban_reason` | String (nullable) | Populated if banned |
| `ban_expiry` | Timestamp (nullable) | Null if permanent |
| `updated_at` | Timestamp (UTC) | Record last updated |

### 10.4 Admin and Owner Roles

| Role | Assignment | Scope |
|---|---|---|
| Admin | Server owner assigns | Moderation, stat management, event control |
| Owner | Reserved for server owner | Unrestricted system access |

Admin and Owner roles carry zero gameplay advantages (no extra speed, no FrostNade count changes, no XP bonuses).

### 10.5 Preferences

| Key | Type | Default | Description |
|---|---|---|---|
| `language` | Enum | `id` | Interface language |
| `hud_jumpstats` | Boolean | true | Show JumpStats HUD overlay |
| `hud_speed` | Boolean | false | Show speed HUD overlay |
| `hud_sync` | Boolean | false | Show sync HUD overlay |
| `hud_strafe` | Boolean | false | Show strafe HUD overlay |
| `chat_notifications` | Boolean | true | Show achievement and record announcements |
| `equipped_knife` | String | `knife_default` | Currently equipped knife skin |
| `equipped_trail` | String | `trail_off` | Currently equipped trail |
| `equipped_title` | String (nullable) | null | Currently equipped title |

### 10.6 Session Tracking

| Field | Type |
|---|---|
| `session_id` | UUID |
| `account_id` | Foreign key |
| `connected_at` | Timestamp UTC |
| `disconnected_at` | Timestamp UTC (nullable) |
| `ip_hash` | SHA-256 of IP (not stored raw) |
| `server_id` | Server identifier |

### 10.7 Data Privacy

- Raw IP addresses are never stored. SHA-256 hashing is applied before any write.
- Usernames are stored as received and are not retained historically.
- Account deletion anonymizes personal identifiers while retaining aggregate stats (XP, JumpStat records) attributed to a deleted-account label.

---

## 11. REGIONAL SYSTEM

### 11.1 Overview

The Regional System classifies players by geographic region to enable regional leaderboards and future inter-regional competition. V1.0 focuses on the SEA region with future-ready architecture for per-country segmentation.

### 11.2 V1.0 Region Definitions

| Region Code | Label | Countries |
|---|---|---|
| `ID` | Indonesia | Indonesia |
| `SG` | Singapore | Singapore |
| `MY` | Malaysia | Malaysia, Brunei |
| `PH` | Philippines | Philippines |
| `TH` | Thailand | Thailand |
| `SEA_OTHER` | SEA Other | All other SEA countries |
| `GLOBAL` | Global | Non-SEA players (catch-all) |

V1.0 ships with a single unified SEA leaderboard. Per-country leaderboards are Phase 2.

### 11.3 Region Assignment

Region is assigned at account creation by GeoIP lookup. The GeoIP database is updated monthly. Region is stored to the account record and is not re-evaluated on subsequent connections.

A player may not self-select their region. Region change requests are reviewed by admins.

### 11.4 Map Metadata Schema

Every map in the rotation carries a `.json` metadata file:

```
map_id            : string
map_name          : string
display_name      : string
author            : string
version           : string
hns_compatible    : boolean
default_round_time: integer (seconds)
spawn_zone_t      : BoundingBox
spawn_zone_ct     : BoundingBox
valid_landing_zones: array<BoundingBox>
boost_geometry    : array<BoostZoneDeclaration>
recommended_players: IntRange
notes             : string
```

Map metadata must be committed and validated before a map enters the rotation.

---

## 12. COMMANDS

### 12.1 Command Standards

- All commands support both `!` (chat) and `/` (console) prefixes.
- Commands with sub-options open the relevant menu section.
- All responses are rendered through the Language System — no hardcoded strings.
- Invalid arguments return a localized usage hint.
- Admin commands return a generic permission-denied message to non-admins.

### 12.2 Player Commands

#### General

| Command | Aliases | Description |
|---|---|---|
| `!menu` | `/menu` | Open main navigation menu |
| `!rules` | `/rules` | Display server rules panel |
| `!discord` | `/discord` | Display Discord invite link |
| `!bug` | `/bug` | Submit exploit/bug report with position data |
| `!help` | `/help` | Open help panel |

#### Profile and Stats

| Command | Aliases | Description |
|---|---|---|
| `!profile` | `/me`, `/p` | Open own profile panel |
| `!profile <name>` | `/p <name>` | Open target player's profile panel |
| `!stats` | `/mystats` | Display personal gameplay statistics |
| `!rank` | `/myrank` | Display personal rank and XP status |
| `!level` | `/mylevel` | Display current level and XP to next |

#### JumpStats

| Command | Aliases | Description |
|---|---|---|
| `!js` | `/jumpstats` | Open personal JumpStats panel |
| `!lb` | `/leaderboard` | Open server JumpStats leaderboard |
| `!pbs` | `/personalbests` | List all personal bests by jump type |
| `!sr` | `/serverrecords` | Display current server records per jump type |

#### Achievements

| Command | Aliases | Description |
|---|---|---|
| `!achievements` | `/ach` | Open achievement browser |

#### Cosmetics

| Command | Aliases | Description |
|---|---|---|
| `!cosmetics` | `/cos` | Open cosmetics management menu |
| `!knife` | `/knife` | Quick-open knife selection |
| `!trail` | `/trail` | Quick-open trail selection |
| `!title` | `/title` | Quick-open title selection |

#### Training

| Command | Aliases | Description |
|---|---|---|
| `!training` | `/train` | Start Training Mode or initiate vote |
| `!play` | `/play` | Exit Training Mode, return to live |
| `!cp` | `/checkpoint` | Save checkpoint at current position |
| `!tp` | `/teleport` | Teleport to saved checkpoint |

#### HUD

| Command | Aliases | Description |
|---|---|---|
| `!hud` | `/hud` | Open HUD settings sub-menu |

#### Team

| Command | Aliases | Description |
|---|---|---|
| `!team ct` | `/team ct` | Join CT Queue for Chaser role |
| `!team t` | `/team t` | Request Hider (T) assignment |
| `!queue` | `/queue` | View current CT Queue position and list |

#### Settings

| Command | Aliases | Description |
|---|---|---|
| `!settings` | `/settings`, `/cfg` | Open settings menu |
| `!lang` | `/language` | Open language selection sub-menu |

### 12.3 Admin Commands

| Command | Description | Min Role |
|---|---|---|
| `!a kick <player> [reason]` | Kick player | Admin |
| `!a ban <player> <duration> [reason]` | Ban player | Admin |
| `!a unban <steamid/account_id>` | Remove ban | Admin |
| `!a mute <player> <duration>` | Mute player in chat | Admin |
| `!a slay <player>` | Eliminate player | Admin |
| `!a slayteam <ct/t>` | Eliminate all players on a team | Admin |
| `!a forceseeker <player>` | Force player to CT next round | Admin |
| `!a setregion <player> <region>` | Override player region | Admin |
| `!a givexp <player> <amount>` | Grant XP | Admin |
| `!a givecosmetic <player> <id>` | Grant cosmetic item | Admin |
| `!a granttag <player> <tag_id>` | Grant FOUNDER or BETA tag | Admin |
| `!a invalidatejs <record_id>` | Invalidate a JumpStat record | Admin |
| `!a training start` | Start Training Mode without vote | Admin |
| `!a training stop` | End Training Mode, return to live | Admin |
| `!a auditlog <player>` | View player's XP audit log | Admin |
| `!a reloadconfig` | Hot-reload all configuration files | Admin |
| `!a reloadmap` | Reload current map metadata | Admin |
| `!a dbstatus` | Display database connection health | Owner |
| `!a reloadplugin <module>` | Hot-reload a specific plugin module | Owner |

---

## 13. MENUS

### 13.1 Menu Design Standards

- All menus use SourceMod VGUI panel or CSS-compatible chat menu.
- Navigable via number key input.
- Every menu includes a Back option and a Close option.
- All titles and items rendered through the Language System.
- Menus do not block gameplay.

### 13.2 Main Navigation Menu (`!menu`)

```
╔══════════════════════════╗
║     GARUDA MOVEMENT      ║
╠══════════════════════════╣
║  1. Profile              ║
║  2. Statistics           ║
║  3. Rankings             ║
║  4. Training             ║
║  5. JumpStats            ║
║  6. Cosmetics            ║
║  7. Settings             ║
║  8. Rules                ║
║  9. Discord              ║
║  0. Close                ║
╚══════════════════════════╝
```

### 13.3 Profile Menu

```
╔════════════════════════════════════╗
║  PROFILE — {player_name}          ║
╠════════════════════════════════════╣
║  Level   : {level} ({xp}/{xp_next}║
║  Rank     : #{rank}               ║
║  Survived : {survived} rounds     ║
║  Elims    : {kills}               ║
║  Best LJ  : {lj_pb} u            ║
╠════════════════════════════════════╣
║  1. Achievements                  ║
║  2. JumpStats                     ║
║  3. Match History                 ║
║  0. Back                          ║
╚════════════════════════════════════╝
```

### 13.4 Statistics Menu

```
╔══════════════════════════╗
║      STATISTICS          ║
╠══════════════════════════╣
║  1. Gameplay Stats       ║
║  2. JumpStats            ║
║  3. Training Records     ║
║  0. Back                 ║
╚══════════════════════════╝
```

### 13.5 Rankings Menu

```
╔══════════════════════════╗
║       RANKINGS           ║
╠══════════════════════════╣
║  1. Server Ranking       ║
║  2. JumpStats LB         ║
║  3. My Rank              ║
║  0. Back                 ║
╚══════════════════════════╝
```

### 13.6 JumpStats Leaderboard Sub-Menu

```
╔════════════════════════════════════╗
║   JUMPSTATS LEADERBOARD            ║
╠════════════════════════════════════╣
║  1. Longjump (LJ)                 ║
║  2. Countjump (CJ)                ║
║  3. Multi-Countjump (MCJ)         ║
║  4. Ladder Jump (LDJ)             ║
║  5. Double Bhop (DBhop)           ║
║  0. Back                          ║
╚════════════════════════════════════╝
```

#### 13.6.1 Leaderboard List View (example: LJ)

```
╔════════════════════════════════════╗
║   SERVER RECORD — LONGJUMP         ║
╠════════════════════════════════════╣
║  #1  PlayerName     261.42 u      ║
║  #2  AnotherPlayer  258.17 u      ║
║  #3  ThirdPlayer    255.90 u      ║
║  ...                              ║
║  [Page 1 / 3]                     ║
╠════════════════════════════════════╣
║  8. Prev Page  9. Next Page       ║
║  0. Back                          ║
╚════════════════════════════════════╝
```

### 13.7 Training Menu

```
╔══════════════════════════╗
║       TRAINING           ║
╠══════════════════════════╣
║  1. Free Movement        ║
║  2. JumpStats Practice   ║
║  3. Strafing Workshop    ║
║  4. Guided Challenges    ║
║  5. My Training Records  ║
║  0. Back                 ║
╚══════════════════════════╝
```

### 13.8 Cosmetics Menu

```
╔═══════════════════════════════════╗
║          COSMETICS                ║
╠═══════════════════════════════════╣
║  1. Knife    [Karambit]           ║
║  2. Trail    [Purple]             ║
║  3. Title    [Veteran]            ║
║  4. Tag      [TOP 10]  (auto)     ║
║  0. Back                          ║
╚═══════════════════════════════════╝
```

Tags assigned automatically (TOP 10, TOP 3) are displayed as read-only in the cosmetics menu with an `(auto)` indicator.

### 13.9 Settings Menu

```
╔══════════════════════════════╗
║          SETTINGS            ║
╠══════════════════════════════╣
║  1. Language    [Indonesian] ║
║  2. HUD         [Manage]     ║
║  3. JumpStats   [ON]         ║
║  4. Notifications [ON]       ║
║  5. Knife       [Karambit]   ║
║  6. Trail       [Purple]     ║
║  0. Back                     ║
╚══════════════════════════════╝
```

#### 13.9.1 HUD Sub-Menu

```
╔══════════════════════════╗
║         HUD              ║
╠══════════════════════════╣
║  1. JumpStats HUD  [ON]  ║
║  2. Speed HUD      [OFF] ║
║  3. Sync HUD       [OFF] ║
║  4. Strafe HUD     [OFF] ║
║  0. Back                 ║
╚══════════════════════════╝
```

---

## 14. LANGUAGE SYSTEM

### 14.1 Overview

The Language System provides complete localization for all player-facing strings. It is a mandatory component of every module. Hardcoded strings of any language in any module constitute a build-breaking violation.

### 14.2 Supported Languages — V1.0

| Code | Language | Status |
|---|---|---|
| `id` | Bahasa Indonesia | **Canonical** — all strings must exist |
| `en` | English | **Mandatory Fallback** — all strings must exist |

All other languages (Malay, Thai, Vietnamese, Filipino) are reserved for Phase 2 and must not be partially implemented in V1.0.

### 14.3 Translation File Format

Each module maintains its own SourceMod Translations phrases file. Naming convention: `gm_<module_name>.phrases.txt`.

Example structure:

```
"Phrases"
{
    "GM.Round.CountdownGo"
    {
        "id"    "GO!"
        "en"    "GO!"
    }

    "GM.Round.HiderWin"
    {
        "id"    "Para Hider menang! Selamat bertahan!"
        "en"    "Hiders win! Well survived!"
    }

    "GM.JumpStats.NewPB"
    {
        "id"    "PB baru! {jump_type}: {distance:2} u | Sync: {sync_pct}%"
        "en"    "New PB! {jump_type}: {distance:2} u | Sync: {sync_pct}%"
    }

    "GM.Achievement.Unlocked"
    {
        "id"    "[Pencapaian] {achievement_name} (+{xp} XP)"
        "en"    "[Achievement] {achievement_name} (+{xp} XP)"
    }

    "GM.Round.Countdown"
    {
        "id"    "{count}..."
        "en"    "{count}..."
    }

    "GM.FrostNade.Applied"
    {
        "id"    "FrostNade! {chaser_name} dibekukan."
        "en"    "FrostNade! {chaser_name} is frozen."
    }
}
```

### 14.4 Token Types

| Token Syntax | Type |
|---|---|
| `{string}` | Raw string substitution |
| `{integer}` | Formatted integer |
| `{float:N}` | Float with N decimal places |
| `{player}` | Player name (auto color-formatted) |

### 14.5 Fallback Behavior

- Player's preferred language missing a key → fall back to `id`.
- Key missing in all locales → render the raw key as an error, log CRITICAL.
- `id` is never missing for any key — this is enforced at build time.

### 14.6 Language Selection

Players select their preferred language via `!lang` command or Settings Menu. Default: `id` for all SEA-region players; `en` for all others.

---

## 15. RANKING SYSTEM

### 15.1 Overview

The Ranking System provides two independent ranking axes in V1.0: Gameplay Rank and Movement Rank. These are separate leaderboards; a player can excel in one without the other.

### 15.2 Ranking Axes — V1.0

| Axis | Basis | Leaderboard |
|---|---|---|
| Gameplay Rank | Win rate, survival rate, eliminations per round | Yes — server-wide |
| Movement Rank | JumpStats composite score across all 5 approved jump types | Yes — server-wide |
| Level Rank | Total XP (implicit — visible on profile, not a separate leaderboard) | No separate LB |

### 15.3 Gameplay Rank

#### 15.3.1 Eligible Rounds
All rounds in standard HNS Chase mode contribute to Gameplay Rank. Training Mode rounds do not.

#### 15.3.2 Rank Score Formula (Reference)
Implementation-defined. Must weight:
- Survival contribution (Hider rounds).
- Elimination contribution (Chaser rounds).
- Round win participation.
- Early elimination penalty (Hider eliminated in first 30 seconds of GO!).

All weights are configuration parameters defined in `gm_rank.cfg`.

#### 15.3.3 Rank Tiers

| Tier | Label | Badge Color |
|---|---|---|
| I | Newcomer | Grey |
| II | Amateur | Green |
| III | Skilled | Teal |
| IV | Expert | Blue |
| V | Master | Purple |
| VI | Grandmaster | Red |
| VII | Legend | Gold |
| VIII | Garuda | Animated |

#### 15.3.4 Rank Decay
**Removed from V1.0.** Rank decay is not implemented at launch. Rank scores accumulate without time-based decay. Decay mechanics are deferred to Phase 2.

### 15.4 Movement Rank

Movement Rank is a composite score derived from JumpStat PBs across all 5 approved jump types.

| Jump Type | Default Weight |
|---|---|
| Longjump (LJ) | 30% |
| Countjump (CJ) | 20% |
| Multi-Countjump (MCJ) | 20% |
| Ladder Jump (LDJ) | 15% |
| Double Bhop (DBhop) | 15% |

A player with no record in a jump type contributes 0 for that type, reducing their composite. This incentivizes completeness across all jump types.

Movement Rank uses the same tier labels as Gameplay Rank with independent score thresholds.

### 15.5 TOP 10 / TOP 3 Tags

The `TOP 10` and `TOP 3` chat tags (Section 9.6) are dynamically computed from the Gameplay Rank leaderboard. Tag assignment and removal are automatic and occur at rank score update events (end of round). No manual action is required.

### 15.6 Rank Display

- Scoreboard: Gameplay Rank badge.
- Profile panel: both Gameplay Rank and Movement Rank.
- Chat: Gameplay Rank tier displayed alongside player name (configurable, default on).

### 15.7 Rank History

A weekly rank snapshot per player is stored to the database. This infrastructure is required from V1.0 launch to enable Phase 2 season mechanics.

---

## 16. DATABASE REQUIREMENTS

### 16.1 Engine

| Requirement | Specification |
|---|---|
| Engine | MySQL 5.7+ or MariaDB 10.4+ |
| Character Set | utf8mb4 |
| Collation | utf8mb4_unicode_ci |
| Connection Mode | Persistent pool via SourceMod SQL |
| Min Pool Size | 4 connections |
| Max Pool Size | 16 connections |
| Query Timeout | 5 seconds (non-transactional), 10 seconds (transactional) |

### 16.2 Schema Design Principles

- All tables use UUID (v4) primary keys.
- `account_id` is the canonical player reference key across all tables. SteamID64 is a secondary nullable attribute on the accounts table.
- All timestamps are UTC, stored as `DATETIME` with microsecond precision.
- All foreign key relationships are declared and enforced.
- No hard-deletes without a soft-delete column or archival strategy.
- All tables carry `created_at` and `updated_at`.

### 16.3 Core Tables

#### 16.3.1 `accounts`
Fields: `account_id` (UUID PK), `steam_id_64` (nullable), `discord_id` (nullable), `lsgi_identifier` (nullable), `username`, `account_type` (enum: guest/linked_steam/linked_discord), `created_at`, `last_seen`, `region`, `preferred_language`, `total_playtime_seconds`, `is_banned`, `ban_reason`, `ban_expiry`, `updated_at`.

#### 16.3.2 `sessions`
Fields: `session_id` (UUID PK), `account_id` (FK), `connected_at`, `disconnected_at` (nullable), `ip_hash`, `server_id`.

#### 16.3.3 `preferences`
Fields: `preference_id` (UUID PK), `account_id` (FK), `pref_key`, `pref_value`, `updated_at`.

#### 16.3.4 `xp_transactions`
Fields: `transaction_id` (UUID PK), `account_id` (FK), `xp_amount`, `source_type` (enum: gameplay/jumpstat/training/achievement), `source_reference`, `round_id` (nullable FK), `created_at`.

#### 16.3.5 `player_levels`
Fields: `account_id` (FK, PK), `current_level`, `total_xp`, `updated_at`.

#### 16.3.6 `rounds`
Fields: `round_id` (UUID PK), `server_id`, `map_name`, `round_type`, `started_at`, `ended_at`, `winner_team`, `hider_count`, `chaser_count`.

#### 16.3.7 `round_participants`
Fields: `participation_id` (UUID PK), `round_id` (FK), `account_id` (FK), `team`, `survived` (boolean), `eliminations`, `time_alive_seconds`, `xp_earned`.

#### 16.3.8 `jumpstat_records`
Fields: `record_id` (UUID PK), `account_id` (FK), `jump_type` (enum: lj/cj/mcj/ldj/dbhop), `distance`, `pre_speed`, `land_speed`, `strafe_count`, `sync_pct`, `airtime_ticks`, `map_name`, `server_id`, `is_pb` (boolean), `is_server_record` (boolean), `recorded_at`, `is_valid` (boolean), `invalidated_by` (nullable), `invalidated_at` (nullable).

#### 16.3.9 `jumpstat_tick_data`
Fields: `tick_record_id` (UUID PK), `record_id` (FK to jumpstat_records), `tick_sequence` (integer), `pos_x`, `pos_y`, `pos_z`, `vel_x`, `vel_y`, `vel_z`, `recorded_at`.
Purpose: Position per tick during validated jumps, reserved for Phase 3 Jump Replay System.

#### 16.3.10 `achievements`
Fields: `achievement_id` (slug PK), `category`, `tier`, `name_key`, `description_key`, `xp_reward`, `cosmetic_reward_id` (nullable), `is_repeatable` (boolean), `is_hidden` (boolean), `condition_json`.

#### 16.3.11 `player_achievements`
Fields: `player_achievement_id` (UUID PK), `account_id` (FK), `achievement_id` (FK), `completed_at`, `times_completed`.

#### 16.3.12 `cosmetic_catalog`
Fields: `cosmetic_id` (slug PK), `category` (enum: knife/trail/title/tag), `name_key`, `description_key`, `acquisition_type`, `is_active` (boolean).

#### 16.3.13 `player_cosmetics`
Fields: `player_cosmetic_id` (UUID PK), `account_id` (FK), `cosmetic_id` (FK), `acquired_at`, `acquired_source`, `is_equipped` (boolean).

#### 16.3.14 `rank_scores`
Fields: `rank_score_id` (UUID PK), `account_id` (FK), `axis` (enum: gameplay/movement), `score`, `tier`, `updated_at`.

#### 16.3.15 `rank_history`
Fields: `history_id` (UUID PK), `account_id` (FK), `axis`, `score`, `tier`, `rank_position`, `snapshotted_at`.

#### 16.3.16 `admin_log`
Fields: `log_id` (UUID PK), `admin_account_id` (FK), `target_account_id` (FK, nullable), `action_type`, `action_detail`, `performed_at`.

#### 16.3.17 `friends` (Reserved — Phase 2)
Schema reserved. Fields: `friendship_id`, `requester_id`, `addressee_id`, `status`, `created_at`.

#### 16.3.18 `linked_accounts` (Reserved — Phase 2)
Schema reserved. Fields: `link_id`, `account_id`, `provider`, `provider_user_id`, `linked_at`.

### 16.4 Critical Indexes

- `accounts`: unique index on `steam_id_64` (where not null); unique index on `discord_id` (where not null); unique index on `lsgi_identifier` (where not null).
- `jumpstat_records`: composite index on `(jump_type, distance DESC, is_valid)` for leaderboard queries.
- `jumpstat_records`: index on `(account_id, jump_type, is_pb)`.
- `xp_transactions`: index on `(account_id, created_at)`.
- `round_participants`: index on `(round_id, account_id)`.
- `rank_scores`: index on `(axis, score DESC)`.

### 16.5 Backup Requirements

| Requirement | Value |
|---|---|
| Full backup | Daily, retained 30 days |
| Incremental backup | Every 6 hours, retained 7 days |
| Point-in-time recovery target | 15 minutes |
| Backup verification | Weekly restore test |
| Backup storage | Geographically separate from production |

---

## 17. PLUGIN ARCHITECTURE

### 17.1 Architecture Philosophy

GM-HNS follows a **modular monorepo** architecture. All modules share a single repository and a single shared library (`gm_core`) but compile to independent SourceMod plugin binaries (`.smx`). Each binary loads, unloads, and hot-reloads independently. The failure of any single module must not crash the server or terminate the game round.

### 17.2 Module Registry — V1.0

| Module ID | Binary | Responsibility | Hard Dependencies |
|---|---|---|---|
| `gm_core` | Library (.inc) | Event bus, DB pool, language resolver, config manager | None |
| `gm_account` | `gm_account.smx` | Account creation, session, preferences, NonSteam support | `gm_core` |
| `gm_region` | `gm_region.smx` | Region assignment, GeoIP lookup | `gm_core`, `gm_account` |
| `gm_movement` | `gm_movement.smx` | Scroll-jump validation, speed cap, boost detection | `gm_core` |
| `gm_team` | `gm_team.smx` | Team assignment, CT Queue, auto-balance, 70/30 ratio | `gm_core`, `gm_account` |
| `gm_gameplay` | `gm_gameplay.smx` | Round rules, freeze time, countdown, FrostNades, win conditions | `gm_core`, `gm_team`, `gm_movement` |
| `gm_jumpstats` | `gm_jumpstats.smx` | Jump detection, measurement, validation, leaderboard | `gm_core`, `gm_account`, `gm_movement` |
| `gm_xp` | `gm_xp.smx` | XP computation, level management | `gm_core`, `gm_account` |
| `gm_rank` | `gm_rank.smx` | Rank score, tier management, TOP 10/3 tag assignment | `gm_core`, `gm_account`, `gm_xp` |
| `gm_achievements` | `gm_achievements.smx` | Achievement evaluation, completion, rewards | `gm_core`, `gm_account`, `gm_xp` |
| `gm_cosmetics` | `gm_cosmetics.smx` | Cosmetic catalog, equip, knife/trail/title/tag rendering | `gm_core`, `gm_account` |
| `gm_training` | `gm_training.smx` | Training mode, vote system, checkpoint/teleport, challenges | `gm_core`, `gm_account`, `gm_jumpstats`, `gm_xp` |
| `gm_hud` | `gm_hud.smx` | All HUD overlays: JumpStats, Speed, Sync, Strafe, countdown | `gm_core`, `gm_jumpstats`, `gm_movement` |
| `gm_antiexploit` | `gm_antiexploit.smx` | Speed validation, teleport detection, scroll-jump integrity | `gm_core`, `gm_movement` |
| `gm_menu` | `gm_menu.smx` | All menu rendering and navigation | `gm_core` + all feature modules |
| `gm_commands` | `gm_commands.smx` | Command registration, routing, permission check | `gm_core` + all feature modules |
| `gm_admin` | `gm_admin.smx` | Admin commands, audit log, server management | `gm_core`, `gm_account` + all feature modules |

`gm_api` is **not a V1.0 module**. The API is a Phase 2 external service. V1.0 only requires the database to be structured to support future API access.

### 17.3 `gm_core` Shared Library

`gm_core` is a SourceMod include library (`.inc`) compiled into each dependent module.

#### 17.3.1 Event Bus
Named event system for decoupled inter-module communication. All event definitions are maintained in `gm_events.inc`. Modules may only fire or subscribe to declared events.

Core V1.0 event definitions:

| Event | Fired By | Payload |
|---|---|---|
| `GM_OnPlayerConnect` | `gm_account` | account_id |
| `GM_OnPlayerDisconnect` | `gm_account` | account_id, session_id |
| `GM_OnRoundStart` | `gm_gameplay` | round_id, map_name |
| `GM_OnCountdownGo` | `gm_gameplay` | round_id |
| `GM_OnRoundEnd` | `gm_gameplay` | round_id, winner_team |
| `GM_OnPlayerEliminated` | `gm_gameplay` | victim_id, attacker_id, round_id |
| `GM_OnPlayerSurvived` | `gm_gameplay` | account_id, round_id, time_alive |
| `GM_OnFrostNadeDetonated` | `gm_gameplay` | thrower_id, targets_hit, round_id |
| `GM_OnJumpRecorded` | `gm_jumpstats` | account_id, jump_type, distance, is_pb, is_sr |
| `GM_OnXPAwarded` | `gm_xp` | account_id, amount, source_type |
| `GM_OnLevelUp` | `gm_xp` | account_id, old_level, new_level |
| `GM_OnAchievementUnlocked` | `gm_achievements` | account_id, achievement_id |
| `GM_OnRankChanged` | `gm_rank` | account_id, axis, old_tier, new_tier |
| `GM_OnTrainingVoteStart` | `gm_training` | initiator_id |
| `GM_OnTrainingStart` | `gm_training` | — |
| `GM_OnTrainingEnd` | `gm_training` | — |

#### 17.3.2 Database Pool
Shared async connection pool. No module maintains its own connection.

- `GM_DB_QueryAsync(query, callback, data)` — Async query with callback.
- `GM_DB_TransactionAsync(queries[], callbacks[], data)` — Transactional multi-query.
- `GM_DB_Escape(str)` — SQL string escaping.
- `GM_DB_Status()` — Returns pool health enum.

#### 17.3.3 Language Resolver
- `GM_Lang_Format(client, key, ...)` — Returns localized formatted string for client.
- `GM_Lang_GetLocale(client)` — Returns player's locale code.

#### 17.3.4 Configuration Manager
All configuration in `gm_*.cfg` files. Hot-reloadable at runtime.

- `GM_Config_GetInt(key)`
- `GM_Config_GetFloat(key)`
- `GM_Config_GetString(key)`

### 17.4 Inter-Module Communication Protocol

Modules communicate through exactly three channels:

1. **Event Bus** — fire-and-forget, asynchronous, decoupled.
2. **Declared Natives** — synchronous cross-module calls; declared in `gm_natives.inc`; used sparingly.
3. **Database** — shared data store; modules may read other modules' tables via documented query patterns only.

Direct plugin-to-plugin function calls outside of declared natives are **prohibited**.

### 17.5 Plugin Load Order

```
gm_core (library) →
gm_account → gm_region → gm_movement →
gm_team → gm_gameplay → gm_jumpstats → gm_xp →
gm_rank → gm_achievements → gm_cosmetics → gm_training →
gm_hud → gm_antiexploit →
gm_menu → gm_commands → gm_admin
```

### 17.6 Configuration Files

| File | Module | Purpose |
|---|---|---|
| `gm_config.cfg` | All | Master server configuration |
| `gm_gameplay.cfg` | `gm_gameplay` | Round timers, FrostNade config, freeze duration |
| `gm_team.cfg` | `gm_team` | Team ratio, balance thresholds, queue config |
| `gm_jumpstats.cfg` | `gm_jumpstats` | Validation thresholds, jump type config |
| `gm_xp.cfg` | `gm_xp` | XP source values |
| `gm_rank.cfg` | `gm_rank` | Rank tier thresholds, weights |
| `gm_training.cfg` | `gm_training` | Vote thresholds, challenge definitions |
| `gm_maps/<mapname>.json` | `gm_gameplay` | Per-map metadata |

All configuration files are hot-reloadable via `!a reloadconfig`.

### 17.7 Logging

Each module writes to its own log: `logs/gm_<module>.log`. Log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL`.

`CRITICAL` events trigger a Discord webhook notification to the server admin channel.

---

## 18. API REQUIREMENTS

### 18.1 V1.0 Scope

A full external REST API is a **Phase 2 deliverable**. In V1.0, the following minimal API infrastructure is required:

1. **Database schema** is structured to be read-friendly by an external service without modification.
2. **Discord webhook integration** for server record and CRITICAL error notifications is implemented directly within `gm_admin` and `gm_jumpstats` via SourceMod HTTP.
3. **No public-facing API endpoints** are served from V1.0 server-side plugins.

### 18.2 V1.0 Discord Webhooks

The following outbound webhooks are implemented in V1.0:

| Event | Target | Payload |
|---|---|---|
| Server Record broken | Discord `#records` channel | Player name, jump type, distance, previous record |
| CRITICAL plugin error | Discord `#admin-alerts` | Module, error, timestamp |
| Player banned | Discord `#admin-log` | Admin, player, reason, duration |

Webhook delivery: at-least-once with exponential backoff retry (max 3 attempts). Webhook URLs are stored in `gm_config.cfg`.

### 18.3 Phase 2 API Specification (Deferred)

The full REST API specification — including public leaderboard endpoints, player profile endpoints, authenticated admin endpoints, JWT authentication, rate limiting, and web panel integration — is deferred to Phase 2. The architecture reservation is:

- API will be an external standalone service (not a SourceMod plugin) connecting to the production database.
- All Phase 2 API endpoints will be prefixed `/v1/`.
- Player identity in API contexts uses `account_id`, not SteamID64.
- Admin endpoints will require MFA-backed JWT authentication.

---

## 19. FUTURE ROADMAP

Features in this section are **explicitly out of V1.0 scope**. All V1.0 architectural decisions must accommodate these features without requiring schema migrations or plugin rewrites.

### Phase 2: Social, API, and Community (Target: 3 months post-launch)

| Feature | Description | V1.0 Infrastructure Required |
|---|---|---|
| External REST API | Full public and admin API | DB schema readable by external service |
| Web Panel v1 | Leaderboards, profiles, stats | API must be live |
| Discord Bot | Leaderboard queries, player lookups | API + Discord webhook |
| Discord Account Link | Link Discord to `account_id` | `linked_accounts` table reserved |
| Friends System | Add and manage friends | `friends` table reserved |
| Per-Country Leaderboards | Separate ID/SG/MY/PH/TH leaderboards | Regional system already partitioned |
| Account Merge | Merge Guest account into Steam-linked account | `account_id` as canonical key from V1.0 |
| Community Translations | Malay, Thai, Vietnamese, Filipino | Language system format already extensible |
| Rank Decay | Time-based rank score decay | `rank_history` table already recording |

### Phase 3: Prestige and Competition (Target: 6 months post-launch)

| Feature | Description | Dependency |
|---|---|---|
| Seasonal Ranking | Quarterly season resets with seasonal cosmetic rewards | 1+ seasons of rank history data |
| Jump Replay System | Replay jump attempts in-game | `jumpstat_tick_data` collected from V1.0 |
| Additional Jump Types | BLJ, DJ, WJ, HJ, SJ, MBH | JumpStats module already extensible |
| Monthly Leaderboard | Monthly JumpStats reset with archive | API + web panel |
| Custom Map Submission | Community map submission pipeline | Web panel |
| Additional Cosmetic Categories | Sprays, nameplates if community-requested | Cosmetics module extensible |

### Phase 4: Platform Expansion (Target: 9–12 months post-launch)

| Feature | Description |
|---|---|
| Multi-Server Federation | Shared rank and stats across multiple GM server instances |
| Clan / Team System | Registered clans with clan leaderboards |
| Inter-Region Events | Head-to-head SEA country competitions |
| Advanced Movement Analysis | Statistical anomaly detection on movement data (6+ months of baseline required) |
| Web Panel v2 | Full dashboard with personal analytics and match history |

### Permanently Excluded Features

The following features are **permanently excluded** from the Garuda Movement ecosystem and may not be re-introduced by any future specification revision without a formal project decision:

- AutoBhop in any live gameplay mode.
- Any paid or tiered XP multiplier or gameplay advantage.
- AI/bot-based training opponents.
- Microtransaction cosmetic economy.
- Character model / player skin replacement (CSS engine compatibility concerns).

---

## APPENDIX A: GLOSSARY

| Term | Definition |
|---|---|
| AutoBhop | Automated bunny hop via server-side jump key hold detection. Disabled in GM-HNS. |
| Chaser | CT-team player whose role is to hunt and eliminate Hiders. |
| CT Freeze | The 5-second period at round start during which Chasers are frozen in spawn. |
| DBhop | Double Bhop; two consecutive scroll-jump bhops, tracked as a jump type. |
| Duck Under | Movement technique: crouching under an obstacle at speed to maintain momentum. |
| Double Boost | Two teammates stacked to provide a two-level height boost. |
| FrostNade | A Hider-exclusive grenade that slows Chasers on detonation. |
| Head Boost | Using a teammate's head as a launch surface to gain additional vertical height. |
| Hider | T-team player whose role is to evade Chasers and survive the round. |
| HNS Chase | The specific Hide and Seek variant used by GM-HNS, with FrostNades and boost mechanics. |
| HU | Hammer Units; the native distance unit of the Source Engine. 1 HU ≈ 1.905 cm. |
| Human Ladder | A vertical stack of teammates used to reach elevated positions. |
| JumpStat | A server-recorded and validated movement measurement. |
| LDJ | Ladder Jump; jump initiated from ladder-acquired momentum. |
| MCJ | Multi-Countjump; consecutive CJ chain, measured per landing. |
| PB | Personal Best; the player's best recorded distance for a jump type. |
| Pre-strafe | Ground acceleration through directional key input before jumping. |
| Scroll Jump | Jump input executed via scroll wheel or manual key press (not hold). The only valid jump technique in GM-HNS. |
| SR | Server Record; the all-time best on this server for a jump type. |
| Sync % | Percentage of airtime during which strafe key direction and mouse movement are aligned. |
| Tick | A single server simulation step; at 66 tick, approximately 15.15ms per tick. |
| Underblock | Movement technique: using map geometry to pass under surfaces at speed. |
| XP | Experience Points; the sole progression currency. |

---

## APPENDIX B: CHANGE LOG FROM ORIGINAL SPECIFICATION

| Change ID | Category | Original | V1.0 Final |
|---|---|---|---|
| C-01 | Gameplay | AutoBhop assumed active | Permanently disabled server-side |
| C-02 | Gameplay | 8s freeze, 3s blind default | 5s CT Freeze, 5..4..3..2..1..GO countdown |
| C-03 | Gameplay | Ghost Round, Blind Seekers special rounds | Removed — out of V1.0 scope |
| C-04 | Gameplay | Missing HNS Chase mechanics | Added: FrostNades, Head/Double Boost, Human Ladder, Duck Under, Underblock |
| C-05 | Teams | 50/50 default ratio | 70% Hider / 30% Chaser |
| C-06 | Teams | "Forced Seeker" | CT Queue System |
| C-07 | Training | AI/Bot Training, Seeker AI | Removed entirely |
| C-08 | Training | No vote system | Vote required for multi-player Training Mode |
| C-09 | Training | Missing commands | Added: !cp, !tp, !play |
| C-10 | JumpStats | 9 jump types | 5 approved types: LJ, CJ, MCJ, LDJ, DBhop |
| C-11 | JumpStats | Monthly leaderboard | Deferred to Phase 3 |
| C-12 | XP | VIP/Party/Event multipliers | Removed — P-09 equal gameplay |
| C-13 | XP | Flawless Round XP source | Removed — not feasible in V1.0 |
| C-14 | Achievements | "Phantom" survival achievement | Removed with Flawless Round |
| C-15 | Cosmetics | Skins, Nameplates, Sprays, Kill Sounds, Name Colors | All removed from V1.0 |
| C-16 | Cosmetics | No knife system | Added: 7 approved knife skins |
| C-17 | Cosmetics | Generic trail system | 12 approved colors + Off |
| C-18 | Cosmetics | Generic titles | 5 approved titles + Training Tier Badges |
| C-19 | Cosmetics | No tags | Added: TOP 10, TOP 3 (auto), FOUNDER, BETA |
| C-20 | Account | SteamID64 as canonical key | account_id (UUID) as canonical; Steam + NonSteam supported |
| C-21 | Account | No Guest account | Guest / Linked Steam / Linked Discord |
| C-22 | Account | VIP/VIP+ tiers with XP boost | Removed — no gameplay tiers in V1.0 |
| C-23 | Monetization | Future microtransaction economy | Permanently excluded |
| C-24 | Server | Max 32, no recommended value | Recommended 24, Hard Limit 32 |
| C-25 | Menu | Different structure | Aligned to approved 9-item main menu |
| C-26 | Settings Menu | Missing Knife/Trail options | Added to Settings Menu |
| C-27 | Regional | 11 global regions | SEA-focused: ID/SG/MY/PH/TH/SEA_OTHER/GLOBAL |
| C-28 | Ranking | Rank decay | Removed from V1.0 |
| C-29 | Ranking | 4 ranking axes | 2 axes: Gameplay Rank + Movement Rank |
| C-30 | Database | SteamID64 as primary reference | account_id (UUID) as primary reference |
| C-31 | Plugin | gm_api as V1.0 module | API is Phase 2; only webhooks in V1.0 |
| C-32 | Roadmap | Microtransaction economy | Permanently excluded |
| C-33 | Roadmap | ML Anti-cheat framing | Repositioned as statistical anomaly detection in Phase 4 |

---

## APPENDIX C: VERSION HISTORY

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0.0 | June 2026 | Lead Technical Architect | Initial draft specification |
| 1.0 FINAL | June 2026 | Lead Technical Architect | Full architecture review; aligned to approved V1.0 project decisions; 33 conflicts resolved |

---

*End of GM_HNS_MASTER_SPEC_V1_FINAL.md*

---

> **This document is the official V1.0 project contract for the Garuda Movement HNS Ecosystem.**
> All implementation decisions must reference and conform to this specification.
> The original GM_HNS_MASTER_SPEC.md is superseded in its entirety.
> Deviations require a formal specification amendment with version increment.
