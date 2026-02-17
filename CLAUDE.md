# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is **tvararu/azerothcore-wotlk**, a maintained fork of `mod-playerbots/azerothcore-wotlk` (itself a fork of AzerothCore, a WoW 3.3.5a server emulator, with playerbot hook points baked into the core C++ source). Our `main` branch tracks upstream's `Playerbot` branch with additional fixes and documentation. The actual `mod-playerbots` module is cloned separately into `modules/mod-playerbots/`:

```bash
git clone https://github.com/mod-playerbots/mod-playerbots.git modules/mod-playerbots
```

## Mise Tasks

All server operations go through mise. Run `mise tasks` for the full list. Key commands:

- `mise setup-modules` — clone/update all modules (source of truth for module list)
- `mise init` — generate `docker-compose.override.yml`
- `mise build` — recompile and start (needed after adding modules)
- `mise start` / `mise stop` — start/stop without rebuild
- `mise restart` — recreate containers (picks up override/env changes, unlike `docker compose restart`)
- `mise nuke --raw` — destroy DB volume (prompts for confirmation)
- `mise seed --raw` — apply module SQL, set realmlist, create admin account
- `mise cmd '<command>'` — run worldserver command via SOAP
- `mise logs` — tail worldserver logs
- `mise sql '<query>'` — run a SQL query non-interactively (use this, not `mise db`, for scripted queries)
- `mise db` — interactive MySQL shell (requires TTY, cannot be used non-interactively)
- `mise create-account <user> <pass> [--gm]` — create game account
- `mise health` — verify secrets, modules, Docker, Tailscale, and worldserver SOAP

Full setup docs in `docs/SETUP.md`.

### Secrets

`DOCKER_DB_ROOT_PASSWORD`, `ADMIN_USERNAME`, `ADMIN_PASSWORD` are age-encrypted in `mise.toml`. Decryption key at `config/secret.key` (gitignored). Set new secrets with `mise set --age-encrypt --prompt <VAR>`.

## Build Commands

### Docker (primary method)

```bash
docker compose up -d --build          # Full build + run
docker compose up -d                  # Run (no rebuild)
docker compose down                   # Stop
docker compose logs -f ac-worldserver # View logs
docker attach ac-worldserver          # Attach to worldserver console (Ctrl+P,Ctrl+Q to detach)
```

### Native build via acore.sh dashboard

```bash
./acore.sh compiler build       # or: ./acore.sh c b  — configure + compile
./acore.sh compiler configure   # or: ./acore.sh c cfg — CMake only
./acore.sh compiler compile     # or: ./acore.sh c cmp — compile only
./acore.sh compiler clean       # or: ./acore.sh c cl  — clean build files
./acore.sh compiler all         # or: ./acore.sh c a   — clean + configure + compile
```

### Direct CMake

```bash
mkdir -p var/build/obj && cd var/build/obj
cmake ../../.. -DCMAKE_INSTALL_PREFIX=$PWD/../../dist \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
  -DSCRIPTS=static -DMODULES=static -DAPPS_BUILD=all
make -j$(nproc)
make install
```

### Nix dev shell

```bash
nix develop   # Provides boost, cmake, openssl, libmysqlclient, readline, bzip2
```

### Key CMake options

| Option | Values | Default |
|---|---|---|
| `SCRIPTS` | none, static, dynamic, minimal-static, minimal-dynamic | static |
| `MODULES` | none, static, dynamic | static |
| `APPS_BUILD` | none, all, auth-only, world-only | all |
| `TOOLS_BUILD` | none, all, db-only, maps-only | none |
| `BUILD_TESTING` | ON/OFF | OFF (run with `ctest` from build dir) |
| `USE_COREPCH` / `USE_SCRIPTPCH` | ON/OFF | ON |

Build output goes to `env/dist/` (native) or is embedded in Docker images.

## Architecture

### Server Executables

- **worldserver** (`src/server/apps/worldserver/`) — game server (port 8085). Loads scripts, modules, maps, and handles all game logic.
- **authserver** (`src/server/apps/authserver/`) — login/authentication server (port 3724). Standalone, no module system.

### Source Tree Layout

```
src/
  common/           — Foundation library (crypto, logging, config, threading, collision/pathfinding)
  server/
    apps/           — worldserver and authserver entry points
    database/       — DatabaseWorkerPool<T>, prepared statements, migrations
    game/           — Core game logic (largest subtree)
      AI/           — CreatureAI base classes
      Entities/     — Object → WorldObject → Unit → Player/Creature hierarchy
      Scripting/    — ScriptMgr hook system (the primary extension point)
      Modules/      — ModuleMgr (minimal; modules register via ScriptMgr)
      Spells/       — Spell system and aura handling
      Maps/         — Map, InstanceMap, grid system
      Movement/     — MotionMaster, movement generators
      ...           — ~50 subdirectories for game subsystems
    scripts/        — Built-in scripts (Commands, dungeons, raids, spells)
    shared/         — SharedDefines.h, DBC data stores, network protocol
modules/            — External modules cloned here (initially empty)
deps/               — Third-party: boost, fmt, recast/detour, SFMT, etc.
data/sql/           — SQL schema and migrations (base/, updates/, custom/)
conf/dist/          — config.cmake (CMake options), config.sh (build settings)
```

### Entity Class Hierarchy

```
Object                              — GUID, update fields, type system
  └─ WorldObject : Object           — position, visibility, phasing, map
       ├─ Unit : WorldObject        — health, combat, auras, movement, AI
       │    ├─ Player : Unit        — player character (split across 12 .cpp files)
       │    └─ Creature : Unit      — NPCs
       │         └─ TempSummon → Minion → Guardian → Pet
       ├─ GameObject : WorldObject
       ├─ DynamicObject : WorldObject
       └─ Corpse : WorldObject
  └─ Item : Object                  — inventory items (not in world grid)
```

### Scripting / Hook System

`ScriptMgr` (singleton at `src/server/game/Scripting/ScriptMgr.h`) is the primary extension point. Script types include `PlayerScript`, `CreatureScript`, `WorldScript`, `SpellScriptLoader`, `PlayerbotScript`, and ~48 others. Each has a corresponding file in `Scripting/ScriptDefines/`.

Modules and scripts register hook handlers by subclassing these script types. The registry uses `EnabledHooks` optimization to only call scripts that override a specific hook.

### Databases

Four MySQL databases accessed via `DatabaseWorkerPool<T>`:

| Database | Connection Class | Purpose |
|---|---|---|
| `acore_world` | `WorldDatabaseConnection` | Game world data (creatures, items, quests) |
| `acore_characters` | `CharacterDatabaseConnection` | Player character data |
| `acore_auth` | `LoginDatabaseConnection` | Accounts, realm list, bans |
| `acore_playerbots` | `PlayerbotsDatabaseConnection` | Bot strategies, equipment, travel nodes (`#ifdef MOD_PLAYERBOTS`) |

Prepared statements follow the pattern `{DB}_{SEL/INS/UPD/DEL/REP}_{Description}`.

### DBC Data Stores

Each `LOAD_DBC(store, "File.dbc", "tablename_dbc")` call in `src/server/game/DataStores/DBCStores.cpp` requires a matching empty table definition at `data/sql/base/db_world/<tablename>_dbc.sql`. Missing tables cause a fatal `ER_NO_SUCH_TABLE` abort in `_HandleMySQLErrno`. Format strings in `src/server/shared/DataStores/DBCfmt.h` define column types (d=indexed PK, i=int, x=skipped uint32, s=string, f=float, b=byte, n=indexed-not-in-struct).

### Module System

Modules live in `modules/<name>/` with their own `CMakeLists.txt`. The build system (`modules/CMakeLists.txt`) auto-discovers them, generates script loaders, and copies `.conf.dist` files. Modules can be static or dynamic.

**DataMap** (`src/common/Utilities/DataMap.h`) lets modules attach custom data to core objects by string key.

Modules are managed by `mise setup-modules` (see `.mise/tasks/setup-modules` for the list). Key notes:

- `mod-individual-progression` uses old SQL directory convention (`world/` instead of `db-world/`) — the seed task handles both conventions automatically
- `mod-playerbots` also uses old SQL convention (`world/`, `characters/`, `playerbots/`) — seed maps `playerbots/` to `acore_playerbots`
- `mod-playerbots` requires `./modules:/azerothcore/modules:ro` volume mount in override (worldserver needs module SQL at runtime) and `AC_PLAYERBOTS_UPDATES_ENABLE_DATABASES: "1"` — without these it crash-loops on startup
- Module `.conf.dist` files are auto-copied to `.conf` by the entrypoint; to customize, edit the `.conf` inside `./env/dist/etc/modules/`
- `mod-individual-progression` requires `EnablePlayerSettings = 1` and `DBC.EnforceItemAttributes = 0` (set in `docker-compose.override.yml`)

### Playerbot Integration (Two-Layer)

1. **Core hooks (this repo):** 16 modified source files with `#ifdef MOD_PLAYERBOTS` guards and a `PlayerbotScript` class in ScriptMgr providing hooks like `OnPlayerbotUpdate`, `OnPlayerbotLogout`, `OnPlayerbotPacketSent`, etc. A dedicated `PlayerbotsDatabaseConnection` is also in the core.

2. **Module (cloned separately):** When `mod-playerbots` exists in `modules/`, CMake sets `-DMOD_PLAYERBOTS` on `database`, `game-interface`, and `modules` targets, activating all guarded code.

### TCP Bot Command Server

mod-playerbots runs a TCP command server on port 8888 (`AiPlayerbot.CommandServerPort`). Key internals:

- `PlayerbotCommandServer` (`modules/mod-playerbots/src/Bot/Cmd/PlayerbotCommandServer.cpp`) — TCP listener, own thread
- `RandomPlayerbotMgr::HandleRemoteCommand` (`RandomPlayerbotMgr.cpp:3414`) — request router
- `PlayerbotAI::HandleRemoteCommand` (`PlayerbotAI.cpp:5151`) — read-only status queries
- `PlayerbotAI::HandleCommand` (`PlayerbotAI.cpp:911`) — behavioral command entry point (same as whisper)
- `GET_PLAYERBOT_AI(player)` macro = `PlayerbotsMgr::instance().GetPlayerbotAI(player)` — finds any bot (random or on-demand)
- `ObjectAccessor::FindPlayerByName(name)` — looks up any online player by character name (thread-safe, shared lock)
- Port 8888 must be exposed in `docker-compose.override.yml` (and `mise init` template)

## Git Workflow

### Branching Strategy

- **`main`** — our primary development branch. All local changes live here. Rebased on `Playerbot`.
- **`Playerbot`** — clean mirror of `upstream/Playerbot`. Never commit here. Sync with: `git fetch upstream && git merge --ff-only upstream/Playerbot`
- **Feature branches** — for upstream PRs, branch off `Playerbot` (not `main`). Delete after PR merges.
- When upstream updates: sync `Playerbot`, then `git checkout main && git rebase Playerbot`
- When a PR merges upstream: the fix lands in `Playerbot` on next sync, and `main` rebases cleanly (duplicate commit drops automatically).

Commit work incrementally. Each logical unit of work should be committed promptly — do not wait for multiple nudges.

### Commits

Always run `git log -n 5` first to match existing style. Never use `--oneline` — commit bodies carry important context.

- Subject line: Conventional Commits format — `type(scope): description` (e.g. `fix(Core/Unit): Prevent crash on despawn during combat`). Common types: fix, feat, refactor, chore, docs. Scope matches the subsystem (Core/Player, Scripts/ICC, DB/SAI, etc.)
- Blank line, then 1-3 sentence description of "why" (wrap at 72 chars)
- No bullet points
- NEVER add "Co-Authored-By" or other footers

### Pull Requests

- Write a short essay (1-2 paragraphs) describing why the changes are needed
- NEVER add a Claude Code attribution footer
- PR context and investigation notes go in `docs/plans/YYYY-MM-DD-<slug>.md` for future reference

## Conventions

### Code Style

- 4-space indentation (`.editorconfig` enforced), no tabs
- PascalCase for classes, methods, and member functions (`GetPlayer()`, `HandleSpellCast()`)
- camelCase for local variables and parameters (`uint32 mapId`, `Player* player`)
- `m_` prefix for private member variables, `_` prefix also common (`m_session`, `_player`)
- UPPER_SNAKE_CASE for constants and enums (`MAX_PLAYER_LEVEL`, `SPELL_EFFECT_HEAL`)
- Braces on their own line for classes/functions, same line for control flow

### General

- C++ codebase, compiled with clang by default. C++20 features available.
- Precompiled headers (PCH) enabled by default — changes to widely-included headers trigger long rebuilds.
- Config values use the `AC_` env var prefix in Docker (dots become underscores).
- SQL migrations go in `data/sql/updates/` organized by database (db_world, db_characters, db_auth).
- The `Playerbot` branch periodically merges upstream AzerothCore `master`.

## Environment

### Git Remotes

- `origin` — `git@github.com:tvararu/azerothcore-wotlk.git` (maintained fork, push here)
- `upstream` — `https://github.com/mod-playerbots/azerothcore-wotlk.git` (PR target)

### Server Setup

- Docker-based, Tailscale access at `100.73.138.96`
- WoW 3.3.5a client at `~/games/wotlk/` (ChromieCraft, launched via Steam/Proton GE)
- Setup plan in `docs/PLAN.md`

### Docker Gotchas

- `ac-db-import` only applies base SQL on first boot — to add new base SQL tables to a running DB, either create them manually via `docker exec ac-database mysql ...` or nuke the `ac-database` volume and reimport
- Don't try to run a second worldserver instance inside the container — it will fail on DB migrations
- `docker-compose.override.yml` is gitignored — regenerate with `mise init`
- `tty: false` is set in the override so that scripted commands can write to the worldserver's stdin via `/proc/1/fd/0`
- SOAP requires `AC_SOAP_IP: "0.0.0.0"` to be reachable from the host (default binds to 127.0.0.1 inside the container)
- Docker `COPY` does not follow symlinks — module SQL with non-standard directory names must be applied from the host (handled by `mise seed`)
- `ac-db-import` only processes module `updates/` SQL, not `base/` — module base SQL is applied by `mise seed` directly via mysql
- The worldserver caches DB tables (like `acore_string`) at boot — SQL applied after startup requires `mise stop && mise start` to take effect
- The worldserver Docker image only contains the compiled binary, not module source/SQL — modules that need runtime SQL access (like mod-playerbots) require a volume mount in the override
- WoW account passwords have a 16-character client hard limit
- Worldserver stdin commands (`/proc/1/fd/0`) need `sleep 2` between dependent operations (e.g. account create then set gmlevel) — the console processes them asynchronously
- `docker compose restart` does NOT pick up changes to `docker-compose.override.yml` — use `mise restart` which does `down && up -d`
- `mise seed` is idempotent — tracks applied SQL files in a `_seed_applied` table per database; safe to re-run

### Gitignore Quirks

- `*.patch` and `*.diff` are globally gitignored (line 55-56). If adding new tracked file types that match global ignores, add a negation rule.
