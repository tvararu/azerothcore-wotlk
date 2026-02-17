# Phase 3: Playerbots (On-Demand Only)

**Date**: 2026-02-16

## Goal

Enable the mod-playerbots module with on-demand bots only (no random world bots). Integrate it into the existing mise automation so the full setup is reproducible with `mise setup-modules && mise build && mise nuke && mise start && mise seed && mise stop && mise start`.

## Decisions

- **Bot mode**: On-demand only. Random bot autologin disabled (`AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"`). Player uses `.bot add` commands to summon bots manually.
- **DB strategy**: Full nuke + rebuild + seed. Clean slate, no incremental patching.
- **Repo**: `mod-playerbots/mod-playerbots` (active org repo, 639 stars, updated daily). The old `liyunfan1223/mod-playerbots` is abandoned.
- **Automation**: Fully integrated into mise tasks (setup-modules, init, seed). No manual steps.

## What Changes

### 1. `setup-modules` task

Add mod-playerbots to the MODULES array:

```
"mod-playerbots https://github.com/mod-playerbots/mod-playerbots.git"
```

No symlink fixups needed. The module uses the old SQL directory convention (`world/`, `characters/`) which the seed task already handles.

### 2. `seed` task

Two additions:

- **Create `acore_playerbots` database** before the SQL loop. The module ships `data/sql/playerbots/create/create_mysql.sql` but it grants to `acore@localhost` (doesn't exist in Docker). Run `CREATE DATABASE IF NOT EXISTS acore_playerbots` directly instead.
- **Handle `playerbots/` SQL directory** in the SQL application loop. Map `playerbots` directory name to the `acore_playerbots` database.

### 3. `init` task

Add to the generated `docker-compose.override.yml`:

```yaml
AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"
```

### 4. Documentation

- Update CLAUDE.md: change module clone URL from `liyunfan1223/mod-playerbots` to `mod-playerbots/mod-playerbots`
- Update PLAN.md: mark Phase 3 as completed

## What Doesn't Change

- No new mise tasks needed
- No config file editing (env vars in override are sufficient)
- The existing `nuke + build + seed + stop + start` flow covers the full setup
- `docker-compose.yml` already has `AC_PLAYERBOTS_DATABASE_INFO` configured

## Key Findings

### SQL structure of mod-playerbots

- `data/sql/characters/base/` — 3 files (bot names for arena teams, guilds, characters)
- `data/sql/characters/updates/` — 1 update file
- `data/sql/playerbots/base/` — ~25 files (bot data, strategies, travel nodes, caches)
- `data/sql/playerbots/create/` — `create_mysql.sql` (creates `acore_playerbots` DB)
- `data/sql/playerbots/updates/` — ~15 update files
- `data/sql/world/base/` — 3 files (including our DBC fixes from PR #169, plus `playerbots_rpg_races`)
- `data/sql/world/updates/` — 1 update file

### Known Docker gotcha

The `ac-db-import` service does NOT create `acore_playerbots` or run module `create/` SQL. The seed task must handle this explicitly. Community reports confirm this is a common setup failure point.

### Config file

The module ships `conf/playerbots.conf.dist` with ~2000 lines of settings. Docker's entrypoint auto-copies `.conf.dist` to `.conf`. All settings can be overridden via `AC_` env vars in `docker-compose.override.yml`.
