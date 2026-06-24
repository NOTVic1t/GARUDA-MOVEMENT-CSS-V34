# GM_HNS_MASTER_SPEC.md
## Garuda Movement — Hide and Seek Ecosystem
### Counter-Strike: Source v34 | Master Technical Specification
### Version 1.0.0 | June 2026

---

> **Document Status:** OFFICIAL PROJECT CONTRACT  
> **Classification:** Internal Architecture & Design  
> **Maintainer:** Lead Technical Architect, Garuda Movement  
> **Scope:** Production-Grade Modular HNS Ecosystem for CSS v34

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

The Garuda Movement HNS Ecosystem (GM-HNS) is a production-grade, modular server-side framework for Counter-Strike: Source v34, engineered to deliver a competitive, community-driven Hide and Seek experience at parity with leading European servers (Cybershoke, CS-Dream) while bearing a distinctly Southeast Asian identity and operational context.

The project is not a simple plugin. It is a fully integrated ecosystem: a living system of interdependent modules united by a shared data layer, a coherent design language, and a long-term commitment to player retention.

### 1.2 Vision Statement

To build Southeast Asia's most technically advanced and community-respected HNS platform on Counter-Strike: Source — attracting competitive players, content creators, and movement enthusiasts through rigorous gameplay systems, transparent progression, and a culture of technical excellence.

### 1.3 Target Audience

| Segment | Description |
|---|---|
| Movement Specialists | Players focused on bhop/air movement skill mastery |
| HNS Competitors | Players invested in ranked survival gameplay |
| Achievement Hunters | Players driven by long-term progression goals |
| Community Regulars | Players seeking social, low-barrier daily play |
| Streamers / Content Creators | Players for whom cosmetic identity and clip potential matter |

### 1.4 Competitive Benchmarks

The following servers are studied as reference implementations. GM-HNS must meet or exceed their feature sets while introducing original systems:

- **Cybershoke HNS** — European market leader; reference for JumpStats, ranking tiers, and plugin stability.
- **CS-Dream** — Reference for movement training systems and community retention mechanics.
- **HNS.gg Network** — Reference for progression architecture and cosmetic economy.

### 1.5 Core Differentiators

- Modular plugin architecture allowing independent deployment per feature.
- A fully localized Language System (Indonesian-first with English fallback).
- A Regional System that groups players by geography for leaderboard segmentation.
- An integrated Training Mode distinct from live servers.
- A transparent, multi-axis Ranking System.

---

## 2. DESIGN PRINCIPLES

### 2.1 Foundational Principles

The following principles are binding on all implementation decisions. Any system or feature that violates a principle must be escalated for architectural review before proceeding.

#### P-01: Modularity First
Every feature domain is a discrete plugin module. No module shall be coupled to another by hard dependency at the code level. Inter-module communication occurs exclusively through defined event interfaces and the shared API layer.

#### P-02: Data Integrity Above All
Player data — experience, ranks, stats, achievements — is sacred. No operation that modifies player data may proceed without a recoverable write path. All writes are transactional. Data loss is classified as a Severity-1 incident.

#### P-03: Performance is a Feature
The server runs on CSS v34 at 66 tick. All modules must operate within a CPU budget that does not degrade server tickrate. Expensive operations (database writes, stat computation) are performed asynchronously. No module may block the game thread.

#### P-04: Transparency to the Player
Every system the player interacts with must be explainable in-game. Ranking formulas, XP rewards, and cooldown timers must be discoverable via menus or commands. Hidden mechanics are prohibited.

#### P-05: Fairness and Anti-Exploit
Gameplay-affecting systems (movement validation, JumpStats recording, XP rewards) must include server-side verification. Client-authoritative data is never trusted. All stat records must be replayable or auditable.

#### P-06: Graceful Degradation
If a module fails (database timeout, plugin error), the server must remain playable. Non-critical features (cosmetics, stat recording) failing must not terminate the round or disconnect players.

#### P-07: Localization by Default
All player-facing strings are defined in the Language System. No hardcoded English strings in any module. The Indonesian locale is canonical; English is the mandatory fallback.

#### P-08: Extensibility over Completeness
Ship fewer systems done correctly than many systems done poorly. Every system is designed to accept future extension without requiring schema migrations or plugin rewrites.

### 2.2 Technical Constraints

| Constraint | Value |
|---|---|
| Engine | Counter-Strike: Source v34 (HL2MP base) |
| Server Framework | SourceMod 1.11+ |
| Tick Rate | 66 tick |
| Max Players | 32 |
| Database Layer | MySQL 5.7+ or MariaDB 10.4+ |
| Min Supported Client | CSS v34 (no OB engine assumptions) |

---

## 3. GAMEPLAY RULES

### 3.1 Game Mode Overview

Hide and Seek (HNS) is a pursuit-and-evasion game mode built on the Counter-Strike: Source round framework. Terrorists (T) are designated **Hiders**; Counter-Terrorists (CT) are designated **Seekers**.

The central mechanic is movement: Hiders use bhop (bunny hopping) and advanced air movement to survive; Seekers use the same movement toolkit to hunt and eliminate Hiders before the round timer expires.

### 3.2 Win Conditions

| Outcome | Condition | Winner |
|---|---|---|
| Hider Victory | At least one Hider survives to round end | Hiders (T) |
| Seeker Victory | All Hiders are eliminated before round end | Seekers (CT) |
| Draw | Round ends with zero Hiders alive and zero Seeker eliminations | No score awarded |

### 3.3 Round Structure

#### 3.3.1 Pre-Round Phase (Freeze Time)
- Duration: configurable per server (default: 8 seconds).
- Hiders are frozen; Seekers are confined to spawn.
- No weapon fire permitted.
- Hiders receive a heads-up display showing round parameters (round timer, map name, player count).

#### 3.3.2 Seeker Blind Phase
- Duration: configurable (default: 3 seconds after freeze time).
- Seekers receive a screen flash or timed blindness at round start.
- Purpose: grants Hiders a movement head-start on larger maps.
- Configurable per map via map metadata.

#### 3.3.3 Active Phase
- Duration: configurable per map (default: 3 minutes).
- Standard HNS gameplay.
- Round timer visible to all players.
- Death is immediate and non-respawnable within the round.

#### 3.3.4 Post-Round Phase
- Duration: 5 seconds.
- Round result displayed (winner, last Hider alive, top killer, survival time).
- Stat snapshots committed to the database.
- XP and achievement triggers evaluated.

### 3.4 Hider Rules

- **Weapons:** Knife only. No firearms.
- **Movement:** All movement techniques permitted (bhop, strafe, longjump).
- **Objective:** Survive to round end.
- **Zone Restrictions:** Hiders may not camp inside spawn zones beyond a configurable timer (default: 15 seconds post-freeze). Violation: auto-slay.
- **Crouch Exploit Prevention:** Crouch-spamming in confined geometry to prevent detection is a detectable exploit. The anti-exploit module may restrict crouch while Hider is stationary beyond a configurable threshold.

### 3.5 Seeker Rules

- **Weapons:** All standard CT loadout. Configurable per server.
- **Movement:** All movement techniques permitted.
- **Objective:** Eliminate all Hiders.
- **Spawn Restriction:** Seekers may not leave spawn during Seeker Blind Phase. Violation: auto-slay.
- **Camping Restriction:** Seekers may not camp Hider spawn exits beyond a configurable timer. Violation: warning then slay.

### 3.6 Special Round Types

Special round types are toggled by vote or server admin and modify the standard ruleset.

| Round Type | Description |
|---|---|
| Normal | Standard ruleset. |
| Speed Round | Round timer reduced by 50%. |
| Blind Seekers | Seekers receive extended blindness (10 seconds). |
| No Crouch | All player crouching disabled server-wide. |
| Ghost Round | Seekers are invisible until within 300 units of a Hider. |
| Practice | No XP, no stats recorded. Used for training or warmup. |

### 3.7 Map Requirements

Maps integrated into the GM-HNS rotation must satisfy the following criteria:

- No exploitable out-of-bounds geometry accessible to either team.
- Distinct spawn zones for T and CT.
- Minimum three navigable paths between spawn zones.
- Compatibility declaration for all Special Round Types.
- Metadata file accompanying each map (see Section 11.3 for map metadata schema).

### 3.8 Anti-Cheat and Anti-Exploit Policy

- **Speed Hack Detection:** Server-side velocity validation against engine movement caps.
- **Teleport Detection:** Player position delta validation per tick. Anomalies flagged.
- **Stat Manipulation:** JumpStats are validated against server-computed trajectory. Client-reported values are not used.
- **Exploit Reporting:** Players may use `!bug` to report map exploits; reports logged with position and round context.

---

## 4. TEAM SYSTEM

### 4.1 Overview

The Team System manages player assignment, team balancing, role swapping, and applies team-specific buffs or modifiers during gameplay.

### 4.2 Team Definitions

| Team | CSS Team | HNS Role | Visual Identity |
|---|---|---|---|
| Hiders | Terrorist | Evade and survive | Configurable model set (see Cosmetics) |
| Seekers | Counter-Terrorist | Hunt and eliminate | Configurable model set |

### 4.3 Team Assignment

#### 4.3.1 Initial Assignment
- Players connecting to the server are assigned to whichever team has fewer players.
- If teams are equal, the player is assigned to Hiders.
- Players who were Seekers in the previous round are eligible for automatic role swap to Hider.

#### 4.3.2 Forced Seeker Assignment
- A player may be designated **Forced Seeker** if:
  - They are the only online player with Seeker eligibility.
  - An admin applies the designation manually.
  - A queue-based rotation places them (see Section 4.4).

#### 4.3.3 Auto-Balance Rules
- Auto-balance triggers at round end if the team size differential exceeds a configurable threshold (default: 3 players).
- Players with the lowest current-round contribution score (kills for Seekers, survival time for Hiders) are candidates for swap.
- Players may not be auto-swapped on consecutive rounds.

### 4.4 Seeker Queue System

The Seeker Queue ensures fair rotation of the Seeker role across a session.

- The queue is persistent per-player per-session (not across reconnects by default; configurable).
- Queue position is awarded based on time spent as Hider consecutively.
- Longest-waiting Hider is offered the Seeker role at round end.
- Player may decline once per session without losing queue position.
- Declined players are moved to the back of the queue on second refusal.

### 4.5 Team Modifiers

Modifiers are passive multipliers or adjustments applied to a team under configurable conditions.

| Modifier | Trigger Condition | Effect |
|---|---|---|
| Underdog Boost | Hiders < 30% of server population | +15% XP for Hider survival |
| Last Stand | 1 Hider remaining | Last Hider is revealed on minimap to all Seekers |
| Swarm | Seekers > 70% of server population | Seeker XP multiplier reduced by 10% |

### 4.6 Spectator System

- Players may spectate freely during a round.
- Spectators cannot communicate via team-restricted chat.
- Spectator duration longer than 3 consecutive rounds triggers an AFK check.
- AFK players (non-responsive to check) are kicked after configurable timeout (default: 5 minutes).

---

## 5. TRAINING SYSTEM

### 5.1 Overview

The Training System is a dedicated, isolated game mode running on a separate server instance or as a server-switch accessible from the main server. It allows players to practice movement mechanics, study JumpStats technique, and receive guided skill instruction without affecting their ranked stats.

### 5.2 Training Modes

#### 5.2.1 Free Bhop Mode
- Infinite autobhop enabled.
- No round timer.
- No opponents.
- JumpStats recorded to a separate training leaderboard (not merged into main stats).
- Velocity HUD displayed at all times.

#### 5.2.2 Strafing Workshop
- A curated map set with strafing zones and accuracy targets.
- Strafing efficiency score computed per zone (based on sync percentage and air acceleration gain).
- Scores stored to training records.
- Recommended for players below Rank 10 (see Section 15).

#### 5.2.3 JumpStats Practice
- Map set designed with measured jump surfaces.
- Ground-truth distances embedded in map geometry.
- Each jump type (longjump, bhop, dropjump, etc.) has a dedicated zone.
- Personal bests tracked against training leaderboard.
- Does not contribute to main JumpStats leaderboard.

#### 5.2.4 Guided Challenges
- Structured skill challenges with pass/fail evaluation.
- Completion triggers XP reward (one-time per challenge, per account).
- Challenges grouped by skill tier (Beginner, Intermediate, Advanced, Elite).

| Tier | Example Challenge | XP Reward |
|---|---|---|
| Beginner | Complete a bhop chain of 5 without landing | 100 XP |
| Intermediate | Achieve 240 u/s average over a 10-second bhop run | 300 XP |
| Advanced | Land a longjump of 240+ units | 500 XP |
| Elite | Land a bhop longjump of 270+ units | 1000 XP |

#### 5.2.5 Seeker Training
- Simulated AI (bot-based) Hider movement patterns for Seeker practice.
- Practice aim and approach angles at configured difficulty levels.
- Does not affect main stats.

### 5.3 Training Session Records

Each training session is recorded with:
- Session start/end timestamp.
- Mode used.
- Total jump count.
- Best JumpStat per type.
- Strafing sync % average.
- Challenges completed.

Sessions are retained for 90 days before archival.

### 5.4 Training Completion Rewards

The Training System feeds into the Achievement System (Section 8). Completing all challenges in a tier unlocks a Training Tier Badge cosmetic (Section 9). XP rewards from Training Challenges are non-repeatable but stackable across tiers.

---

## 6. JUMPSTATS SYSTEM

### 6.1 Overview

The JumpStats System is the core technical prestige system of GM-HNS. It records, validates, and ranks player movement feats (jumps) with precision and transparency. This system is modeled on the CS-Dream and Cybershoke JumpStats implementations, extended with regional segmentation and historical record archival.

### 6.2 Tracked Jump Types

| Jump Type | Code | Description |
|---|---|---|
| Longjump | LJ | Ground-to-ground jump from standing pre-strafe |
| Bhop Longjump | BLJ | Ground-to-ground jump from bhop chain |
| Multi Bhop | MBH | Consecutive bhop landings counted per chain |
| Dropjump | DJ | Jump from elevated surface (drop to ground) |
| Countjump | CJ | Double-tap jump off crouch, measured on landing |
| Weirdjump | WJ | Jump with full crouch during airtime |
| Ladderjump | LDJ | Jump initiated from ladder momentum |
| Higherjump | HJ | Vertical height achievement from ground jump |
| Stairjump | SJ | Jump from descending stairs geometry |

### 6.3 Measurement Specification

All measurements are in Hammer Units (HU), the CSS engine's native unit. 1 HU ≈ 1.905 cm in-game distance.

| Metric | Description |
|---|---|
| Distance | Horizontal ground-to-ground (or ground-to-surface) displacement |
| Height | Peak vertical displacement from launch surface |
| Speed (Pre-jump) | Velocity at moment of jump input |
| Speed (Landing) | Velocity at moment of landing detection |
| Strafes | Count of directional strafe inputs during airtime |
| Sync % | Percentage of airtime during which mouse movement and key input were synchronized |
| Airtime | Duration in server ticks from liftoff to landing |
| Block Height | Height of the surface jumped from (for DJ and HJ) |

### 6.4 Validation Rules

To prevent record fraud and physics exploits, all jumps must pass validation before being committed to the leaderboard.

#### 6.4.1 Velocity Validation
- Pre-jump velocity must not exceed the server-enforced ground speed cap.
- Mid-air velocity must conform to the air acceleration model for the server's configured `sv_airaccelerate` value.
- Any tick where velocity delta exceeds the physics-possible maximum is flagged as an anomaly; the jump is discarded.

#### 6.4.2 Trajectory Validation
- The jump arc (position per tick) must be consistent with a ballistic trajectory given measured velocity and sv_gravity.
- Teleports (position delta > theoretical maximum per tick) discard the jump.

#### 6.4.3 Surface Validation
- Launch surface must be a valid ground plane (not a player hitbox, not a moving brush if not declared in map metadata).
- Landing surface must be within declared valid landing zones for the jump type.

#### 6.4.4 Pre-Strafe Validation
- For LJ specifically: pre-strafe velocity must be reached from a standing start within the last N ticks (configurable, default 20 ticks). Pre-existing bhop velocity invalidates a LJ (classifies it as BLJ).

### 6.5 JumpStat Record Tiers

Records are classified by quality bracket for display and rewards:

| Tier | Color | Longjump Range (Example) | Notes |
|---|---|---|---|
| World Record | Gold | Server all-time #1 | One per jump type per region |
| National Record | Purple | Country #1 | One per jump type per country |
| Personal Best | Blue | Player's best | Per player per jump type |
| Good | Green | Top 25% of recorded jumps | |
| Normal | White | Within statistical range | |

### 6.6 JumpStat Leaderboards

#### 6.6.1 Global Leaderboard
- All-time best per jump type.
- Paginated, top 100 visible in-game.
- Full leaderboard accessible via web panel.

#### 6.6.2 Regional Leaderboard
- Segmented by region (see Section 11).
- Top 50 per jump type per region.

#### 6.6.3 Monthly Leaderboard
- Resets at the start of each calendar month.
- Retains a historical archive per month (permanent).
- Monthly leaderboard winners receive cosmetic rewards (Section 9).

#### 6.6.4 Personal Stats Panel
- Accessible via `!js` or `!jumpstats` command.
- Displays all-time PBs per jump type, rank on each leaderboard, recent 10 jumps.

### 6.7 Record Announcements

The following events trigger a server-wide announcement:

| Event | Announcement Scope |
|---|---|
| World Record broken | Server-wide + Discord webhook |
| National Record broken | Server-wide |
| Personal Best improved | Player-visible + chat notice |

### 6.8 Jump Replay System (Future, see Section 19)

Specification deferred to roadmap. Infrastructure (position recording per tick during jump) must be implemented now to enable future replay playback.

---

## 7. XP AND LEVEL SYSTEM

### 7.1 Overview

The XP and Level System provides a horizontal progression layer that rewards sustained participation across all game modes. XP is a single currency earned through gameplay, training, and achievement completion. Levels are milestone markers with cosmetic and unlock significance.

### 7.2 XP Sources

#### 7.2.1 Gameplay XP

| Event | Condition | Base XP |
|---|---|---|
| Round Survival | Hider survives to round end | 50 XP |
| Elimination | Seeker eliminates a Hider | 30 XP per kill |
| Last Survivor | Hider is the final remaining | 100 XP bonus |
| First Blood | First elimination of the round | 20 XP bonus |
| Flawless Round | Hider survives without being found (no line-of-sight from Seeker) | 80 XP bonus |
| Round Win (team) | Member of winning team | 10 XP bonus |
| Round Participation | Alive at any point during round | 10 XP (participation floor) |

#### 7.2.2 JumpStats XP

| Event | Condition | XP |
|---|---|---|
| Personal Best (any jump) | Improved PB in any jump type | 25 XP |
| Regional Record | Set a new regional record | 200 XP |
| World Record | Set a new world record | 500 XP |
| Monthly #1 | Finish #1 on monthly JumpStats leaderboard (end-of-month snapshot) | 300 XP |

#### 7.2.3 Training XP

| Event | Condition | XP |
|---|---|---|
| Challenge Completion | First-time completion of any guided challenge | Per challenge table (Section 5.2.4) |
| Session Milestone | Complete 50 tracked jumps in a training session | 50 XP |

#### 7.2.4 Achievement XP
Achievement completion rewards vary per achievement. Defined in the Achievement Catalog (Section 8.3).

#### 7.2.5 Social XP (Bonus Multipliers)

| Source | Multiplier | Condition |
|---|---|---|
| Daily Login | +10% XP for first 30 minutes of play | Log in and play within 24h of last session |
| Weekly Streak | +5% per consecutive week (cap: 30%) | Log in and play at least once per week |
| Party Bonus | +5% XP per party member (cap: 3 members) | Playing with registered friends |

### 7.3 XP Modifiers

All XP modifiers are additive before applying to base XP.

| Modifier | Value | Source |
|---|---|---|
| VIP Status | +20% | Account tier (Section 10) |
| Weekend Event | +25% | Server admin-activated event |
| Double XP Event | +100% | Server admin-activated event |
| Underdog Bonus | +15% | Team modifier (Section 4.5) |

### 7.4 Level Thresholds

Levels use an exponential curve. The formula is implementation-defined but must satisfy:
- Level 1 → Level 2 requires approximately 500 XP.
- Each subsequent level requires approximately 15% more XP than the previous.
- Level cap: 100 (extendable in future roadmap).
- Total XP to reach Level 100: approximately 2,500,000 XP (reference figure; exact value determined during implementation).

| Level Range | Label | Visual Indicator |
|---|---|---|
| 1–10 | Rookie | Grey badge |
| 11–25 | Player | Green badge |
| 26–50 | Veteran | Blue badge |
| 51–75 | Elite | Purple badge |
| 76–99 | Legend | Gold badge |
| 100 | Garuda | Animated Garuda emblem |

### 7.5 Level Rewards

Each level unlock delivers at least one of the following reward types:

- Cosmetic item (trail, trail color, chat tag, model skin).
- Title unlock.
- Command unlock (e.g., access to extended stats panel).
- Emote or spray unlock.

Level 100 unlocks the Garuda Title (permanently displayed) and a unique animated nameplate.

### 7.6 XP Audit Log

Every XP transaction is logged with:
- Player SteamID64.
- XP amount.
- Source type (gameplay / jumpstat / training / achievement / event).
- Timestamp.
- Round ID (if applicable).

Audit logs are retained for 180 days and are queryable by server admin.

---

## 8. ACHIEVEMENT SYSTEM

### 8.1 Overview

The Achievement System defines a catalog of player goals that reward engagement across all system domains: gameplay, movement, progression, social, and exploration. Achievements are non-repeatable unless explicitly marked as repeatable in their definition.

### 8.2 Achievement Structure

Each achievement is defined by:

| Field | Type | Description |
|---|---|---|
| `achievement_id` | String (slug) | Unique identifier, e.g., `hns_first_survival` |
| `name` | Localization key | Display name |
| `description` | Localization key | Condition description |
| `category` | Enum | See Section 8.3 |
| `tier` | Enum | Bronze / Silver / Gold / Platinum |
| `xp_reward` | Integer | XP awarded on completion |
| `cosmetic_reward` | Optional string | Cosmetic item unlocked on completion |
| `condition` | Structured object | Machine-readable condition definition |
| `repeatable` | Boolean | Whether the achievement can be earned again |
| `hidden` | Boolean | If true, not shown until earned |

### 8.3 Achievement Categories and Catalog

#### 8.3.1 Survival Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `surv_first` | First Blood (Green) | Survive your first round | Bronze | 50 |
| `surv_10` | Persistent | Survive 10 rounds total | Bronze | 100 |
| `surv_100` | Tenacious | Survive 100 rounds total | Silver | 500 |
| `surv_1000` | Immortal | Survive 1,000 rounds | Gold | 2,000 |
| `surv_streak_5` | Five Alive | Survive 5 consecutive rounds | Silver | 300 |
| `surv_streak_20` | Ghost Protocol | Survive 20 consecutive rounds | Gold | 1,000 |
| `surv_last_10` | Last Man x10 | Be the last Hider surviving 10 times | Silver | 500 |
| `surv_perfect` | Phantom | Survive without being spotted in line-of-sight (verified by server) 5 times | Gold | 800 |

#### 8.3.2 Hunter Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `hunt_first_kill` | First Hunt | Get your first elimination as Seeker | Bronze | 50 |
| `hunt_50` | Tracker | Eliminate 50 Hiders total | Bronze | 200 |
| `hunt_500` | Hunter | Eliminate 500 Hiders total | Silver | 1,000 |
| `hunt_5000` | Apex Predator | Eliminate 5,000 Hiders total | Gold | 5,000 |
| `hunt_flawless` | Clean Sweep | Eliminate all Hiders in one round (as part of Seeker team) | Silver | 400 |
| `hunt_solo` | One-Man Army | Eliminate all Hiders in one round alone | Gold | 1,500 |
| `hunt_speedkill` | Blink | Eliminate a Hider within 5 seconds of round start | Silver | 300 |

#### 8.3.3 Movement Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `mv_first_lj` | First Leap | Record any Longjump | Bronze | 50 |
| `mv_lj_220` | Strider | Land a Longjump of 220+ units | Bronze | 100 |
| `mv_lj_240` | Jumper | Land a Longjump of 240+ units | Silver | 300 |
| `mv_lj_260` | Flier | Land a Longjump of 260+ units | Gold | 1,000 |
| `mv_lj_270` | Transcendent | Land a Longjump of 270+ units | Platinum | 3,000 |
| `mv_bhop_chain_10` | Rhythm | Complete a 10-bhop chain | Bronze | 100 |
| `mv_bhop_chain_50` | Flow State | Complete a 50-bhop chain | Silver | 500 |
| `mv_all_jumptypes` | Polymath | Record a PB in every jump type | Gold | 2,000 |
| `mv_wr` | World Beater | Hold any World Record | Platinum | 5,000 |

#### 8.3.4 Progression Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `prog_level_10` | Rising | Reach Level 10 | Bronze | 200 |
| `prog_level_50` | Established | Reach Level 50 | Silver | 1,000 |
| `prog_level_100` | Garuda | Reach Level 100 | Platinum | 10,000 |
| `prog_rank_top10` | Elite Class | Reach the top 10 on the Regional Ranking | Gold | 2,000 |
| `prog_rank_1` | The Best | Reach Rank 1 on any leaderboard | Platinum | 5,000 |

#### 8.3.5 Social Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `social_10_sessions` | Regular | Play in 10 separate sessions | Bronze | 100 |
| `social_100_sessions` | Dedicated | Play in 100 separate sessions | Silver | 500 |
| `social_party` | Squad Up | Play in a party of 3+ friends | Bronze | 100 |
| `social_refer` | Recruiter | Refer a new player who reaches Level 5 | Silver | 500 |

#### 8.3.6 Training Achievements

| ID | Name | Condition | Tier | XP |
|---|---|---|---|---|
| `train_beginner` | First Steps | Complete all Beginner challenges | Bronze | 200 |
| `train_intermediate` | Student | Complete all Intermediate challenges | Silver | 500 |
| `train_advanced` | Practitioner | Complete all Advanced challenges | Gold | 1,500 |
| `train_elite` | Master | Complete all Elite challenges | Platinum | 5,000 |

#### 8.3.7 Hidden Achievements

Hidden achievements are not listed in the in-game catalog until earned. They exist to reward discovery and exploration.

- At least 10 hidden achievements must be defined in the launch catalog.
- Each hidden achievement carries a minimum Gold tier reward.
- Examples (not exhaustive): surviving on a server with 30+ players, landing a WJ in a live round, playing at a specific real-world time.

### 8.4 Achievement Evaluation

- Achievements are evaluated server-side at the end of each round, on JumpStat commit, and on session events (login, logout).
- Evaluation is asynchronous and does not block the game thread.
- On completion, a notification is shown to the player (chat + HUD announcement).
- Repeatable achievements reset their counter after each completion.

---

## 9. COSMETICS SYSTEM

### 9.1 Overview

The Cosmetics System manages all player-visible personalization items. Cosmetics are non-gameplay-affecting: no cosmetic may confer a movement advantage, visibility advantage, or stat benefit beyond the XP/Level modifiers explicitly defined in Section 7.

### 9.2 Cosmetic Categories

| Category | Description | Examples |
|---|---|---|
| Player Skin | Model skin applied to the player character | Garuda Warrior, Shadow, Neon |
| Trail | Visual particle effect following the player during movement | Flame, Starfall, Void Smoke |
| Chat Tag | Colored tag prefix displayed in chat | [GM], [Elite], [Garuda] |
| Name Color | Chat name color override | Gold, Purple, Red |
| Kill Sound | Sound played when the player eliminates a Hider | Custom thud, KO sound |
| Level Badge | Badge icon displayed in HUD and scoreboard | Per level range |
| Title | Text title displayed below the player name | Configurable per unlock |
| Spray | In-game spray graphic | Logo designs, personal images (moderated) |
| Nameplate | Animated border around the player nameplate | Flame border, Lightning |

### 9.3 Cosmetic Acquisition

| Source | Description |
|---|---|
| Level Rewards | Automatically unlocked at specific levels |
| Achievement Rewards | Unlocked on achievement completion |
| Monthly Record Reward | Awarded to JumpStats monthly leaderboard winners |
| VIP Account Tier | Exclusive set available to VIP+ accounts |
| Admin Grant | Admin-issued for events, contests, or moderation purposes |
| Future: Event Drops | Time-limited cosmetics from server events (see Roadmap) |

Cosmetics are never purchasable for real money in the base system. A microtransaction economy is a future roadmap item requiring a separate design document and legal review.

### 9.4 Cosmetic Equipping

- Players manage their equipped cosmetics via `!cosmetics` menu (see Section 13).
- Multiple items per category may be owned; only one can be equipped per category at a time.
- Equipped cosmetic preferences are stored to the Account System and persist across sessions.

### 9.5 Cosmetic Visibility

- Trails are visible to all players on the server.
- Skins are visible to all players on the server.
- Chat tags and name colors are visible in all chat contexts.
- Kill sounds are heard by all players within a configurable radius.
- An option to disable cosmetic rendering client-side is NOT provided at the server level; this is a client-side preference outside the scope of this specification.

### 9.6 Cosmetic Moderation

- Player-submitted sprays must be approved by an admin before activation.
- Unapproved sprays default to a GM logo placeholder.
- Admin may revoke any cosmetic without notice for policy violations.
- All cosmetic grant events are logged to the audit system.

---

## 10. ACCOUNT SYSTEM

### 10.1 Overview

The Account System manages player identity, session state, preferences, and account tier privileges. It is the root of all per-player data in the ecosystem.

### 10.2 Account Registration

Account creation is automatic on first server connection. No external registration is required for base-tier accounts. A SteamID64 is the canonical identity key.

Account record fields:

| Field | Type | Description |
|---|---|---|
| `account_id` | UUID | Internal primary key |
| `steam_id_64` | String | Canonical identity |
| `username` | String | Steam display name at last login |
| `created_at` | Timestamp | First connection timestamp |
| `last_seen` | Timestamp | Most recent connection |
| `account_tier` | Enum | See Section 10.3 |
| `region` | Enum | Assigned region (see Section 11) |
| `preferred_language` | Enum | Language preference |
| `total_playtime_seconds` | Integer | Lifetime server playtime |
| `is_banned` | Boolean | Account ban status |
| `ban_reason` | String | Populated if banned |
| `ban_expiry` | Timestamp | Null if permanent |

### 10.3 Account Tiers

| Tier | Label | Acquisition | Benefits |
|---|---|---|---|
| Standard | Player | Default (automatic) | Full feature access at base rates |
| VIP | VIP | Admin grant or community milestones | +20% XP, exclusive cosmetics, reserved slot |
| VIP+ | VIP+ | Admin grant | All VIP benefits + exclusive nameplate + extended cosmetic set |
| Admin | Admin | Server admin assignment | Full moderation access, stat management tools |
| Owner | Owner | Reserved for server owner | Unrestricted system access |

VIP and VIP+ tiers must not provide gameplay-affecting mechanical advantages (no speed boosts, no extra hitpoints, no weapon modifications).

### 10.4 Preferences

Player preferences stored per account:

| Preference Key | Type | Default | Description |
|---|---|---|---|
| `language` | Enum | `id` (Indonesian) | Interface language |
| `show_jumpstats_hud` | Boolean | true | Show JumpStats HUD overlay |
| `show_velocity_hud` | Boolean | false | Show velocity HUD overlay |
| `show_trails_other` | Boolean | true | Render other players' trails |
| `jump_sound` | Boolean | true | Play own jump sound FX |
| `chat_notifications` | Boolean | true | Show achievement/record announcements |
| `private_stats` | Boolean | false | Hide stats from public leaderboard inspection |

### 10.5 Session Tracking

Each server connection creates a session record:

| Field | Type |
|---|---|
| `session_id` | UUID |
| `account_id` | Foreign key |
| `connected_at` | Timestamp |
| `disconnected_at` | Timestamp (nullable) |
| `ip_hash` | SHA256 of IP (for abuse tracking, not stored raw) |
| `server_id` | Server identifier |

Sessions exceeding 8 hours without disconnection are auto-closed and flagged for review.

### 10.6 Account Linking (Future)

Future support for Discord OAuth account linking (see Roadmap, Section 19). Infrastructure for `linked_accounts` table must be reserved in schema.

### 10.7 Data Privacy

- No raw IP addresses are stored. IP hashing is performed before any write.
- Player usernames are stored as-received from Steam and may change. Historical usernames are not retained beyond the most recent login.
- Account deletion requests are supported via admin tool. Deletion anonymizes personal data while retaining aggregate stats (XP, rank history, JumpStat records) attributed to a deleted account label.

---

## 11. REGIONAL SYSTEM

### 11.1 Overview

The Regional System segments players by geography to enable regional leaderboards, regional records in JumpStats, and future inter-regional competition events. It is a passive classification layer; region does not affect matchmaking or team assignment.

### 11.2 Region Definitions

| Region Code | Region Name | Countries Included |
|---|---|---|
| `ID` | Indonesia | Indonesia (all provinces) |
| `MY` | Malaysia | Malaysia, Brunei |
| `SG` | Singapore | Singapore |
| `TH` | Thailand | Thailand |
| `PH` | Philippines | Philippines |
| `VN` | Vietnam | Vietnam |
| `SEA` | Southeast Asia Other | All other SEA countries |
| `APAC` | Asia-Pacific | AU, NZ, JP, KR, TW, HK |
| `EU` | Europe | All European countries |
| `NA` | North America | US, CA, MX |
| `GLOBAL` | Global | Catch-all for unclassified |

### 11.3 Region Assignment

Region is assigned at account creation by GeoIP lookup of the connecting IP. The GeoIP database used must be updated at minimum monthly. Region is stored to the account record and is not re-evaluated on subsequent connections unless an admin triggers a re-check or the player explicitly requests via support.

A player may not self-select their region. Region change requests are reviewed by admins and require documented justification (e.g., relocation).

### 11.4 Regional Leaderboards

Each region maintains independent leaderboards for:

- JumpStats (per jump type).
- Round Survival rate.
- Elimination count.
- Total XP (regional rank).

Regional leaderboards are displayed in-game via `!rlb` command and on the web panel.

### 11.5 Map Metadata Schema

Every map in the rotation carries a metadata file (`.json` format) with the following fields:

```
map_id: string
map_name: string
display_name: string
author: string
version: string
hns_compatible: boolean
valid_round_types: array<enum>
seeker_blind_duration_override: integer | null
spawn_zone_bounds: array<Vector3Pair>
valid_landing_zones: array<BoundingBox>
recommended_player_count: IntRange
jump_surface_declarations: array<SurfaceDeclaration>
```

Map metadata must be reviewed and committed before a map is added to the rotation.

---

## 12. COMMANDS

### 12.1 Command Design Standards

- All commands support both `/` (console) and `!` (chat) prefixes.
- Commands with sub-options open the relevant menu section (see Section 13).
- All command responses are rendered through the Language System.
- Invalid arguments return a localized usage string.
- Admin-only commands return a localized permission-denied message to non-admins without disclosing admin-only status.

### 12.2 Player Commands

#### 12.2.1 General Commands

| Command | Aliases | Description |
|---|---|---|
| `!help` | `/help` | Opens the main help menu |
| `!menu` | `/menu` | Opens the main navigation menu |
| `!settings` | `/cfg`, `/options` | Opens player preferences menu |
| `!lang` | `/language` | Opens language selection menu |
| `!rules` | `/rules` | Displays server rules in a popup panel |
| `!bug` | `/bug` | Opens bug/exploit report form |
| `!discord` | `/discord` | Displays Discord invite link |
| `!website` | `/web` | Displays server website URL |

#### 12.2.2 Profile and Stats Commands

| Command | Aliases | Description |
|---|---|---|
| `!profile` | `/me`, `/p` | Opens own profile panel |
| `!profile <name>` | `/p <name>` | Opens target player's profile panel |
| `!stats` | `/mystats` | Displays personal gameplay statistics |
| `!rank` | `/myrank` | Displays personal rank and XP status |
| `!level` | `/mylevel` | Displays current level and XP to next level |
| `!achievements` | `/ach` | Opens achievement browser panel |

#### 12.2.3 JumpStats Commands

| Command | Aliases | Description |
|---|---|---|
| `!js` | `/jumpstats` | Opens personal JumpStats panel |
| `!lb` | `/leaderboard` | Opens global JumpStats leaderboard menu |
| `!rlb` | `/regionlb` | Opens regional JumpStats leaderboard |
| `!mlb` | `/monthlylb` | Opens monthly JumpStats leaderboard |
| `!pbs` | `/personalbests` | Lists all personal bests by jump type |
| `!wr` | `/worldrecords` | Displays current world records per jump type |
| `!nrec` | `/nationalrecord` | Displays national records for player's region |

#### 12.2.4 Cosmetics Commands

| Command | Aliases | Description |
|---|---|---|
| `!cosmetics` | `/cos`, `/skins` | Opens cosmetics management menu |
| `!trail` | `/trail` | Quick-open trail selection sub-menu |
| `!title` | `/title` | Quick-open title selection sub-menu |
| `!tag` | `/chattag` | Quick-open chat tag sub-menu |

#### 12.2.5 Training Commands

| Command | Aliases | Description |
|---|---|---|
| `!training` | `/train`, `/practice` | Opens training mode selector menu |
| `!challenges` | `/ch` | Opens challenge browser |
| `!velocity` | `/vel` | Toggles velocity HUD overlay |
| `!hud` | `/togglehud` | Toggles JumpStats HUD overlay |

#### 12.2.6 Social Commands

| Command | Aliases | Description |
|---|---|---|
| `!friends` | `/fl` | Opens friends list panel |
| `!addfriend <name>` | `/af <name>` | Sends a friend request to target player |
| `!party` | `/squad` | Opens party management panel |

### 12.3 Admin Commands

Admin commands are prefixed with `!a` or `/a` by convention.

| Command | Description | Min Tier |
|---|---|---|
| `!a kick <player> [reason]` | Kick player from server | Admin |
| `!a ban <player> <duration> [reason]` | Ban player | Admin |
| `!a unban <steamid>` | Remove ban | Admin |
| `!a mute <player> <duration>` | Mute player (chat) | Admin |
| `!a slay <player>` | Eliminate player | Admin |
| `!a slayteam <team>` | Eliminate all players on team | Admin |
| `!a forceseeker <player>` | Force player to Seeker role next round | Admin |
| `!a setregion <player> <region>` | Override player region | Admin |
| `!a givexp <player> <amount>` | Grant XP to player | Admin |
| `!a givecosm <player> <cosmetic_id>` | Grant cosmetic to player | Admin |
| `!a setvip <player> <tier>` | Set account tier | Admin |
| `!a reloadmap` | Reload map metadata | Admin |
| `!a event <type> <duration>` | Start a server event (XP boost, etc.) | Admin |
| `!a invalidatejs <record_id>` | Invalidate a JumpStat record | Admin |
| `!a auditlog <player>` | View player's recent XP audit log | Admin |
| `!a dbstatus` | Display database connection health | Owner |
| `!a reloadplugin <module>` | Hot-reload a specific plugin module | Owner |

---

## 13. MENUS

### 13.1 Menu Design Standards

- All menus use SourceMod's VGUI panel or chat-based menu system, depending on CSS compatibility.
- Menus are navigable with number key inputs.
- All menus support a Back and Close option.
- Menu titles and items are rendered through the Language System.
- Menus do not block gameplay.

### 13.2 Main Navigation Menu (`!menu`)

```
[GM-HNS] MAIN MENU
─────────────────────
1. My Profile
2. JumpStats
3. Leaderboards
4. Achievements
5. Cosmetics
6. Training
7. Settings
8. Help & Support
0. Close
```

### 13.3 Profile Menu

```
[GM-HNS] PROFILE — {player_name}
─────────────────────────────────
Level: {level} ({xp}/{xp_next})
Rank: #{rank} ({region})
Rounds Survived: {survived}
Eliminations: {kills}
JumpStat PB (LJ): {lj_pb} u

1. View Achievements
2. View JumpStats
3. View Cosmetics
4. View Stats History
0. Back
```

### 13.4 JumpStats Menu

```
[GM-HNS] JUMPSTATS
───────────────────
1. My Personal Bests
2. Global Leaderboard
3. Regional Leaderboard
4. Monthly Leaderboard
5. World Records
6. Jump Type Guide
0. Back
```

#### 13.4.1 Leaderboard Sub-Menu (example: Global LJ)

```
[GM-HNS] GLOBAL LEADERBOARD — LONGJUMP
──────────────────────────────────────
#1  PlayerName     285.42 u    2026-04-11
#2  PlayerTwo      283.17 u    2026-03-28
...
[Page 1 / 5]

8. Previous Page
9. Next Page
0. Back
```

### 13.5 Achievements Menu

```
[GM-HNS] ACHIEVEMENTS
──────────────────────
Category:
1. Survival          [12/16]
2. Hunter            [8/14]
3. Movement          [5/12]
4. Progression       [3/6]
5. Social            [2/6]
6. Training          [1/4]
7. Hidden            [? / ?]
0. Back
```

#### 13.5.1 Achievement Category View

```
[GM-HNS] ACHIEVEMENTS — MOVEMENT
──────────────────────────────────
[✓] First Leap       — Bronze — 50 XP
[✓] Strider (220u)   — Bronze — 100 XP
[✓] Jumper (240u)    — Silver — 300 XP
[ ] Flier (260u)     — Gold   — 1000 XP
    Progress: Best LJ 251.3u

0. Back
```

### 13.6 Cosmetics Menu

```
[GM-HNS] COSMETICS
───────────────────
1. Player Skin       [Equipped: Garuda Warrior]
2. Trail             [Equipped: Void Smoke]
3. Chat Tag          [Equipped: [GM]]
4. Name Color        [Equipped: Purple]
5. Kill Sound        [Equipped: Default]
6. Title             [Equipped: Veteran]
7. Spray             [Equipped: GM Logo]
8. Nameplate         [Equipped: Flame Border]
0. Back
```

### 13.7 Settings Menu

```
[GM-HNS] SETTINGS
──────────────────
1. Language             [Indonesian]
2. JumpStats HUD        [ON]
3. Velocity HUD         [OFF]
4. Show Other Trails    [ON]
5. Chat Notifications   [ON]
6. Jump Sound FX        [ON]
7. Private Stats        [OFF]
0. Back
```

### 13.8 Training Menu

```
[GM-HNS] TRAINING
──────────────────
1. Free Bhop Mode
2. Strafing Workshop
3. JumpStats Practice
4. Guided Challenges
5. Seeker Training
6. My Training Records
0. Back
```

---

## 14. LANGUAGE SYSTEM

### 14.1 Overview

The Language System provides a complete localization infrastructure for all player-facing strings in the ecosystem. It is a mandatory component of every module. Hardcoded strings are a build-breaking violation.

### 14.2 Supported Languages

| Code | Language | Status |
|---|---|---|
| `id` | Indonesian (Bahasa Indonesia) | **Canonical** — all strings must exist |
| `en` | English | **Mandatory Fallback** — all strings must exist |
| `ms` | Malay (Bahasa Melayu) | Planned |
| `th` | Thai | Planned |
| `vi` | Vietnamese | Planned |

The Indonesian locale is the canonical truth. If a string exists in `en` but not `id`, this is a specification error, not acceptable behavior.

### 14.3 Translation File Format

Translation files use the SourceMod Translations `.phrases.txt` format. Each module maintains its own phrases file. File naming convention: `gm_<module_name>.phrases.txt`.

Example phrases file structure:

```
"Phrases"
{
    "GM.Round.HiderWin"
    {
        "id"    "Para Hider menang! Selamat bertahan!"
        "en"    "Hiders win! Well survived!"
    }

    "GM.JumpStats.NewPB"
    {
        "id"    "PB baru! {jump_type}: {distance} unit"
        "en"    "New PB! {jump_type}: {distance} units"
    }

    "GM.Achievement.Unlocked"
    {
        "id"    "Pencapaian Terbuka: {achievement_name} (+{xp} XP)"
        "en"    "Achievement Unlocked: {achievement_name} (+{xp} XP)"
    }
}
```

### 14.4 Variable Interpolation

Dynamic values in strings use curly-brace token syntax: `{token_name}`. The language module resolves tokens against a key-value map provided by the calling module at render time.

Tokens are typed:
- `{string}` — raw string substitution.
- `{integer}` — formatted integer.
- `{float:2}` — float with precision specifier.
- `{player}` — player name (automatically color-formatted).

### 14.5 Fallback Behavior

- If a string key exists in `id` but not the player's preferred language: fall back to `id`.
- If a string key exists in `id` but not `en`: fall back to `id` for English users (logged as warning).
- If a string key does not exist in any locale: render the raw key as an error indicator and log a critical error.

### 14.6 Language Selection

Players select their preferred language via `!lang` command or Settings menu. Preference is stored to the Account System. Default language is `id` for all players in the `ID` and `SEA` regions; `en` for all others.

### 14.7 Community Translation Contributions

A community translation portal is specified for the web panel (see Section 19). Approved community translations are merged into official phrase files. All translations require admin approval before deployment.

---

## 15. RANKING SYSTEM

### 15.1 Overview

The Ranking System is a multi-axis player classification framework. It does not reduce player quality to a single number; instead, it maintains independent ranks across gameplay, movement, and overall activity, allowing players to excel in their domain.

### 15.2 Ranking Axes

| Axis | Label | Basis | Leaderboard |
|---|---|---|---|
| Gameplay Rank | Combat Rank | Win rate, survival rate, eliminations per round | Yes |
| Movement Rank | Movement Rank | JumpStats PBs across all jump types (composite score) | Yes |
| Activity Rank | Level Rank | Total XP (Level) | Yes |
| Regional Rank | Regional Rank | Regional total XP | Yes (per region) |

### 15.3 Gameplay Rank

#### 15.3.1 Eligible Rounds
Only rounds played in Standard and Speed Round types contribute to Gameplay Rank. Practice and event rounds are excluded.

#### 15.3.2 Rank Score Formula (Reference)
Rank Score is computed per round and accumulated:

```
Round Score = (survival_bonus × survival_weight)
            + (eliminations × elimination_weight)
            + (last_survivor_bonus)
            + (round_win_bonus)
            - (early_death_penalty)
```

Weights and penalty values are configuration parameters, not hardcoded. Initial values are determined during balance testing.

#### 15.3.3 Rank Tiers

| Tier | Label | Score Range | Badge Color |
|---|---|---|---|
| I | Newcomer | 0 – 999 | Grey |
| II | Amateur | 1,000 – 4,999 | Green |
| III | Skilled | 5,000 – 14,999 | Teal |
| IV | Expert | 15,000 – 39,999 | Blue |
| V | Master | 40,000 – 99,999 | Purple |
| VI | Grandmaster | 100,000 – 249,999 | Red |
| VII | Legend | 250,000 – 499,999 | Gold |
| VIII | Garuda | 500,000+ | Animated |

#### 15.3.4 Rank Decay
Gameplay Rank decays after 30 consecutive days of inactivity at a rate of 2% of current rank score per day, to a floor of the bottom of the current tier. Decay resumes immediately upon return.

### 15.4 Movement Rank

Movement Rank is a composite score derived from JumpStats PBs. Each jump type contributes to the composite with a configurable weight.

| Jump Type | Default Weight |
|---|---|
| Longjump | 25% |
| Bhop Longjump | 20% |
| Multi Bhop | 15% |
| Dropjump | 10% |
| Countjump | 10% |
| Weirdjump | 8% |
| Ladderjump | 7% |
| Higherjump | 5% |

A player with no record in a jump type contributes 0 for that type, reducing their composite. Players are incentivized to record in all categories.

Movement Rank tiers use the same tier labels as Gameplay Rank but independent score thresholds.

### 15.5 Rank Display

Ranks are displayed:
- On the scoreboard (Gameplay Rank badge).
- In the player's profile panel (all axes).
- In chat alongside the player name (configurable, default: Gameplay Rank).
- On the web panel leaderboards.

### 15.6 Rank History

A weekly rank snapshot per player is stored to the database indefinitely. This enables:
- Personal rank history graph on web panel.
- Season-end recap features (Roadmap).
- Anti-decay auditing.

---

## 16. DATABASE REQUIREMENTS

### 16.1 Database Engine

| Requirement | Specification |
|---|---|
| Engine | MySQL 5.7+ or MariaDB 10.4+ |
| Character Set | utf8mb4 |
| Collation | utf8mb4_unicode_ci |
| Connection Mode | Persistent connection pool via SourceMod SQL |
| Minimum Pool Size | 4 concurrent connections |
| Maximum Pool Size | 16 concurrent connections |
| Query Timeout | 5 seconds (non-transactional); 10 seconds (transactional) |

### 16.2 Schema Design Principles

- All tables use UUID primary keys (version 4) to avoid integer overflow and enable future cross-server federation.
- Timestamps use UTC and are stored as `DATETIME` with microsecond precision.
- All foreign key relationships are declared and enforced.
- No application-level data deletion without a corresponding soft-delete column or archival strategy.
- All tables carry `created_at` and `updated_at` columns.

### 16.3 Core Tables

The following tables are mandatory for the base system. This is a logical schema definition; physical column types are implementation-defined within the constraints above.

#### 16.3.1 `accounts`
Stores player identity and tier.

Fields: `account_id`, `steam_id_64`, `username`, `created_at`, `last_seen`, `account_tier`, `region`, `preferred_language`, `total_playtime_seconds`, `is_banned`, `ban_reason`, `ban_expiry`, `updated_at`.

#### 16.3.2 `sessions`
Stores per-connection session records.

Fields: `session_id`, `account_id`, `connected_at`, `disconnected_at`, `ip_hash`, `server_id`.

#### 16.3.3 `preferences`
Stores per-player preference key-value pairs.

Fields: `preference_id`, `account_id`, `pref_key`, `pref_value`, `updated_at`.

#### 16.3.4 `xp_transactions`
Audit log of all XP events.

Fields: `transaction_id`, `account_id`, `xp_amount`, `source_type`, `source_reference`, `round_id`, `created_at`.

#### 16.3.5 `player_levels`
Current level and total XP snapshot per player.

Fields: `account_id`, `current_level`, `total_xp`, `updated_at`.

#### 16.3.6 `rounds`
Round-level metadata for stat attribution.

Fields: `round_id`, `server_id`, `map_name`, `round_type`, `started_at`, `ended_at`, `winner_team`, `hider_count`, `seeker_count`.

#### 16.3.7 `round_participants`
Per-player round participation records.

Fields: `participation_id`, `round_id`, `account_id`, `team`, `survived`, `eliminations`, `time_alive_seconds`, `xp_earned`.

#### 16.3.8 `jumpstat_records`
All validated JumpStat entries.

Fields: `record_id`, `account_id`, `jump_type`, `distance`, `height`, `pre_speed`, `land_speed`, `strafe_count`, `sync_pct`, `airtime_ticks`, `map_name`, `server_id`, `is_pb`, `is_regional_record`, `is_world_record`, `recorded_at`, `is_valid`, `invalidated_by`, `invalidated_at`.

#### 16.3.9 `jumpstat_leaderboard_snapshots`
Monthly snapshot of leaderboard standings.

Fields: `snapshot_id`, `jump_type`, `scope` (global/regional/monthly), `scope_value`, `account_id`, `rank_position`, `distance`, `snapshot_month`, `snapshot_year`.

#### 16.3.10 `achievements`
Achievement catalog definition table.

Fields: `achievement_id`, `category`, `tier`, `name_key`, `description_key`, `xp_reward`, `cosmetic_reward_id`, `is_repeatable`, `is_hidden`, `condition_json`.

#### 16.3.11 `player_achievements`
Player achievement completion records.

Fields: `player_achievement_id`, `account_id`, `achievement_id`, `completed_at`, `times_completed`.

#### 16.3.12 `cosmetic_catalog`
Cosmetic item definitions.

Fields: `cosmetic_id`, `category`, `name_key`, `description_key`, `rarity`, `acquisition_type`, `is_active`.

#### 16.3.13 `player_cosmetics`
Cosmetics owned and equipped by players.

Fields: `player_cosmetic_id`, `account_id`, `cosmetic_id`, `acquired_at`, `acquired_source`, `is_equipped`.

#### 16.3.14 `rank_scores`
Current rank scores per axis per player.

Fields: `rank_score_id`, `account_id`, `axis` (gameplay/movement/activity/regional), `score`, `tier`, `updated_at`.

#### 16.3.15 `rank_history`
Weekly rank snapshots per player.

Fields: `history_id`, `account_id`, `axis`, `score`, `tier`, `rank_position`, `snapshotted_at`.

#### 16.3.16 `admin_log`
Record of all admin actions.

Fields: `log_id`, `admin_account_id`, `target_account_id`, `action_type`, `action_detail`, `performed_at`.

#### 16.3.17 `server_events`
Active server events (XP boosts, special rounds).

Fields: `event_id`, `event_type`, `multiplier`, `started_at`, `ends_at`, `created_by`.

#### 16.3.18 `friends` (Reserved)
Player friend relationships. Reserved; populated in Phase 2.

Fields: `friendship_id`, `requester_id`, `addressee_id`, `status`, `created_at`.

#### 16.3.19 `linked_accounts` (Reserved)
External account links. Reserved; populated in Phase 3.

Fields: `link_id`, `account_id`, `provider`, `provider_user_id`, `linked_at`.

### 16.4 Indexing Requirements

Critical indexes (non-exhaustive):

- `accounts`: unique index on `steam_id_64`.
- `jumpstat_records`: composite index on `(jump_type, distance DESC)` for leaderboard queries.
- `jumpstat_records`: index on `(account_id, jump_type, is_pb)`.
- `xp_transactions`: index on `(account_id, created_at)`.
- `round_participants`: index on `(round_id, account_id)`.
- `rank_scores`: index on `(axis, score DESC)` for ranking queries.

### 16.5 Backup Requirements

- Full database backup: daily, retained for 30 days.
- Incremental backup: every 6 hours, retained for 7 days.
- Point-in-time recovery target: 15 minutes.
- Backup verification (restore test): weekly.
- Backup storage: geographically separate from production database.

---

## 17. PLUGIN ARCHITECTURE

### 17.1 Architecture Philosophy

The GM-HNS plugin architecture follows a **modular monorepo** design. All modules share a single repository and a single shared library (`gm_core`) but compile to independent SourceMod plugin binaries (`.smx` files). Each binary can be loaded, unloaded, and hot-reloaded independently.

This satisfies Principle P-01 (Modularity First) and Principle P-06 (Graceful Degradation).

### 17.2 Module Registry

| Module ID | Plugin Binary | Responsibility | Depends On |
|---|---|---|---|
| `gm_core` | `gm_core.smx` | Shared library, event bus, DB pool, language | None |
| `gm_account` | `gm_account.smx` | Account registration, session, preferences | `gm_core` |
| `gm_team` | `gm_team.smx` | Team assignment, balance, modifiers | `gm_core`, `gm_account` |
| `gm_gameplay` | `gm_gameplay.smx` | Round rules, win conditions, special rounds | `gm_core`, `gm_team` |
| `gm_movement` | `gm_movement.smx` | Movement validation, autobhop, speed cap | `gm_core` |
| `gm_jumpstats` | `gm_jumpstats.smx` | Jump detection, measurement, validation | `gm_core`, `gm_account`, `gm_movement` |
| `gm_xp` | `gm_xp.smx` | XP computation, level management, modifiers | `gm_core`, `gm_account` |
| `gm_rank` | `gm_rank.smx` | Rank score computation, tier management, decay | `gm_core`, `gm_account`, `gm_xp` |
| `gm_achievements` | `gm_achievements.smx` | Achievement evaluation, completion, rewards | `gm_core`, `gm_account`, `gm_xp` |
| `gm_cosmetics` | `gm_cosmetics.smx` | Cosmetic catalog, equip, render triggers | `gm_core`, `gm_account` |
| `gm_training` | `gm_training.smx` | Training mode management, challenge evaluation | `gm_core`, `gm_account`, `gm_jumpstats`, `gm_xp` |
| `gm_region` | `gm_region.smx` | Region assignment, regional leaderboard queries | `gm_core`, `gm_account` |
| `gm_menu` | `gm_menu.smx` | All menu rendering, navigation | `gm_core` + all feature modules |
| `gm_commands` | `gm_commands.smx` | Command registration, routing, permission | `gm_core` + all feature modules |
| `gm_hud` | `gm_hud.smx` | HUD overlays (velocity, JumpStats) | `gm_core`, `gm_jumpstats`, `gm_movement` |
| `gm_antiexploit` | `gm_antiexploit.smx` | Server-side cheat and exploit detection | `gm_core`, `gm_movement` |
| `gm_admin` | `gm_admin.smx` | Admin commands, audit log, event management | `gm_core`, `gm_account` + all feature modules |
| `gm_api` | `gm_api.smx` | REST API bridge (HTTP listener to external API) | `gm_core` + all feature modules |

### 17.3 `gm_core` — Shared Library Specification

`gm_core` is a SourceMod include library (`.inc`) compiled into each dependent module, not a standalone plugin. It provides:

#### 17.3.1 Event Bus
An inter-module event system allowing modules to fire and subscribe to named events without direct plugin dependency.

Events are typed structs. All event definitions are maintained in `gm_events.inc`. No module may fire or subscribe to an event not declared in this file without a specification update.

Core event definitions (non-exhaustive):

| Event Name | Fired By | Payload |
|---|---|---|
| `GM_OnPlayerConnect` | `gm_account` | account_id, steam_id |
| `GM_OnPlayerDisconnect` | `gm_account` | account_id, session_id |
| `GM_OnRoundStart` | `gm_gameplay` | round_id, map_name, round_type |
| `GM_OnRoundEnd` | `gm_gameplay` | round_id, winner_team |
| `GM_OnPlayerEliminated` | `gm_gameplay` | victim_id, attacker_id, round_id |
| `GM_OnPlayerSurvived` | `gm_gameplay` | account_id, round_id, time_alive |
| `GM_OnJumpRecorded` | `gm_jumpstats` | account_id, jump_type, distance, is_pb, is_wr |
| `GM_OnXPAwarded` | `gm_xp` | account_id, amount, source_type |
| `GM_OnLevelUp` | `gm_xp` | account_id, old_level, new_level |
| `GM_OnAchievementUnlocked` | `gm_achievements` | account_id, achievement_id |
| `GM_OnRankChanged` | `gm_rank` | account_id, axis, old_tier, new_tier |

#### 17.3.2 Database Pool
Shared async database connection pool. All modules use this pool exclusively. Modules do not maintain own connections.

Public functions provided:
- `GM_DB_QueryAsync(query, callback, data)` — async query, callback on result.
- `GM_DB_TransactionAsync(queries[], callbacks[], data)` — transactional multi-query.
- `GM_DB_Escape(str)` — SQL string escaping.
- `GM_DB_Status()` — returns pool health enum.

#### 17.3.3 Language Resolver
- `GM_Lang_Format(client, key, ...)` — returns localized formatted string for client.
- `GM_Lang_GetLocale(client)` — returns player's locale code.

#### 17.3.4 Configuration Manager
- All server-side configuration values are defined in `gm_config.cfg`.
- `GM_Config_GetInt(key)`, `GM_Config_GetFloat(key)`, `GM_Config_GetString(key)`.
- Configuration is reloadable at runtime without plugin restart.

### 17.4 Inter-Module Communication Protocol

Modules communicate through exactly three channels:

1. **Event Bus** (fire-and-forget; asynchronous).
2. **Native Functions** (synchronous calls via SourceMod forwards/natives; use sparingly).
3. **Database** (shared data store; modules may read other modules' tables via documented query patterns).

Direct plugin-to-plugin function calls outside of declared natives are prohibited.

### 17.5 Plugin Load Order

Plugins must load in dependency order. The server's `plugins.ini` is maintained by the toolchain:

```
gm_core → gm_account → gm_region → gm_movement →
gm_team → gm_gameplay → gm_jumpstats → gm_xp →
gm_rank → gm_achievements → gm_cosmetics → gm_training →
gm_hud → gm_antiexploit → gm_menu → gm_commands →
gm_admin → gm_api
```

### 17.6 Configuration Files

| File | Purpose |
|---|---|
| `gm_config.cfg` | Master server configuration |
| `gm_jumpstats.cfg` | JumpStats thresholds and weights |
| `gm_xp.cfg` | XP source values and multipliers |
| `gm_rank.cfg` | Rank tier thresholds and decay rates |
| `gm_gameplay.cfg` | Round timers, team balance thresholds |
| `gm_maps/<mapname>.json` | Per-map metadata |

All configuration files are hot-reloadable via `!a reloadconfig`.

### 17.7 Logging

Each module writes to its own log file under `logs/gm_<module>.log`. Log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL`.

`CRITICAL` events trigger an immediate Discord webhook notification to the admin channel.

---

## 18. API REQUIREMENTS

### 18.1 Overview

The GM-HNS API is an external HTTP REST API that exposes read and limited write access to ecosystem data for the web panel, Discord bot, and future integrations. The API is a standalone service (not a SourceMod plugin) that connects to the production database in read mode and to a write queue for safe data mutations.

### 18.2 API Design Standards

- Protocol: HTTPS only. HTTP redirects to HTTPS.
- Format: JSON request and response bodies.
- Authentication: Bearer token (JWT) for all non-public endpoints.
- Versioning: URI versioning (`/v1/`). All endpoints under a version are stable.
- Rate Limiting: 100 requests/minute per authenticated consumer; 20 requests/minute for public endpoints.
- Pagination: Cursor-based for leaderboard endpoints; page-based for lists.

### 18.3 Public Endpoints (No Authentication)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/v1/status` | API and database health status |
| GET | `/v1/leaderboards/jumpstats/{jump_type}` | Global JumpStats leaderboard |
| GET | `/v1/leaderboards/jumpstats/{jump_type}/region/{region}` | Regional JumpStats leaderboard |
| GET | `/v1/leaderboards/jumpstats/{jump_type}/monthly/{year}/{month}` | Monthly JumpStats leaderboard |
| GET | `/v1/leaderboards/rank/{axis}` | Global rank leaderboard by axis |
| GET | `/v1/players/{steam_id}/profile` | Public player profile |
| GET | `/v1/players/{steam_id}/jumpstats` | Player's JumpStat PBs |
| GET | `/v1/players/{steam_id}/achievements` | Player's completed achievements |
| GET | `/v1/players/{steam_id}/rank` | Player's current ranks (all axes) |
| GET | `/v1/records/world` | Current world records per jump type |

### 18.4 Authenticated Endpoints

#### Player Endpoints (Player JWT)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/v1/me/profile` | Own full profile |
| GET | `/v1/me/xp-history` | Own XP transaction history |
| GET | `/v1/me/sessions` | Own session history |
| PATCH | `/v1/me/preferences` | Update own preferences |

#### Admin Endpoints (Admin JWT)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/v1/admin/players` | Player list with search and filter |
| GET | `/v1/admin/players/{account_id}/audit` | XP and action audit log |
| POST | `/v1/admin/players/{account_id}/ban` | Apply ban |
| DELETE | `/v1/admin/players/{account_id}/ban` | Remove ban |
| PATCH | `/v1/admin/players/{account_id}/tier` | Set account tier |
| POST | `/v1/admin/players/{account_id}/xp` | Grant XP |
| POST | `/v1/admin/players/{account_id}/cosmetics` | Grant cosmetic |
| DELETE | `/v1/admin/jumpstats/{record_id}` | Invalidate JumpStat record |
| POST | `/v1/admin/events` | Start a server event |
| GET | `/v1/admin/logs` | Admin action log |

### 18.5 Webhook Outputs

The API fires outbound webhooks to configured endpoints on the following events:

| Event | Destination | Payload |
|---|---|---|
| World Record broken | Discord #records channel | Player, jump type, distance, previous WR |
| Server event started | Discord #announcements | Event type, duration, multiplier |
| CRITICAL plugin error | Discord #admin-alerts | Module, error message, stack (if available) |
| Player banned | Discord #admin-log | Admin, player, reason, duration |

Webhook delivery is at-least-once with exponential backoff retry (max 5 attempts).

### 18.6 API Security Requirements

- All JWT tokens expire after 24 hours. Refresh token support required.
- Admin JWTs require MFA (one-time code via Discord or TOTP).
- All write operations are logged to an API audit log.
- SQL injection prevention: all queries use parameterized statements; no string interpolation.
- CORS policy: restricted to declared origins (web panel domain + Discord bot domain).

---

## 19. FUTURE ROADMAP

The following features are out of scope for the initial production release (v1.0) but are explicitly planned. Architecture decisions in v1.0 must not foreclose these features. Where a v1.0 system must lay infrastructure for a roadmap feature, this is noted.

### Phase 2: Social & Community Layer (Target: 3 months post-launch)

| Feature | Description | Infrastructure Dependency |
|---|---|---|
| Friends System | Add, manage, party with friends in-game | `friends` table reserved in schema |
| Party System | Play together with shared XP bonus | Friends system must be live |
| Discord Account Link | Link Steam to Discord for Discord leaderboard bot | `linked_accounts` table reserved |
| Web Panel v1 | Public web panel with leaderboards, profiles, stats | API v1 must be live |
| Discord Bot | Leaderboard queries, player lookups, live record announcements | API v1 + Discord OAuth link |
| Community Translation Portal | Submit and vote on translations | Web panel must be live |

### Phase 3: Economy & Events (Target: 6 months post-launch)

| Feature | Description | Infrastructure Dependency |
|---|---|---|
| Server Events System | Admin-defined time-limited events with custom rules | `server_events` table must be live |
| Seasonal Ranking | Quarterly season resets with seasonal cosmetic rewards | Rank history must be 1+ seasons old |
| Monthly Recap | End-of-month player performance summary via Discord DM | Discord link required |
| Event Cosmetic Drops | Time-limited cosmetics available during events | Events system required |
| Referral System | Structured referral tracking with rewards | Friends system required |

### Phase 4: Technical Prestige (Target: 9 months post-launch)

| Feature | Description | Infrastructure Dependency |
|---|---|---|
| Jump Replay System | Record and replay jump attempts in-game | Position tick data collected from v1.0 |
| Custom Map Submission | Community map submission with admin review pipeline | Web panel required |
| Clan / Team System | Registered clans with clan leaderboards | Social layer required |
| Inter-Region Events | Head-to-head regional competitions | Regional system + seasonal ranking |
| Advanced Anti-Cheat | ML-assisted anomaly detection on movement data | 6+ months of baseline data required |

### Phase 5: Platform Expansion (Target: 12+ months post-launch)

| Feature | Description |
|---|---|
| Multi-Server Federation | Shared rank and stats across multiple GM server instances |
| Web Panel v2 | Full-featured dashboard with personalized analytics |
| Cosmetic Microtransaction Economy | Optional purchase model (requires separate legal/design review) |
| Mobile Companion App | View stats, achievements, leaderboards on mobile |
| CS2 Migration Study | Feasibility assessment of porting ecosystem to CS2 |

---

## APPENDIX A: GLOSSARY

| Term | Definition |
|---|---|
| BHop | Bunny Hop; the technique of jumping at the moment of landing to preserve and gain horizontal velocity |
| HNS | Hide and Seek; the game mode this ecosystem is built for |
| HU | Hammer Units; the native distance unit of the Source Engine |
| JumpStat | A recorded and validated jump measurement |
| PB | Personal Best; a player's best recorded result in a JumpStat category |
| WR | World Record; the best recorded JumpStat result across all players |
| Seeker | CT-team player whose role is to eliminate Hiders |
| Hider | T-team player whose role is to survive the round |
| SteamID64 | The canonical 64-bit Steam identifier used as account key |
| Sync % | Synchronization percentage; the proportion of airtime during which strafing inputs align with mouse movement |
| Tick | A single server simulation step; at 66 tick, approximately 15.15ms |
| XP | Experience Points; the primary progression currency |

---

## APPENDIX B: VERSION HISTORY

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0.0 | June 2026 | Lead Technical Architect | Initial production specification |

---

*End of GM_HNS_MASTER_SPEC.md*

---

> **This document is the official project contract for the Garuda Movement HNS Ecosystem.**  
> All implementation decisions must reference and conform to this specification.  
> Deviations require a formal specification amendment with version increment.
