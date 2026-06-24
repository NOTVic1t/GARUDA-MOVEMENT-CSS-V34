# GM_DATABASE_SPEC.md
## Garuda Movement — HNS Chase Ecosystem
### Production Database Architecture Specification
### Version 1.0 | June 2026

---

> **Document Status:** OFFICIAL DATABASE SPECIFICATION
> **References:** GM_HNS_MASTER_SPEC_V1_FINAL.md (V1.0 FINAL — LOCKED)
> **Classification:** Internal Architecture
> **Maintainer:** Lead Database Architect, Garuda Movement
> **Engine:** MySQL 8.0+ / MariaDB 10.6+
> **Scope:** Complete database architecture for GM-HNS V1.0 — SEA Region

---

## TABLE OF CONTENTS

1. [Architecture Overview](#1-architecture-overview)
2. [Design Decisions](#2-design-decisions)
3. [Entity Relationship Diagram](#3-entity-relationship-diagram)
4. [Table Catalog](#4-table-catalog)
5. [Table Definitions](#5-table-definitions)
6. [Index Strategy](#6-index-strategy)
7. [Unique Constraints](#7-unique-constraints)
8. [Data Retention Policy](#8-data-retention-policy)
9. [Archiving Strategy](#9-archiving-strategy)
10. [Performance Considerations](#10-performance-considerations)
11. [Scalability Considerations](#11-scalability-considerations)
12. [SourceMod Access Patterns](#12-sourcemod-access-patterns)
13. [Future Compatibility Provisions](#13-future-compatibility-provisions)

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 Purpose

This document defines the complete production database architecture for the Garuda Movement HNS Chase Ecosystem (GM-HNS) V1.0. It translates the locked gameplay specification into a concrete, implementation-ready data model.

This document covers:
- All table structures, column definitions, and data types.
- Primary key strategy, foreign key relationships, and constraint definitions.
- Index design for each table including justification.
- Data lifecycle policy: retention, archival, and soft-delete strategy.
- Performance design for SourceMod's async SQL access patterns.
- Forward-compatibility provisions for Phase 2 and beyond.

This document does **not** contain SQL `CREATE TABLE` statements, migration files, or SourcePawn code.

### 1.2 System Classification

Tables in GM-HNS are classified into four tiers based on access frequency and criticality:

| Tier | Label | Description | Examples |
|---|---|---|---|
| T1 | Hot | Read and written every round or on player connect. Must be in memory-friendly format. | `accounts`, `player_stats`, `player_settings` |
| T2 | Warm | Written at round end or on significant game events. Read frequently for leaderboards. | `jumpstat_records`, `rank_scores`, `xp_ledger` |
| T3 | Cold | Written infrequently. Read for audit, history, or display panels. | `sessions`, `admin_log`, `rank_history` |
| T4 | Archive | Rarely read. Long-term record of events. Eligible for table partitioning. | `round_participants`, `xp_transactions_archive` |

### 1.3 Scope of V1.0 Database

The following systems are served by this database:

- Account identity (Steam, NonSteam, Guest)
- Session tracking
- Player settings and preferences
- Gameplay statistics (rounds, survival, eliminations)
- XP and level progression
- JumpStats records and server leaderboard
- JumpStat tick data (Phase 3 replay reservation)
- Achievements
- Cosmetics catalog and ownership
- Ranking (Gameplay and Movement axes)
- Rank history snapshots
- Admin audit log
- Server registry
- Bug report log

---

## 2. DESIGN DECISIONS

### 2.1 Primary Key Strategy: BIGINT AUTO_INCREMENT

**Decision:** All tables use `BIGINT UNSIGNED AUTO_INCREMENT` as the primary key type.

**Rationale:**

| Consideration | UUID | BIGINT AUTO_INCREMENT |
|---|---|---|
| Insert performance | Poor — random ordering causes B-tree fragmentation | Excellent — sequential inserts, minimal fragmentation |
| Index size | 16 bytes per value | 8 bytes per value |
| SourceMod SQL usage | Requires string handling for UUIDs | Native integer; `SQL_FetchInt()` is direct and fast |
| Join performance | Slower — larger key comparisons | Faster — integer equality is the cheapest comparison |
| Readability in admin tools | Difficult | Simple numeric IDs |
| Auto_increment overflow risk | N/A | BIGINT: 9.2 × 10¹⁸ — effectively inexhaustible |
| Multi-server federation (Phase 2+) | Globally unique without coordination | Requires a server_id offset or distributed ID strategy (planned, Section 13.2) |

**Conclusion:** BIGINT AUTO_INCREMENT is the correct choice for V1.0 SourceMod performance. UUID fragmentation on high-write tables (jumpstat_records, xp_transactions) would cause unacceptable InnoDB performance degradation at scale.

**Foreign key references** throughout the schema use `BIGINT UNSIGNED` to match.

### 2.2 account_id as Canonical Identity

Every table that references a player uses `account_id` as the foreign key. `steam_id_64`, `discord_id`, and `lsgi_identifier` are stored exclusively on the `accounts` table as secondary lookup fields.

This means:
- A SourceMod plugin resolves any external identity (SteamID, LSGI) to `account_id` at player connect time.
- All subsequent queries within the session use `account_id` — no external identity string is passed to the game thread after the initial lookup.
- Account Merge (Phase 2) only requires updating the `account_id` on one record; all child records remain attached via FK.

### 2.3 Data Separation Strategy

Static data (catalog records that rarely change) is separated from dynamic data (per-player, per-round records that change constantly).

| Data Class | Examples | Strategy |
|---|---|---|
| Catalog / Static | `achievement_catalog`, `cosmetic_catalog`, `servers` | Small tables, loaded into SourceMod memory at startup |
| Account Identity | `accounts` | Single lookup per session; cached in plugin |
| Active State | `player_stats`, `player_levels`, `rank_scores` | One row per player; updated in place; heavily cached |
| Ledger / Append-Only | `xp_transactions`, `round_participants` | Insert-only; never updated; partition-eligible |
| Audit | `admin_log`, `sessions` | Insert-only; long-term retention; cold storage eligible |

### 2.4 Soft Delete Policy

No player-facing records are hard-deleted. Instead:

- `accounts` table has an `is_deleted` flag and a `deleted_at` timestamp.
- When an account is deleted, personal identifiers (`steam_id_64`, `discord_id`, `lsgi_identifier`, `username`) are set to NULL or replaced with an anonymized placeholder.
- All child records (stats, JumpStats, achievements) remain in place attributed to the anonymized account. This preserves server record integrity.
- The `is_deleted` flag is checked during the connect-time account lookup. Deleted accounts are refused connection.

### 2.5 Timestamp Convention

- All timestamps are stored as `DATETIME(3)` — UTC, with millisecond precision.
- Millisecond precision is required for JumpStat `recorded_at` to correctly order rapid successive records.
- `created_at` columns use `DEFAULT CURRENT_TIMESTAMP(3)`.
- `updated_at` columns use `DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3)`.

### 2.6 Character Set

- All tables and columns: `CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`.
- Reason: CSS player names may contain Unicode characters including full-width CJK, Thai, and Arabic script characters from the SEA player base.

### 2.7 Storage Engine

- All tables: InnoDB.
- Reason: InnoDB is required for foreign key enforcement, transactional writes, row-level locking, and crash recovery. MyISAM is prohibited.

---

## 3. ENTITY RELATIONSHIP DIAGRAM

The following text-format ERD documents all entity relationships in the V1.0 schema. Cardinality notation: `1` (one), `N` (many), `?` (zero or one / optional).

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    GARUDA MOVEMENT — V1.0 ERD                           ║
╚══════════════════════════════════════════════════════════════════════════╝

[ servers ]
    |
    | 1 : N
    |
[ rounds ] ──────────────────────────────────────────────────────────────┐
    |                                                                     |
    | 1 : N                                                               |
    |                                                                   [ round_participants ]
    |                                                                     |
    └─────────────────────────────────────────────────────────────────── | ──┐
                                                                          |   |
                                                                          N   |
                                                                          |   |
[ accounts ] ─────────────────────────────────────────────────────────── 1   |
    |                                                                         |
    ├── 1 : 1  ── [ player_stats ]        (lifetime aggregate stats)         |
    |                                                                         |
    ├── 1 : 1  ── [ player_levels ]       (XP + level)                       |
    |                                                                         |
    ├── 1 : 1  ── [ player_settings ]     (preferences, equipped cosmetics)  |
    |                                                                         |
    ├── 1 : N  ── [ sessions ]            (per-connection records)           |
    |                                                                         |
    ├── 1 : N  ── [ xp_transactions ]     (XP audit ledger)                  |
    |                                                                         |
    ├── 1 : N  ── [ jumpstat_records ]    (all jump entries)                  |
    |                   |                                                      |
    |                   └── 1 : N ── [ jumpstat_tick_data ]  (per-tick pos)  |
    |                                                                         |
    ├── 1 : N  ── [ player_achievements ] ── N : 1 ── [ achievement_catalog ]|
    |                                                                         |
    ├── 1 : N  ── [ player_cosmetics ]    ── N : 1 ── [ cosmetic_catalog ]   |
    |                                                                         |
    ├── 1 : N  ── [ rank_scores ]         (current score per axis)           |
    |                                                                         |
    ├── 1 : N  ── [ rank_history ]        (weekly snapshots)                 |
    |                                                                         |
    ├── 1 : N  ── [ admin_log ]           (actions performed by this admin)  |
    |                   |                                                      |
    |                   └── N : ?  ── [ accounts ]  (target of admin action) |
    |                                                                         |
    ├── 1 : N  ── [ bug_reports ]         (map exploit reports)              |
    |                                                                         |
    └── 1 : N  ── [ round_participants ] ─────────────────────────────────── ┘
                       |
                       N : 1  ── [ rounds ]

[ achievement_catalog ]
    └── 1 : N  ── [ player_achievements ]

[ cosmetic_catalog ]
    └── 1 : N  ── [ player_cosmetics ]

[ servers ]
    └── 1 : N  ── [ rounds ]
    └── 1 : N  ── [ jumpstat_records ]
    └── 1 : N  ── [ sessions ]

[ account_merge_log ]   (reserved — Phase 2)
    └── links source_account_id : N : 1 : target_account_id ── [ accounts ]

[ linked_accounts ]     (reserved — Phase 2)
    └── N : 1  ── [ accounts ]

[ friends ]             (reserved — Phase 2)
    └── requester_id : N : 1 ── [ accounts ]
    └── addressee_id : N : 1 ── [ accounts ]
```

---

## 4. TABLE CATALOG

Complete list of all V1.0 tables with tier classification and row growth estimate.

| # | Table Name | Tier | Description | Expected Row Growth |
|---|---|---|---|---|
| 01 | `accounts` | T1 | Player identity and ban status | Slow (unique players) |
| 02 | `player_stats` | T1 | Lifetime gameplay aggregate per player | Slow (1 row/player) |
| 03 | `player_levels` | T1 | XP and level state per player | Slow (1 row/player) |
| 04 | `player_settings` | T1 | Preferences and equipped cosmetics | Slow (1 row/player) |
| 05 | `sessions` | T3 | Per-connection session log | Moderate |
| 06 | `servers` | T3 | Registered server registry | Static |
| 07 | `rounds` | T4 | Per-round metadata | Fast |
| 08 | `round_participants` | T4 | Per-player per-round participation | Very fast |
| 09 | `xp_transactions` | T4 | Append-only XP audit ledger | Very fast |
| 10 | `jumpstat_records` | T2 | All validated jump entries | Fast |
| 11 | `jumpstat_tick_data` | T3 | Per-tick position data per jump | Very fast (reserved) |
| 12 | `achievement_catalog` | T1 | Achievement definitions (static) | Static |
| 13 | `player_achievements` | T2 | Achievement completion per player | Moderate |
| 14 | `cosmetic_catalog` | T1 | Cosmetic item definitions (static) | Static |
| 15 | `player_cosmetics` | T2 | Cosmetic ownership per player | Moderate |
| 16 | `rank_scores` | T1 | Current rank score per player per axis | Slow (2 rows/player) |
| 17 | `rank_history` | T3 | Weekly rank snapshots | Moderate |
| 18 | `admin_log` | T3 | Admin action audit trail | Slow |
| 19 | `bug_reports` | T3 | Map exploit / bug reports | Slow |
| 20 | `account_merge_log` | T3 | (Reserved Phase 2) Account merge records | Reserved |
| 21 | `linked_accounts` | T2 | (Reserved Phase 2) External identity links | Reserved |
| 22 | `friends` | T2 | (Reserved Phase 2) Friend relationships | Reserved |

---

## 5. TABLE DEFINITIONS

The following definitions specify all columns, data types, constraints, and default values. No SQL DDL statements are included.

---

### 5.01 — `accounts`

**Tier:** T1 (Hot)
**Purpose:** Core player identity record. One row per player. Created on first server connection.
**SourceMod usage:** Loaded into plugin memory at player connect; cached for session duration.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `account_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** Canonical player identity across all tables. |
| `steam_id_64` | VARCHAR(25) | YES | NULL | Steam 64-bit identity. NULL for NonSteam/Guest accounts. |
| `discord_id` | VARCHAR(30) | YES | NULL | Discord snowflake ID. NULL if unlinked. (Phase 2 populated) |
| `lsgi_identifier` | VARCHAR(64) | YES | NULL | NonSteam LSGI client identifier string. NULL for Steam players. |
| `username` | VARCHAR(64) | NO | '' | Last known display name. Updated on every connect. |
| `account_type` | TINYINT UNSIGNED | NO | 0 | Enum: 0=guest, 1=linked_steam, 2=linked_discord, 3=linked_both |
| `region_code` | VARCHAR(12) | NO | 'SEA_OTHER' | GeoIP-assigned region. See Section 5.06. |
| `preferred_language` | VARCHAR(5) | NO | 'id' | Language preference. 'id' or 'en' in V1.0. |
| `total_playtime_seconds` | INT UNSIGNED | NO | 0 | Lifetime server playtime in seconds. Incremented on disconnect. |
| `is_banned` | TINYINT(1) | NO | 0 | 1 if account is currently banned. |
| `ban_reason` | VARCHAR(255) | YES | NULL | Human-readable ban reason. NULL if not banned. |
| `ban_expiry` | DATETIME(3) | YES | NULL | UTC expiry timestamp. NULL if permanent ban. |
| `admin_level` | TINYINT UNSIGNED | NO | 0 | Enum: 0=player, 1=admin, 2=owner. No gameplay effect. |
| `is_deleted` | TINYINT(1) | NO | 0 | Soft delete flag. Deleted accounts are denied connection. |
| `deleted_at` | DATETIME(3) | YES | NULL | Timestamp of deletion. NULL if not deleted. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | First connection timestamp. |
| `last_seen` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | Updated on every connect. |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on any field change. |

**Notes:**
- `account_type` uses TINYINT (not ENUM) to allow bitfield representation of combined link states and to avoid schema alteration when adding link types.
- `steam_id_64` stores the Steam 64-bit ID as a string to avoid 64-bit integer handling issues in SourceMod's SQL layer.
- `lsgi_identifier` and `steam_id_64` are mutually exclusive in practice but both are nullable for Guest accounts with neither identity confirmed.

---

### 5.02 — `player_stats`

**Tier:** T1 (Hot)
**Purpose:** Lifetime aggregate gameplay statistics. One row per player. Updated in-place at round end.
**Design:** Separate from `accounts` to avoid locking the identity row during stat updates. Stats are updated frequently; account identity is read-only mid-session.
**SourceMod usage:** Loaded at player connect; written at round end via async UPDATE.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `account_id` | BIGINT UNSIGNED | NO | — | **Primary Key + Foreign Key → `accounts.account_id`** |
| `rounds_played` | INT UNSIGNED | NO | 0 | Total rounds participated in. |
| `rounds_survived` | INT UNSIGNED | NO | 0 | Rounds survived as Hider. |
| `rounds_as_hider` | INT UNSIGNED | NO | 0 | Total rounds played as Hider. |
| `rounds_as_chaser` | INT UNSIGNED | NO | 0 | Total rounds played as Chaser. |
| `total_eliminations` | INT UNSIGNED | NO | 0 | Lifetime eliminations as Chaser. |
| `times_last_survivor` | INT UNSIGNED | NO | 0 | Times survived as the final Hider alive. |
| `frostnades_deployed` | INT UNSIGNED | NO | 0 | Total successful FrostNade deployments. |
| `total_time_alive_seconds` | BIGINT UNSIGNED | NO | 0 | Cumulative time alive across all Hider rounds. |
| `rounds_won_hider` | INT UNSIGNED | NO | 0 | Rounds won as Hider (team). |
| `rounds_won_chaser` | INT UNSIGNED | NO | 0 | Rounds won as Chaser (team). |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on any field change. |

**Notes:**
- This table is the source of truth for aggregate stats. Individual round records are in `round_participants` (T4 ledger).
- Survival rate is computed at query time: `rounds_survived / NULLIF(rounds_as_hider, 0)`.
- Elimination rate is computed at query time: `total_eliminations / NULLIF(rounds_as_chaser, 0)`.
- No `created_at` column needed — `accounts.created_at` serves this purpose.

---

### 5.03 — `player_levels`

**Tier:** T1 (Hot)
**Purpose:** Current XP total and computed level for each player. One row per player.
**Design:** Separated from `accounts` because XP is updated at round end (every ~3 minutes per active player). Keeping it separate avoids lock contention on the heavily-read `accounts` row.
**SourceMod usage:** Loaded at connect; written after every XP award event.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `account_id` | BIGINT UNSIGNED | NO | — | **Primary Key + Foreign Key → `accounts.account_id`** |
| `total_xp` | BIGINT UNSIGNED | NO | 0 | Lifetime cumulative XP. Never decremented. |
| `current_level` | SMALLINT UNSIGNED | NO | 1 | Computed level based on `total_xp`. Range: 1–100. |
| `xp_at_current_level` | INT UNSIGNED | NO | 0 | XP accumulated within the current level (for progress bar display). |
| `xp_to_next_level` | INT UNSIGNED | NO | 500 | XP required to reach next level. Updated on level-up. |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on any XP change. |

**Notes:**
- `current_level`, `xp_at_current_level`, and `xp_to_next_level` are derived from `total_xp` but stored explicitly to avoid recomputing on every read. They are updated atomically with `total_xp`.
- Storing these precomputed values is a deliberate denormalization for read performance — the level progress bar is rendered on every player profile view.

---

### 5.04 — `player_settings`

**Tier:** T1 (Hot)
**Purpose:** All per-player preferences and currently equipped cosmetics. One row per player.
**Design:** A wide single-row table (not a key-value store) to allow a single SELECT to load all settings at player connect. Avoids N+1 reads from a key-value preferences table.
**SourceMod usage:** Loaded once at player connect; updated on any preference change.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `account_id` | BIGINT UNSIGNED | NO | — | **Primary Key + Foreign Key → `accounts.account_id`** |
| `lang` | VARCHAR(5) | NO | 'id' | Language preference. 'id' or 'en'. |
| `hud_jumpstats` | TINYINT(1) | NO | 1 | Show JumpStats HUD overlay. |
| `hud_speed` | TINYINT(1) | NO | 0 | Show speed HUD overlay. |
| `hud_sync` | TINYINT(1) | NO | 0 | Show sync percentage HUD. |
| `hud_strafe` | TINYINT(1) | NO | 0 | Show per-strafe gain/loss HUD. |
| `chat_notifications` | TINYINT(1) | NO | 1 | Show achievement and server record announcements. |
| `equipped_knife` | VARCHAR(32) | NO | 'knife_default' | Cosmetic ID of currently equipped knife. |
| `equipped_trail` | VARCHAR(32) | NO | 'trail_off' | Cosmetic ID of currently equipped trail. |
| `equipped_title` | VARCHAR(32) | YES | NULL | Cosmetic ID of currently equipped title. NULL = no title shown. |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on any preference change. |

**Notes:**
- `equipped_knife`, `equipped_trail`, `equipped_title` store cosmetic string IDs from the `cosmetic_catalog`. They are VARCHAR references rather than FK integers to avoid joins at HUD render time (the plugin reads these from its in-memory cache).
- This design trades a small amount of referential integrity for significant read performance gain. Validity of equipped cosmetics is enforced at the application layer when the player equips an item.
- `equipped_tag` is **not stored here**. TOP 10 / TOP 3 tags are computed dynamically from `rank_scores` at round end. FOUNDER / BETA tags are stored in `player_cosmetics` and loaded at connect.

---

### 5.05 — `sessions`

**Tier:** T3 (Cold)
**Purpose:** Records each individual server connection. Used for playtime calculation, abuse detection, and admin tooling.
**Design:** Insert-only. No updates except to write `disconnected_at` on player disconnect.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `session_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `server_id` | SMALLINT UNSIGNED | NO | — | **Foreign Key → `servers.server_id`** |
| `connected_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC connect timestamp. |
| `disconnected_at` | DATETIME(3) | YES | NULL | UTC disconnect timestamp. NULL until player disconnects. |
| `ip_hash` | CHAR(64) | NO | — | SHA-256 hex digest of the raw IP address. Raw IP never stored. |
| `duration_seconds` | INT UNSIGNED | YES | NULL | Computed on disconnect: TIMESTAMPDIFF(SECOND, connected_at, disconnected_at). Stored for fast playtime queries. |

**Notes:**
- Sessions with NULL `disconnected_at` after 8 hours are considered stale and are auto-closed by a scheduled server-side task.
- `ip_hash` enables identifying multiple accounts connecting from the same IP without storing the raw IP.

---

### 5.06 — `servers`

**Tier:** T3 (Static)
**Purpose:** Registry of all GM-HNS server instances. Supports future multi-server federation.
**Design:** Manually maintained by the server owner. Small, static table. Loaded into plugin memory at startup.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `server_id` | SMALLINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** Maximum 65,535 registered servers. |
| `server_name` | VARCHAR(64) | NO | — | Human-readable server name. e.g., "GM-HNS ID Main" |
| `server_ip` | VARCHAR(45) | NO | — | Server IP address (IPv4 or IPv6). Not the player IP. |
| `server_port` | SMALLINT UNSIGNED | NO | 27015 | Game port. |
| `region_code` | VARCHAR(12) | NO | 'SEA_OTHER' | Geographic region of this server. |
| `is_active` | TINYINT(1) | NO | 1 | 0 = decommissioned server; records retained. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | Date this server was registered. |

---

### 5.07 — `rounds`

**Tier:** T4 (Archive)
**Purpose:** Per-round metadata record. Created at round start; completed at round end.
**Design:** Insert-mostly. Only updated once to write end-of-round fields. High volume over time — partition-eligible.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `round_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `server_id` | SMALLINT UNSIGNED | NO | — | **Foreign Key → `servers.server_id`** |
| `map_name` | VARCHAR(64) | NO | — | CSS map name string. e.g., "hns_rush_v2" |
| `round_type` | TINYINT UNSIGNED | NO | 0 | Enum: 0=standard, 1=training. |
| `hider_count` | TINYINT UNSIGNED | NO | 0 | Number of Hiders at round start. |
| `chaser_count` | TINYINT UNSIGNED | NO | 0 | Number of Chasers at round start. |
| `started_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC round start timestamp. |
| `ended_at` | DATETIME(3) | YES | NULL | UTC round end timestamp. NULL until round ends. |
| `winner_team` | TINYINT UNSIGNED | YES | NULL | Enum: 0=draw, 1=hiders, 2=chasers. NULL until round ends. |
| `duration_seconds` | SMALLINT UNSIGNED | YES | NULL | Round duration in seconds. Computed on end. |

---

### 5.08 — `round_participants`

**Tier:** T4 (Archive)
**Purpose:** One row per player per round. The raw participation ledger from which aggregate stats are computed.
**Design:** Insert-only. Never updated. Very high insert volume — partition-eligible by `created_at` (round date).
**SourceMod usage:** Batch inserted at round end for all participants.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `participant_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `round_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `rounds.round_id`** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `team` | TINYINT UNSIGNED | NO | — | Enum: 1=hider, 2=chaser. |
| `survived` | TINYINT(1) | NO | 0 | 1 if Hider survived to round end. Always 0 for Chasers. |
| `eliminations` | TINYINT UNSIGNED | NO | 0 | Eliminations made this round. Always 0 for Hiders. |
| `time_alive_seconds` | SMALLINT UNSIGNED | NO | 0 | Seconds the player was alive during the round. |
| `frostnades_deployed` | TINYINT UNSIGNED | NO | 0 | Successful FrostNade hits this round. |
| `xp_earned` | SMALLINT UNSIGNED | NO | 0 | XP awarded for this round's performance. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | Timestamp of round end (when this row was inserted). |

---

### 5.09 — `xp_transactions`

**Tier:** T4 (Archive)
**Purpose:** Append-only XP audit ledger. Every XP award is recorded here permanently.
**Design:** Insert-only. Never updated or deleted. Partitioned by month in production.
**SourceMod usage:** Inserted asynchronously after every XP award event.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `transaction_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `xp_amount` | SMALLINT UNSIGNED | NO | — | XP awarded in this transaction. Always positive. |
| `source_type` | TINYINT UNSIGNED | NO | — | Enum: 1=gameplay, 2=jumpstat, 3=training_challenge, 4=achievement. |
| `source_reference` | VARCHAR(64) | YES | NULL | Context-dependent reference. e.g., round_id (as string), achievement_id, jumpstat record_id. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of XP award. |

**Notes:**
- `source_reference` is VARCHAR to allow flexible referencing across different source types without per-type join tables.
- No FK on `source_reference` — it is a soft reference for human-readable audit purposes.
- Total XP is never computed from this table at runtime. `player_levels.total_xp` is the authoritative current total. This table is for audit and investigation only.

---

### 5.10 — `jumpstat_records`

**Tier:** T2 (Warm)
**Purpose:** All validated jump measurements. The core of the JumpStats system and server leaderboard.
**Design:** Insert-mostly. `is_pb` and `is_server_record` flags are updated when a new record supersedes an old one. High read volume for leaderboard queries.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `record_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `server_id` | SMALLINT UNSIGNED | NO | — | **Foreign Key → `servers.server_id`** |
| `jump_type` | TINYINT UNSIGNED | NO | — | Enum: 1=LJ, 2=CJ, 3=MCJ, 4=LDJ, 5=DBhop. |
| `distance` | FLOAT | NO | — | Horizontal distance in Hammer Units. Primary measurement. |
| `pre_speed` | FLOAT | NO | — | Velocity at jump input frame, in HU/s. |
| `land_speed` | FLOAT | NO | — | Velocity at landing frame, in HU/s. |
| `strafe_count` | TINYINT UNSIGNED | NO | 0 | Number of directional strafe key presses during airtime. |
| `sync_pct` | FLOAT | NO | 0.0 | Strafe synchronization percentage. Range: 0.0–100.0. |
| `airtime_ticks` | SMALLINT UNSIGNED | NO | 0 | Duration in server ticks from liftoff to landing. |
| `map_name` | VARCHAR(64) | NO | — | Map where jump was recorded. |
| `is_pb` | TINYINT(1) | NO | 0 | 1 if this is the player's current personal best for this jump_type. |
| `is_server_record` | TINYINT(1) | NO | 0 | 1 if this is the current server record for this jump_type. |
| `is_valid` | TINYINT(1) | NO | 1 | 0 if invalidated by admin. |
| `invalidated_by` | BIGINT UNSIGNED | YES | NULL | account_id of the admin who invalidated this record. |
| `invalidated_at` | DATETIME(3) | YES | NULL | Timestamp of invalidation. |
| `recorded_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of jump validation. |

**Notes:**
- When a player breaks their PB: the previous PB row has `is_pb` set to 0; the new row inserts with `is_pb = 1`. This preserves full jump history.
- When a player breaks the server record: the previous server record row has `is_server_record` set to 0; the new row inserts with `is_server_record = 1`.
- The server record query is: `WHERE jump_type = ? AND is_server_record = 1 AND is_valid = 1`.
- Full history is preserved for future replay system (Phase 3).
- `distance` and speed values use FLOAT, not DECIMAL. CSS engine velocity calculations are floating-point; FLOAT accurately represents engine output and avoids false precision with DECIMAL.

---

### 5.11 — `jumpstat_tick_data`

**Tier:** T3 (Cold — Reserved for Phase 3)
**Purpose:** Records player position and velocity for every tick during a validated jump. Required infrastructure for the Phase 3 Jump Replay System. Data collection begins at V1.0 launch to ensure replay data is available when the feature ships.
**Design:** Insert-only. Very high volume (a 1-second jump at 66 tick = ~66 rows). Write-once, read-rarely. Partition by `record_id` range or by month.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `tick_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `record_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `jumpstat_records.record_id`** |
| `tick_sequence` | SMALLINT UNSIGNED | NO | — | Tick number within this jump. Starts at 1. |
| `pos_x` | FLOAT | NO | — | Player origin X at this tick. |
| `pos_y` | FLOAT | NO | — | Player origin Y at this tick. |
| `pos_z` | FLOAT | NO | — | Player origin Z at this tick. |
| `vel_x` | FLOAT | NO | — | Player velocity X at this tick. |
| `vel_y` | FLOAT | NO | — | Player velocity Y at this tick. |
| `vel_z` | FLOAT | NO | — | Player velocity Z at this tick. |

**Notes:**
- No `created_at` — the timestamp is inherited from the parent `jumpstat_records.recorded_at`.
- This table will grow very large (estimate: 50–100M rows per year of active play). Partitioning by `record_id` range is mandatory from launch.
- If storage constraints are critical at V1.0 launch, collection may be restricted to is_pb and is_server_record jumps only. This restriction is a deployment decision, not an architecture change.

---

### 5.12 — `achievement_catalog`

**Tier:** T1 (Static)
**Purpose:** Defines all available achievements. Managed by developers, not modified by gameplay.
**Design:** Static catalog table. Loaded entirely into SourceMod memory at plugin startup. Not expected to exceed a few hundred rows.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `achievement_id` | VARCHAR(48) | NO | — | **Primary Key.** Slug identifier. e.g., 'surv_1000', 'mv_lj_260'. |
| `category` | TINYINT UNSIGNED | NO | — | Enum: 1=survival, 2=chaser, 3=movement, 4=progression, 5=training, 6=hidden. |
| `tier` | TINYINT UNSIGNED | NO | — | Enum: 1=bronze, 2=silver, 3=gold, 4=platinum. |
| `name_key` | VARCHAR(64) | NO | — | Language system key for the achievement name. |
| `description_key` | VARCHAR(64) | NO | — | Language system key for the description. |
| `xp_reward` | SMALLINT UNSIGNED | NO | 0 | XP awarded on first completion. |
| `cosmetic_reward_id` | VARCHAR(32) | YES | NULL | Cosmetic catalog ID awarded on completion. NULL if no cosmetic reward. |
| `is_repeatable` | TINYINT(1) | NO | 0 | 1 if the achievement can be completed multiple times. |
| `is_hidden` | TINYINT(1) | NO | 0 | 1 if achievement is hidden until earned. |
| `condition_json` | JSON | NO | — | Machine-readable condition definition. Parsed by `gm_achievements` module. |
| `sort_order` | SMALLINT UNSIGNED | NO | 0 | Display order within category. |
| `is_active` | TINYINT(1) | NO | 1 | 0 to soft-disable without deleting. |

**Notes:**
- `achievement_id` is VARCHAR PK because it is a developer-defined slug used in SourcePawn code by name. This avoids a lookup join when the plugin checks a specific achievement by ID.
- `condition_json` is a structured JSON field defining the tracking logic. The achievement module interprets it; this database spec does not define the JSON schema (it is defined in the plugin spec).

---

### 5.13 — `player_achievements`

**Tier:** T2 (Warm)
**Purpose:** Records which achievements each player has earned and when.
**Design:** Insert-mostly. Updated only for repeatable achievements (incrementing `times_completed`).

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `player_achievement_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `achievement_id` | VARCHAR(48) | NO | — | **Foreign Key → `achievement_catalog.achievement_id`** |
| `completed_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | First (or most recent) completion timestamp. |
| `times_completed` | SMALLINT UNSIGNED | NO | 1 | Total completions. 1 for non-repeatable. |

**Constraint:** UNIQUE on `(account_id, achievement_id)` — one row per player per achievement, regardless of repeatability. `times_completed` tracks repeat count.

---

### 5.14 — `cosmetic_catalog`

**Tier:** T1 (Static)
**Purpose:** Defines all available cosmetic items across all categories.
**Design:** Static catalog table. Loaded into SourceMod memory at plugin startup. Small table — never exceeds a few hundred rows in V1.0.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `cosmetic_id` | VARCHAR(32) | NO | — | **Primary Key.** Slug identifier. e.g., 'knife_karambit', 'trail_purple'. |
| `category` | TINYINT UNSIGNED | NO | — | Enum: 1=knife, 2=trail, 3=title, 4=tag. |
| `name_key` | VARCHAR(64) | NO | — | Language system key for display name. |
| `acquisition_type` | TINYINT UNSIGNED | NO | — | Enum: 1=level_reward, 2=achievement_reward, 3=admin_grant, 4=auto_rank. |
| `unlock_condition` | VARCHAR(128) | YES | NULL | Human-readable unlock condition description. |
| `unlock_level` | SMALLINT UNSIGNED | YES | NULL | Level required for level_reward type items. NULL for other types. |
| `sort_order` | SMALLINT UNSIGNED | NO | 0 | Display order within category. |
| `is_active` | TINYINT(1) | NO | 1 | 0 to soft-disable without deleting. |

**Notes:**
- `cosmetic_id` is VARCHAR PK for the same reason as `achievement_id` — direct plugin reference by slug name.
- Auto-rank tags (TOP 10, TOP 3) have `acquisition_type = 4`. These are NOT stored in `player_cosmetics`. They are computed and applied dynamically from `rank_scores`.

---

### 5.15 — `player_cosmetics`

**Tier:** T2 (Warm)
**Purpose:** Records which cosmetics each player owns. Source of truth for unlocked items.
**Design:** Insert-mostly. One row per player per owned cosmetic. Auto-rank tags (TOP 10, TOP 3) are not stored here; they are dynamic.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `player_cosmetic_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `cosmetic_id` | VARCHAR(32) | NO | — | **Foreign Key → `cosmetic_catalog.cosmetic_id`** |
| `acquired_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | Timestamp of acquisition. |
| `acquired_source` | TINYINT UNSIGNED | NO | — | Enum: 1=level, 2=achievement, 3=admin_grant. |

**Constraint:** UNIQUE on `(account_id, cosmetic_id)` — a player cannot own the same item twice.

**Notes:**
- Currently equipped item is stored in `player_settings`, not here. This table tracks what is *owned*. `player_settings` tracks what is *equipped*.
- The equip action in the plugin: (1) verify `player_cosmetics` row exists for this player + cosmetic, (2) update `player_settings.equipped_<category>`.

---

### 5.16 — `rank_scores`

**Tier:** T1 (Hot)
**Purpose:** Current rank score and tier for each player on each ranking axis. One row per player per axis (2 rows per player in V1.0: gameplay + movement).
**Design:** Updated in-place at round end (gameplay axis) or on JumpStat PB (movement axis). Small, hot table — fits in InnoDB buffer pool easily.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `rank_score_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `axis` | TINYINT UNSIGNED | NO | — | Enum: 1=gameplay, 2=movement. |
| `score` | INT UNSIGNED | NO | 0 | Current rank score on this axis. |
| `tier` | TINYINT UNSIGNED | NO | 1 | Computed rank tier (1=Newcomer ... 8=Garuda). |
| `server_position` | INT UNSIGNED | YES | NULL | Current position on this axis leaderboard. Recomputed after each round. |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on any change. |

**Constraint:** UNIQUE on `(account_id, axis)` — exactly one score per player per axis.

**Notes:**
- `server_position` is stored (denormalized) to avoid an expensive COUNT(*) query every time the leaderboard is displayed. It is recomputed server-side at round end for all affected players on the axis.
- TOP 10 / TOP 3 tag assignment: after `server_position` is updated, the plugin evaluates whether the player qualifies for automatic tag display.

---

### 5.17 — `rank_history`

**Tier:** T3 (Cold)
**Purpose:** Weekly snapshot of each player's rank score, tier, and position. Required for Phase 2 season mechanics and personal rank history visualization.
**Design:** Insert-only. Taken by a scheduled task every Sunday at 00:00 UTC. Never updated. Grows at 2 rows/player/week.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `history_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `axis` | TINYINT UNSIGNED | NO | — | Enum: 1=gameplay, 2=movement. |
| `score` | INT UNSIGNED | NO | 0 | Rank score at snapshot time. |
| `tier` | TINYINT UNSIGNED | NO | — | Tier at snapshot time. |
| `server_position` | INT UNSIGNED | YES | NULL | Server position at snapshot time. |
| `snapshotted_at` | DATETIME(3) | NO | — | Exact timestamp of the snapshot. |

---

### 5.18 — `admin_log`

**Tier:** T3 (Cold)
**Purpose:** Permanent audit trail of all admin actions taken via `!a` commands.
**Design:** Insert-only. Never modified or deleted. Retained permanently.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `log_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `admin_account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** — the admin performing the action. |
| `target_account_id` | BIGINT UNSIGNED | YES | NULL | **FK → `accounts.account_id`** — the player affected. NULL for non-player actions. |
| `action_type` | VARCHAR(32) | NO | — | Action code string. e.g., 'ban', 'kick', 'givexp', 'invalidate_jumpstat'. |
| `action_detail` | VARCHAR(512) | YES | NULL | Human-readable detail of the action. e.g., reason, amount, duration. |
| `performed_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of the action. |

---

### 5.19 — `bug_reports`

**Tier:** T3 (Cold)
**Purpose:** Stores map exploit and bug reports submitted via `!bug`.
**Design:** Insert-only. Reviewed and resolved by admins via admin tooling.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `report_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** — reporter. |
| `server_id` | SMALLINT UNSIGNED | NO | — | **Foreign Key → `servers.server_id`** |
| `round_id` | BIGINT UNSIGNED | YES | NULL | **FK → `rounds.round_id`** — round during which the report was filed. |
| `map_name` | VARCHAR(64) | NO | — | Map name at time of report. |
| `pos_x` | FLOAT | NO | — | Player X position at time of report. |
| `pos_y` | FLOAT | NO | — | Player Y position at time of report. |
| `pos_z` | FLOAT | NO | — | Player Z position at time of report. |
| `description` | VARCHAR(256) | YES | NULL | Optional player-submitted description. |
| `is_resolved` | TINYINT(1) | NO | 0 | 1 when an admin marks it resolved. |
| `resolved_by` | BIGINT UNSIGNED | YES | NULL | **FK → `accounts.account_id`** — resolving admin. |
| `resolved_at` | DATETIME(3) | YES | NULL | UTC resolution timestamp. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC report submission timestamp. |

---

### 5.20 — `account_merge_log` (Reserved — Phase 2)

**Tier:** T3 (Reserved)
**Purpose:** Records all account merge events when a Guest or duplicate account is merged into a primary account. Required for audit trail and merge reversal.
**Design:** Table is created in V1.0 schema (empty) to avoid Phase 2 migration. Insert-only.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `merge_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `source_account_id` | BIGINT UNSIGNED | NO | — | Account that was merged (consumed). |
| `target_account_id` | BIGINT UNSIGNED | NO | — | Account that received the merge (retained). |
| `merged_by` | BIGINT UNSIGNED | YES | NULL | Admin account_id who authorized the merge. NULL if automatic. |
| `merged_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of merge. |
| `source_snapshot_json` | JSON | NO | — | Snapshot of source account's key stats before merge (for audit). |

---

### 5.21 — `linked_accounts` (Reserved — Phase 2)

**Tier:** T2 (Reserved)
**Purpose:** Links external identity providers to an `account_id`. Supports Discord OAuth linking in Phase 2.
**Design:** Table is created in V1.0 schema (empty). Populated by Phase 2 Discord link feature.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `link_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `account_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** |
| `provider` | VARCHAR(20) | NO | — | Identity provider. e.g., 'discord', 'steam_oauth'. |
| `provider_user_id` | VARCHAR(64) | NO | — | User ID as returned by the provider. |
| `linked_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of linking. |

**Constraint:** UNIQUE on `(provider, provider_user_id)` — one account per external identity.

---

### 5.22 — `friends` (Reserved — Phase 2)

**Tier:** T2 (Reserved)
**Purpose:** Stores friend relationships between players. Required for Phase 2 party system.
**Design:** Table is created in V1.0 schema (empty).

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `friendship_id` | BIGINT UNSIGNED | NO | AUTO_INCREMENT | **Primary Key.** |
| `requester_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** — sender of friend request. |
| `addressee_id` | BIGINT UNSIGNED | NO | — | **Foreign Key → `accounts.account_id`** — recipient. |
| `status` | TINYINT UNSIGNED | NO | 0 | Enum: 0=pending, 1=accepted, 2=declined, 3=blocked. |
| `created_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) | UTC timestamp of request. |
| `updated_at` | DATETIME(3) | NO | CURRENT_TIMESTAMP(3) ON UPDATE | Auto-updated on status change. |

**Constraint:** UNIQUE on `(requester_id, addressee_id)`.

---

## 6. INDEX STRATEGY

### 6.1 Index Design Principles

- Every foreign key column has a corresponding index (InnoDB does not auto-create these).
- Covering indexes are used for the highest-frequency queries to eliminate table row lookups.
- Indexes on high-cardinality columns are preferred over low-cardinality columns.
- Composite indexes are ordered: most-selective column first, unless the query pattern dictates otherwise.
- No index is created without a documented query it serves.

---

### 6.2 `accounts` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `account_id` | Identity lookup |
| `idx_steam_id` | UNIQUE | `steam_id_64` | Connect-time Steam identity resolution |
| `idx_lsgi` | UNIQUE | `lsgi_identifier` | Connect-time NonSteam identity resolution |
| `idx_discord_id` | UNIQUE | `discord_id` | Phase 2 Discord link lookup (reserved) |
| `idx_region` | Standard | `region_code` | Regional admin queries |
| `idx_is_banned` | Standard | `is_banned` | Ban check at connect; WHERE is_banned = 1 |
| `idx_last_seen` | Standard | `last_seen` | Activity reports, AFK detection |

**Note:** `idx_steam_id`, `idx_lsgi`, and `idx_discord_id` use UNIQUE partial indexes filtering out NULLs (only one value per NULL column). Check MySQL/MariaDB version compatibility for partial UNIQUE index behavior.

---

### 6.3 `player_stats` / `player_levels` / `player_settings` Indexes

| Table | Index Name | Type | Columns | Purpose |
|---|---|---|---|---|
| All three | PRIMARY | PK | `account_id` | Direct lookup by canonical identity |

These tables are keyed exclusively by `account_id`. No secondary indexes are needed — all access is by primary key.

---

### 6.4 `sessions` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `session_id` | — |
| `idx_account_id` | Standard | `account_id` | Per-player session history queries |
| `idx_ip_hash` | Standard | `ip_hash` | IP abuse pattern detection |
| `idx_connected_at` | Standard | `connected_at` | Time-range queries for activity reports |
| `idx_open_sessions` | Standard | `disconnected_at` | WHERE disconnected_at IS NULL — finding open sessions |

---

### 6.5 `rounds` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `round_id` | — |
| `idx_server_id` | Standard | `server_id` | Per-server round history |
| `idx_map_name` | Standard | `map_name` | Per-map statistics queries |
| `idx_started_at` | Standard | `started_at` | Time-range queries |

---

### 6.6 `round_participants` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `participant_id` | — |
| `idx_round_id` | Standard | `round_id` | All participants in a round |
| `idx_account_id` | Standard | `account_id` | Player's round history |
| `idx_account_round` | Composite | `(account_id, round_id)` | Single player in single round lookup |
| `idx_created_at` | Standard | `created_at` | Partition pruning |

---

### 6.7 `xp_transactions` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `transaction_id` | — |
| `idx_account_id` | Standard | `account_id` | Player XP history (admin audit) |
| `idx_created_at` | Standard | `created_at` | Time-range audit queries; partition pruning |
| `idx_source_type` | Standard | `source_type` | Filter by XP source in audit tool |

---

### 6.8 `jumpstat_records` Indexes

This table has the most complex index requirements. Leaderboard queries are the primary read pattern.

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `record_id` | — |
| `idx_account_id` | Standard | `account_id` | Player's full jump history |
| `idx_server_record_lb` | Composite | `(jump_type, distance DESC, is_valid)` | **Leaderboard query.** Top distances per jump type. WHERE is_valid=1 AND is_server_record=1 |
| `idx_pb_lookup` | Composite | `(account_id, jump_type, is_pb)` | **PB lookup.** Finding a player's current PB for a jump type. |
| `idx_is_pb` | Composite | `(jump_type, distance DESC)` WHERE `is_pb = 1` | All players' PBs ranked — used for Movement Rank computation |
| `idx_map_name` | Standard | `map_name` | Per-map JumpStat queries |
| `idx_recorded_at` | Standard | `recorded_at` | Time-range queries |

**Critical note on `idx_server_record_lb`:** The leaderboard query pattern is: `SELECT ... WHERE jump_type = ? AND is_server_record = 1 AND is_valid = 1 ORDER BY distance DESC LIMIT 100`. The composite index on `(jump_type, is_server_record, distance DESC)` must be evaluated with the server's query optimizer. For MariaDB, a covering index including `account_id` and `distance` may be more efficient depending on plan analysis.

---

### 6.9 `jumpstat_tick_data` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `tick_id` | — |
| `idx_record_id` | Standard | `record_id` | Fetch all ticks for a jump (replay lookup) |
| `idx_record_tick` | Composite | `(record_id, tick_sequence)` | Ordered tick fetch for replay playback |

---

### 6.10 `player_achievements` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `player_achievement_id` | — |
| `idx_account_id` | Standard | `account_id` | All achievements for a player |
| `idx_achievement_id` | Standard | `achievement_id` | All players who earned a specific achievement |

---

### 6.11 `player_cosmetics` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `player_cosmetic_id` | — |
| `idx_account_id` | Standard | `account_id` | All cosmetics owned by a player |
| `idx_account_cosmetic` | UNIQUE | `(account_id, cosmetic_id)` | Ownership check; deduplication |

---

### 6.12 `rank_scores` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `rank_score_id` | — |
| `idx_account_axis` | UNIQUE | `(account_id, axis)` | Player's score on a specific axis |
| `idx_leaderboard` | Composite | `(axis, score DESC)` | **Leaderboard sort.** Rank all players on an axis by score. |
| `idx_tier_lookup` | Composite | `(axis, tier)` | Count of players per tier |

---

### 6.13 `rank_history` Indexes

| Index Name | Type | Columns | Purpose |
|---|---|---|---|
| PRIMARY | PK | `history_id` | — |
| `idx_account_axis` | Composite | `(account_id, axis, snapshotted_at)` | Player's rank history timeline |
| `idx_snapshotted_at` | Standard | `snapshotted_at` | Snapshot range queries |

---

## 7. UNIQUE CONSTRAINTS

Summary of all UNIQUE constraints across the schema, separate from primary keys.

| Table | Constraint Name | Columns | Description |
|---|---|---|---|
| `accounts` | `uq_steam_id` | `steam_id_64` | One account per Steam identity (NULL excluded) |
| `accounts` | `uq_lsgi` | `lsgi_identifier` | One account per LSGI identifier (NULL excluded) |
| `accounts` | `uq_discord_id` | `discord_id` | One account per Discord ID (NULL excluded) |
| `player_stats` | (PK is unique) | `account_id` | One stats row per player |
| `player_levels` | (PK is unique) | `account_id` | One level row per player |
| `player_settings` | (PK is unique) | `account_id` | One settings row per player |
| `player_achievements` | `uq_account_achievement` | `(account_id, achievement_id)` | One completion record per player per achievement |
| `player_cosmetics` | `uq_account_cosmetic` | `(account_id, cosmetic_id)` | No duplicate ownership records |
| `rank_scores` | `uq_account_axis` | `(account_id, axis)` | One score row per player per ranking axis |
| `linked_accounts` | `uq_provider_user` | `(provider, provider_user_id)` | One GM account per external identity |
| `friends` | `uq_friendship` | `(requester_id, addressee_id)` | No duplicate friend requests |

---

## 8. DATA RETENTION POLICY

### 8.1 Policy Overview

| Table | Retention Policy | Reason |
|---|---|---|
| `accounts` | Permanent (soft delete) | Identity integrity; JumpStats attribution |
| `player_stats` | Permanent | Lifetime stat record |
| `player_levels` | Permanent | Progression record |
| `player_settings` | Permanent | Player preference persistence |
| `sessions` | 90 days active, then archive | Playtime calculation; abuse detection window |
| `rounds` | 180 days active, then archive | Recent round browsing; stat auditing |
| `round_participants` | 180 days active, then archive | Recent round stats; aggregate integrity check |
| `xp_transactions` | 180 days active, then archive | XP audit; dispute resolution window |
| `jumpstat_records` | Permanent | Server records are permanent; full history preserved |
| `jumpstat_tick_data` | 365 days (PB/SR records only) | Phase 3 replay; non-PB ticks eligible for purge |
| `achievement_catalog` | Permanent | Static definitions |
| `player_achievements` | Permanent | Earned achievements do not expire |
| `cosmetic_catalog` | Permanent | Static definitions |
| `player_cosmetics` | Permanent | Earned cosmetics do not expire |
| `rank_scores` | Permanent (updated in place) | Current rank is always live |
| `rank_history` | Permanent | Required for Phase 2 season mechanics |
| `admin_log` | Permanent | Audit and accountability |
| `bug_reports` | 365 days after resolution | Map maintenance window |

### 8.2 Deleted Account Policy

When an account is deleted (soft delete):

1. `accounts.is_deleted` set to 1; `accounts.deleted_at` populated.
2. `accounts.steam_id_64`, `discord_id`, `lsgi_identifier` set to NULL.
3. `accounts.username` replaced with `[Deleted]`.
4. All child records (`player_stats`, `jumpstat_records`, etc.) are retained without modification.
5. JumpStat server records attributed to a deleted account remain valid and are displayed as "[Deleted] — {distance} u".
6. Deleted accounts are refused connection. Their UNIQUE identity fields (now NULL) free the constraint for a new account to use the same Steam ID.

---

## 9. ARCHIVING STRATEGY

### 9.1 Archive Table Naming Convention

Archive tables are named: `<original_table>_archive`.
Example: `round_participants` → `round_participants_archive`

Archive tables have identical structure to the source table with two additional columns:
- `archived_at DATETIME(3)` — when this row was moved to archive.
- `archive_batch_id BIGINT UNSIGNED` — the ID of the archival job that moved this row.

### 9.2 Archival Process

Archival is performed by a scheduled MySQL Event or external cron job (not a SourceMod plugin):

1. Rows older than the retention threshold are SELECTed into the archive table in batches (maximum 10,000 rows per batch).
2. After successful INSERT into the archive table, the source rows are DELETEd.
3. The process runs during low-traffic hours (e.g., 03:00–05:00 UTC for SEA primetime avoidance).
4. Failed archival batches are logged and retried; source rows are not deleted if archive INSERT fails.

### 9.3 Partitioning Strategy

The following tables are eligible for range partitioning by date at scale. Partitioning should be evaluated when the table exceeds 5 million rows.

| Table | Partition Key | Partition By |
|---|---|---|
| `round_participants` | `created_at` | RANGE by month |
| `xp_transactions` | `created_at` | RANGE by month |
| `jumpstat_tick_data` | `tick_id` | RANGE by BIGINT range block |
| `sessions` | `connected_at` | RANGE by month |

Partitioning enables:
- Efficient partition pruning for time-range queries.
- Bulk DROP PARTITION for archival (faster than DELETE).

### 9.4 Table Size Estimates (12-Month Projection)

Assumptions: 500 unique players; 100 rounds/day; average 15 players/round; 20 JumpStats/player/day.

| Table | Estimated 12-Month Rows |
|---|---|
| `accounts` | ~500 |
| `player_stats` / `player_levels` / `player_settings` | ~500 each |
| `sessions` | ~50,000 |
| `rounds` | ~36,500 |
| `round_participants` | ~547,500 |
| `xp_transactions` | ~600,000 |
| `jumpstat_records` | ~3,650,000 |
| `jumpstat_tick_data` | ~200,000,000+ |
| `player_achievements` | ~5,000 |
| `player_cosmetics` | ~2,000 |
| `rank_scores` | ~1,000 |
| `rank_history` | ~52,000 |
| `admin_log` | <10,000 |

**Critical observation:** `jumpstat_tick_data` is the only table that will reach hundreds of millions of rows within the first year. Partitioning this table from launch is mandatory, not optional. If storage is constrained, restrict tick recording to PB and server record jumps only (see Section 5.11 notes).

---

## 10. PERFORMANCE CONSIDERATIONS

### 10.1 InnoDB Buffer Pool

The `innodb_buffer_pool_size` should be configured to hold the most frequently accessed tables entirely in memory.

At V1.0 launch, the hot data set is very small:

| Data Class | Estimated Memory |
|---|---|
| `accounts` (500 players) | < 1 MB |
| `player_stats`, `player_levels`, `player_settings` (500 players each) | < 1 MB each |
| `rank_scores` (1,000 rows) | < 1 MB |
| JumpStats leaderboard (top 100 per 5 types) | < 1 MB |
| Achievement and cosmetic catalogs | < 1 MB |
| **Total hot data set** | **< 10 MB** |

A buffer pool of 256 MB is sufficient for V1.0 and will accommodate growth to thousands of players before requiring increase.

### 10.2 SourceMod Async Query Pattern

SourceMod SQL operates asynchronously via callbacks. All database writes from game events follow this pattern:

```
[Game Event Fires]
    → Plugin builds query string
    → GM_DB_QueryAsync() called
    → Control returns to game thread immediately
    → Database executes query on pool thread
    → Callback fires on next frame (or scheduled timer)
    → Result processed
```

**Implications for schema design:**

- Queries must be simple. A single UPDATE to `player_stats` or `player_levels` is preferred over a complex transaction with multiple tables.
- The SourceMod SQL pool handles at most 16 concurrent connections. Under high load (32 players, round end cascade), up to 5–8 queries may fire simultaneously. The pool must queue and serialize these without deadlock.
- Batch inserts at round end (all `round_participants` rows) should be assembled as a single multi-row INSERT, not N individual INSERTs.

### 10.3 Connect-Time Query Plan

The player connect sequence is the most latency-sensitive database interaction. A player noticeably waiting to "load" is a bad experience. The target total connect-time DB time is under 50ms.

The connect-time query plan for a returning Steam player:

```
Query 1: SELECT account_id, is_banned, ban_expiry, ... FROM accounts
         WHERE steam_id_64 = ?
         → Returns account_id for all subsequent queries

Query 2: SELECT * FROM player_stats WHERE account_id = ?
Query 3: SELECT * FROM player_levels WHERE account_id = ?
Query 4: SELECT * FROM player_settings WHERE account_id = ?
Query 5: SELECT rs.axis, rs.score, rs.tier, rs.server_position
         FROM rank_scores WHERE account_id = ?
Query 6: SELECT cosmetic_id FROM player_cosmetics WHERE account_id = ?
         (owned cosmetics list; small result set)
Query 7: SELECT achievement_id FROM player_achievements WHERE account_id = ?
         (owned achievements list; for in-session evaluation)
```

Queries 2–7 fire concurrently (async, multiple pool threads). Total wall time is dominated by the slowest individual query, not their sum. All 7 queries are covered by primary key or indexed lookups.

For new players (Guest account creation):

```
Query 1: INSERT INTO accounts (lsgi_identifier, username, region_code, ...)
         → Returns new account_id via LAST_INSERT_ID()

Queries 2–4: INSERT INTO player_stats, player_levels, player_settings
             (account_id, ...) — insert default rows
Query 5: INSERT INTO rank_scores (account_id, axis, score, tier) × 2 rows
```

### 10.4 Round-End Write Plan

At round end, the following writes occur (async, non-blocking):

```
1. INSERT INTO rounds (ended_at, winner_team, duration_seconds)
   WHERE round_id = ? [UPDATE to complete the round record]

2. INSERT INTO round_participants (...) VALUES (...), (...), (...)...
   [Single multi-row INSERT for all participants — do NOT loop single INSERTs]

3. UPDATE player_stats SET rounds_played = rounds_played + 1,
   rounds_survived = rounds_survived + ?, ...
   WHERE account_id = ?
   [One UPDATE per player; fire concurrently via pool]

4. UPDATE player_levels SET total_xp = total_xp + ?, current_level = ?, ...
   WHERE account_id = ?
   [One UPDATE per player]

5. INSERT INTO xp_transactions (account_id, xp_amount, source_type, ...)
   [One INSERT per XP award event; may batch]

6. UPDATE rank_scores SET score = ?, tier = ?, server_position = ?
   WHERE account_id = ? AND axis = 1
   [One UPDATE per player on gameplay axis]

7. Achievement evaluation queries (async, post-round, lowest priority)
```

The total write count at round end for 24 players: approximately 7 × 24 = ~168 individual queries plus batch inserts. The pool of 16 connections will serialize these over ~200–400ms. This is acceptable as it is fully async.

### 10.5 Leaderboard Query Performance

The JumpStats leaderboard is the most frequently read complex query. Its performance must be benchmarked before launch.

Primary leaderboard query pattern:
```
SELECT jr.account_id, a.username, jr.distance, jr.sync_pct, jr.recorded_at
FROM jumpstat_records jr
INNER JOIN accounts a ON jr.account_id = a.account_id
WHERE jr.jump_type = ?
  AND jr.is_server_record = 1
  AND jr.is_valid = 1
ORDER BY jr.distance DESC
LIMIT 100
```

With only one row per jump type having `is_server_record = 1`, this query returns at most 1 row before the ORDER BY. For a paginated top-100 leaderboard showing all players' PBs (not just server records), the query becomes:

```
SELECT jr.account_id, a.username, jr.distance, jr.sync_pct, jr.recorded_at
FROM jumpstat_records jr
INNER JOIN accounts a ON jr.account_id = a.account_id
WHERE jr.jump_type = ?
  AND jr.is_pb = 1
  AND jr.is_valid = 1
ORDER BY jr.distance DESC
LIMIT 100 OFFSET ?
```

The `idx_pb_lookup` composite index on `(jump_type, distance DESC)` WHERE `is_pb = 1` must cover this query. Evaluate with EXPLAIN in the test environment.

---

## 11. SCALABILITY CONSIDERATIONS

### 11.1 Player Count Scaling

The schema is designed to support millions of rows without structural changes. The key scaling thresholds and interventions are:

| Threshold | Table | Intervention |
|---|---|---|
| > 5M rows | `round_participants` | Enable RANGE partitioning by month |
| > 5M rows | `xp_transactions` | Enable RANGE partitioning by month |
| > 100M rows | `jumpstat_tick_data` | Enable RANGE partitioning by `record_id` block |
| > 1M rows | `jumpstat_records` | Review index efficiency; consider covering index optimization |
| > 100K players | `rank_scores` | Leaderboard `server_position` recomputation becomes expensive — introduce a background job |

### 11.2 Multi-Server Architecture (Phase 2)

V1.0 is designed for a single logical database serving one or more server instances. All servers share one `accounts` table and one set of leaderboards.

The `servers` table and `server_id` foreign keys on `rounds`, `sessions`, and `jumpstat_records` mean all data is already tagged by originating server. Multi-server expansion requires no schema changes — only configuration of additional server records.

**BIGINT AUTO_INCREMENT and Multi-Server Federation:**

When multiple servers write to the same database simultaneously:
- `AUTO_INCREMENT` is safe with InnoDB — each INSERT gets a unique ID without coordination.
- No additional sharding key is needed for V1.0.

For future physical database sharding (Phase 5 scale):
- The `server_id` column on all relevant tables provides a natural shard key.
- The account-centric model means a player's data can be co-located by `account_id` on a future sharded cluster with minimal cross-shard joins.

### 11.3 Multi-Region Architecture (Phase 2+)

V1.0 uses a single database. Regional segmentation is handled by the `region_code` column on `accounts` and the `region_code` column on `servers`. No separate database per region exists in V1.0.

When a Phase 2 multi-region architecture is implemented:
- The recommended approach is **read replicas per region** (e.g., a MariaDB replica in Singapore for the SG server cluster, syncing from the primary DB in Jakarta).
- Write operations always go to the primary.
- Leaderboard reads (high volume, low latency requirement) go to the regional replica.
- This approach requires no schema changes.

### 11.4 BIGINT AUTO_INCREMENT Capacity

At the highest projected write rates:

| Table | Writes/Day (Estimate) | Years to BIGINT Overflow |
|---|---|---|
| `round_participants` | ~1,500 | 16+ billion years |
| `xp_transactions` | ~1,500 | 16+ billion years |
| `jumpstat_records` | ~10,000 | 2.5+ billion years |
| `jumpstat_tick_data` | ~2,000,000 | 12+ million years |

BIGINT UNSIGNED overflow is not a practical concern for this project's lifetime.

---

## 12. SOURCEMOD ACCESS PATTERNS

### 12.1 SourceMod SQL Constraints

SourceMod's SQL API (SQL_Query, SQL_TQuery, Database.Query) has the following constraints that directly influence schema design:

| Constraint | Impact on Design |
|---|---|
| No native UUID type support | BIGINT AUTO_INCREMENT is the only practical PK type |
| Results accessed by column index or name | Column ordering in SELECT matters; always use named columns |
| `SQL_FetchInt()` for integers, `SQL_FetchString()` for text | INTEGER columns are preferred for enum-type fields; avoid storing enums as strings in columns queried by plugin |
| No native prepared statement parameterization in older SM versions | `GM_DB_Escape()` must be called for all string inputs |
| Callbacks fire on main thread | Keep callback processing minimal — parse result, update in-memory cache, return. No heavy computation in callbacks. |
| Connection pool size limits concurrency | Round-end write cascade must not queue more queries than pool × 3 (configurable queue depth) |

### 12.2 In-Memory Caching Strategy

The plugin maintains an in-memory cache per connected player to minimize per-frame database reads.

| Cache Entry | Source Table | Loaded | Invalidated |
|---|---|---|---|
| `account_id` | `accounts` | On connect | On disconnect |
| `is_banned`, `ban_expiry` | `accounts` | On connect | On connect (re-checked each connection) |
| `username`, `region_code`, `admin_level` | `accounts` | On connect | On disconnect |
| `total_xp`, `current_level`, `xp_to_next_level` | `player_levels` | On connect | On XP award |
| All stats fields | `player_stats` | On connect | On round end write |
| All settings fields | `player_settings` | On connect | On settings change |
| `gameplay_score`, `gameplay_tier`, `gameplay_position` | `rank_scores` | On connect | On round end |
| `movement_score`, `movement_tier` | `rank_scores` | On connect | On JumpStat PB |
| Owned cosmetics list | `player_cosmetics` | On connect | On cosmetic grant |
| Earned achievements list | `player_achievements` | On connect | On achievement unlock |
| Achievement catalog (all) | `achievement_catalog` | Plugin startup | Plugin reload |
| Cosmetic catalog (all) | `cosmetic_catalog` | Plugin startup | Plugin reload |

### 12.3 Critical Query List

The following are the most performance-critical queries. Each must be benchmarked in the test environment before production deployment.

| ID | When | Query Description | Target Time |
|---|---|---|---|
| Q-01 | Player connect | Resolve steam_id_64 to account_id + ban check | < 5ms |
| Q-02 | Player connect | Load player_stats, player_levels, player_settings (concurrent) | < 10ms |
| Q-03 | Player connect | Load rank_scores (2 rows) | < 5ms |
| Q-04 | Player connect | Load player_cosmetics (N rows) | < 5ms |
| Q-05 | Round end | Multi-row INSERT into round_participants | < 20ms |
| Q-06 | Round end | Batch UPDATE player_stats per player (concurrent) | < 30ms total |
| Q-07 | JumpStat recorded | INSERT jumpstat_records + UPDATE is_pb on previous PB | < 10ms |
| Q-08 | JumpStat recorded | Check if new distance > current server record | < 5ms |
| Q-09 | !lb command | SELECT top 100 PBs for a jump type | < 20ms |
| Q-10 | !profile command | JOIN accounts + player_stats + player_levels + rank_scores | < 15ms |

---

## 13. FUTURE COMPATIBILITY PROVISIONS

### 13.1 Phase 2 Provisions Already in Schema

The following Phase 2 features have database infrastructure reserved in V1.0:

| Feature | Provision |
|---|---|
| Discord Account Link | `accounts.discord_id` column present; `linked_accounts` table created (empty) |
| Friends System | `friends` table created (empty) |
| Account Merge | `account_merge_log` table created (empty); `account_id` as canonical key means merge only requires child record re-attribution |
| Per-Country Leaderboards | `accounts.region_code` stores country-level code; regional queries require only a WHERE filter |
| Season Rankings | `rank_history` collects weekly snapshots from launch |
| Additional Languages | `player_settings.lang` is VARCHAR(5); adding a language requires no schema change |

### 13.2 Multi-Server ID Coordination (Phase 2)

When Phase 2 introduces multiple physical GM-HNS server instances writing to the same database:

- `AUTO_INCREMENT` with InnoDB handles concurrent inserts safely.
- If physical database sharding is required (Phase 5), the recommended migration path is: introduce a `server_id` prefix component to the effective key (application-layer compound key), not a schema change to PK types.
- The existing `server_id` FK on all write-heavy tables (`rounds`, `round_participants`, `sessions`, `jumpstat_records`) provides the natural partition/shard key without any schema migration.

### 13.3 Additional Jump Types (Phase 3)

`jumpstat_records.jump_type` is `TINYINT UNSIGNED` (range: 0–255). Five types are used in V1.0. Adding Phase 3 jump types (BLJ, DJ, WJ, HJ, SJ, MBH) requires only adding new ENUM values to the application-layer mapping — no schema change.

### 13.4 Additional Cosmetic Categories (Phase 3)

`cosmetic_catalog.category` is `TINYINT UNSIGNED`. Four categories used in V1.0 (knife, trail, title, tag). Adding categories requires only a new TINYINT value in the application-layer mapping and new rows in `cosmetic_catalog` — no schema change.

### 13.5 API Access (Phase 2)

The Phase 2 external REST API will connect to the database in read-only mode via a dedicated `api_reader` MySQL user with SELECT privileges only. No schema changes are required. The `account_id`-centric design means API endpoints expose internal `account_id` as the player identifier in all responses.

---

## APPENDIX A: TABLE DEPENDENCY MAP

Load order for initial schema creation (respecting FK dependencies):

```
1. servers
2. accounts
3. player_stats
4. player_levels
5. player_settings
6. sessions
7. rounds
8. round_participants
9. xp_transactions
10. achievement_catalog
11. player_achievements
12. cosmetic_catalog
13. player_cosmetics
14. rank_scores
15. rank_history
16. jumpstat_records
17. jumpstat_tick_data
18. admin_log
19. bug_reports
20. account_merge_log     (empty — Phase 2 reserved)
21. linked_accounts        (empty — Phase 2 reserved)
22. friends               (empty — Phase 2 reserved)
```

---

## APPENDIX B: ENUM VALUE REFERENCE

Complete reference of all TINYINT UNSIGNED enum values used across the schema. These must be consistent between the database schema and all SourceMod plugin modules.

### Account Type (`accounts.account_type`)
| Value | Label |
|---|---|
| 0 | guest |
| 1 | linked_steam |
| 2 | linked_discord |
| 3 | linked_both |

### Admin Level (`accounts.admin_level`)
| Value | Label |
|---|---|
| 0 | player |
| 1 | admin |
| 2 | owner |

### Jump Type (`jumpstat_records.jump_type`)
| Value | Label | Phase |
|---|---|---|
| 1 | LJ — Longjump | V1.0 |
| 2 | CJ — Countjump | V1.0 |
| 3 | MCJ — Multi-Countjump | V1.0 |
| 4 | LDJ — Ladder Jump | V1.0 |
| 5 | DBhop — Double Bhop | V1.0 |
| 6–15 | Reserved for Phase 3 jump types | Phase 3 |

### Round Type (`rounds.round_type`)
| Value | Label |
|---|---|
| 0 | standard |
| 1 | training |

### Winner Team (`rounds.winner_team`)
| Value | Label |
|---|---|
| 0 | draw |
| 1 | hiders |
| 2 | chasers |

### Participant Team (`round_participants.team`)
| Value | Label |
|---|---|
| 1 | hider |
| 2 | chaser |

### XP Source Type (`xp_transactions.source_type`)
| Value | Label |
|---|---|
| 1 | gameplay |
| 2 | jumpstat |
| 3 | training_challenge |
| 4 | achievement |

### Achievement Category (`achievement_catalog.category`)
| Value | Label |
|---|---|
| 1 | survival |
| 2 | chaser |
| 3 | movement |
| 4 | progression |
| 5 | training |
| 6 | hidden |

### Achievement Tier (`achievement_catalog.tier`)
| Value | Label |
|---|---|
| 1 | bronze |
| 2 | silver |
| 3 | gold |
| 4 | platinum |

### Cosmetic Category (`cosmetic_catalog.category`)
| Value | Label |
|---|---|
| 1 | knife |
| 2 | trail |
| 3 | title |
| 4 | tag |

### Cosmetic Acquisition Type (`cosmetic_catalog.acquisition_type`)
| Value | Label |
|---|---|
| 1 | level_reward |
| 2 | achievement_reward |
| 3 | admin_grant |
| 4 | auto_rank |

### Cosmetic Acquired Source (`player_cosmetics.acquired_source`)
| Value | Label |
|---|---|
| 1 | level |
| 2 | achievement |
| 3 | admin_grant |

### Rank Axis (`rank_scores.axis`, `rank_history.axis`)
| Value | Label |
|---|---|
| 1 | gameplay |
| 2 | movement |

### Rank Tier (`rank_scores.tier`, `rank_history.tier`)
| Value | Label | Gameplay Spec |
|---|---|---|
| 1 | Newcomer | Tier I |
| 2 | Amateur | Tier II |
| 3 | Skilled | Tier III |
| 4 | Expert | Tier IV |
| 5 | Master | Tier V |
| 6 | Grandmaster | Tier VI |
| 7 | Legend | Tier VII |
| 8 | Garuda | Tier VIII |

### Friendship Status (`friends.status`)
| Value | Label |
|---|---|
| 0 | pending |
| 1 | accepted |
| 2 | declined |
| 3 | blocked |

---

## APPENDIX C: DATABASE CONNECTION CONFIGURATION REFERENCE

| Parameter | Recommended Value | Notes |
|---|---|---|
| `innodb_buffer_pool_size` | 256MB (launch), 1GB (scale) | Set to 70–80% of available RAM if dedicated DB server |
| `innodb_log_file_size` | 256MB | Larger log reduces I/O for high-insert workloads |
| `max_connections` | 100 | SourceMod pool uses max 16; headroom for admin tools and Phase 2 API |
| `wait_timeout` | 300 | Disconnect idle connections after 5 minutes |
| `character_set_server` | utf8mb4 | Required for full Unicode support |
| `collation_server` | utf8mb4_unicode_ci | Required |
| `innodb_flush_log_at_trx_commit` | 2 | Trade 1-second crash window for significant write performance. Acceptable for this use case. |
| `sync_binlog` | 0 | Disable binlog sync for performance if binlog is not required for replication at V1.0 |
| `innodb_file_per_table` | ON | Enables individual table space management and OPTIMIZE TABLE |
| `slow_query_log` | ON | Log queries exceeding 200ms for monitoring |
| `long_query_time` | 0.2 | 200ms slow query threshold |

---

## APPENDIX D: VERSION HISTORY

| Version | Date | Author | Summary |
|---|---|---|---|
| 1.0 | June 2026 | Lead Database Architect | Initial production database specification for GM-HNS V1.0 |

---

*End of GM_DATABASE_SPEC.md*

---

> **This document is the official V1.0 database architecture specification for the Garuda Movement HNS Ecosystem.**
> All schema implementation decisions must reference and conform to this specification.
> This document is subordinate to GM_HNS_MASTER_SPEC_V1_FINAL.md on all gameplay decisions.
> Database architecture deviations require a formal amendment with version increment.
