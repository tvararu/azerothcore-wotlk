# Phase 3: Playerbots Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enable mod-playerbots with on-demand bots (no random world bots), fully integrated into mise automation.

**Architecture:** Four existing files get small edits (setup-modules, seed, init, CLAUDE.md), one doc gets a section rewrite (PLAN.md). No new files. The module is already cloned at `modules/mod-playerbots/`.

**Tech Stack:** Bash (mise tasks), Docker Compose YAML, Markdown

---

### Task 1: Add mod-playerbots to setup-modules

**Files:**
- Modify: `.mise/tasks/setup-modules:8-12`

**Step 1: Edit the MODULES array**

Add mod-playerbots as the last entry in the array. In `.mise/tasks/setup-modules`, change the MODULES block from:

```bash
MODULES=(
  "mod-aoe-loot        https://github.com/azerothcore/mod-aoe-loot.git"
  "mod-learn-spells    https://github.com/azerothcore/mod-learn-spells.git"
  "mod-individual-progression https://github.com/ZhengPeiRu21/mod-individual-progression.git"
)
```

to:

```bash
MODULES=(
  "mod-aoe-loot        https://github.com/azerothcore/mod-aoe-loot.git"
  "mod-learn-spells    https://github.com/azerothcore/mod-learn-spells.git"
  "mod-individual-progression https://github.com/ZhengPeiRu21/mod-individual-progression.git"
  "mod-playerbots      https://github.com/mod-playerbots/mod-playerbots.git"
)
```

**Step 2: Verify the task parses correctly**

Run: `bash -n .mise/tasks/setup-modules`
Expected: no output (syntax OK)

**Step 3: Commit**

```bash
git add .mise/tasks/setup-modules
git commit -m "chore: add mod-playerbots to setup-modules"
```

---

### Task 2: Create acore_playerbots database in seed

**Files:**
- Modify: `.mise/tasks/seed:7-8`

**Step 1: Add database creation before the SQL loop**

In `.mise/tasks/seed`, after the `apply_sql` function definition (line 12) and before the `for mod_dir` loop (line 14), add:

```bash
# Create acore_playerbots database (mod-playerbots ships create_mysql.sql but it
# grants to acore@localhost which doesn't exist in Docker — just create the DB directly)
echo "Creating acore_playerbots database..."
docker exec ac-database mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" -e \
  "CREATE DATABASE IF NOT EXISTS acore_playerbots DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;" 2>/dev/null
```

The full file after the `apply_sql` function should read:

```bash
apply_sql() {
  local db="$1" file="$2"
  docker exec -i ac-database mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" "$db" < "$file" 2>/dev/null
}

# Create acore_playerbots database (mod-playerbots ships create_mysql.sql but it
# grants to acore@localhost which doesn't exist in Docker — just create the DB directly)
echo "Creating acore_playerbots database..."
docker exec ac-database mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" -e \
  "CREATE DATABASE IF NOT EXISTS acore_playerbots DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;" 2>/dev/null

for mod_dir in "$REPO_ROOT"/modules/mod-*/; do
```

**Step 2: Verify syntax**

Run: `bash -n .mise/tasks/seed`
Expected: no output (syntax OK)

**Step 3: Commit**

```bash
git add .mise/tasks/seed
git commit -m "chore(seed): create acore_playerbots database before SQL import"
```

---

### Task 3: Handle playerbots SQL directory in seed

**Files:**
- Modify: `.mise/tasks/seed:19-24`

**Step 1: Add playerbots case to the database mapping**

In `.mise/tasks/seed`, change the `case` statement from:

```bash
    case "$(basename "$(dirname "$db_dir")")" in
      world|db-world)   db="acore_world" ;;
      auth|db-auth)     db="acore_auth" ;;
      characters|db-characters) db="acore_characters" ;;
      *) continue ;;
    esac
```

to:

```bash
    case "$(basename "$(dirname "$db_dir")")" in
      world|db-world)   db="acore_world" ;;
      auth|db-auth)     db="acore_auth" ;;
      characters|db-characters) db="acore_characters" ;;
      playerbots)       db="acore_playerbots" ;;
      *) continue ;;
    esac
```

**Step 2: Verify syntax**

Run: `bash -n .mise/tasks/seed`
Expected: no output (syntax OK)

**Step 3: Commit**

```bash
git add .mise/tasks/seed
git commit -m "chore(seed): map playerbots SQL directory to acore_playerbots database"
```

---

### Task 4: Disable random bots in init

**Files:**
- Modify: `.mise/tasks/init:18-24`

**Step 1: Add bot config to the override template**

In `.mise/tasks/init`, change the heredoc from:

```yaml
      # Enable SOAP API for scripted commands
      AC_SOAP_ENABLED: "1"
      AC_SOAP_IP: "0.0.0.0"
EOF
```

to:

```yaml
      # Enable SOAP API for scripted commands
      AC_SOAP_ENABLED: "1"
      AC_SOAP_IP: "0.0.0.0"
      # Playerbots: on-demand only (no random bots roaming the world)
      AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"
EOF
```

**Step 2: Verify syntax**

Run: `bash -n .mise/tasks/init`
Expected: no output (syntax OK)

**Step 3: Commit**

```bash
git add .mise/tasks/init
git commit -m "chore(init): disable random bot autologin in override template"
```

---

### Task 5: Update CLAUDE.md module clone URL

**Files:**
- Modify: `CLAUDE.md:10`

**Step 1: Replace the old URL**

Change:

```bash
git clone https://github.com/liyunfan1223/mod-playerbots.git modules/mod-playerbots
```

to:

```bash
git clone https://github.com/mod-playerbots/mod-playerbots.git modules/mod-playerbots
```

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update mod-playerbots clone URL to active org repo"
```

---

### Task 6: Update PLAN.md to reflect Phase 3 completion

**Files:**
- Modify: `docs/PLAN.md:96-118`

**Step 1: Rewrite the Phase 3 section**

Replace the entire Phase 3 block (lines 96-118) from:

```markdown
## Phase 3: Playerbots (later)

Clone the playerbots module:
```bash
cd ~/code/azerothcore-modplayerbots/modules
git clone https://github.com/liyunfan1223/mod-playerbots.git
```

Create `docker-compose.override.yml` to configure bots:
```yaml
services:
  ac-worldserver:
    environment:
      # Start with on-demand bots only (no random world bots)
      AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"
      # When ready for populated world, set to "1" and tune these:
      # AC_AI_PLAYERBOT_MIN_RANDOM_BOTS: "50"
      # AC_AI_PLAYERBOT_MAX_RANDOM_BOTS: "200"
```

Rebuild: `docker compose up -d --build`

With 128GB RAM, you can comfortably run hundreds of bots if desired.
```

with:

```markdown
## Phase 3: Playerbots (done)

On-demand bots enabled. Random bot autologin is off — use `.bot add <name>` to summon bots manually. Managed by `mise setup-modules` (clones `mod-playerbots/mod-playerbots`), `mise init` (sets `AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"`), and `mise seed` (creates `acore_playerbots` DB + applies SQL).

To enable random world bots later, set `AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "1"` in `docker-compose.override.yml` and tune `AC_AI_PLAYERBOT_MIN_RANDOM_BOTS` / `AC_AI_PLAYERBOT_MAX_RANDOM_BOTS`. With 128GB RAM, hundreds of bots are comfortable.
```

**Step 2: Commit**

```bash
git add docs/PLAN.md
git commit -m "docs: mark Phase 3 playerbots as done in PLAN.md"
```

---

### Task 7: Regenerate override and verify

**Step 1: Delete existing override so init regenerates it**

```bash
rm docker-compose.override.yml
mise init
```

Expected: "Created docker-compose.override.yml"

**Step 2: Verify the new override contains bot config**

```bash
grep -q "RANDOM_BOT_AUTOLOGIN" docker-compose.override.yml && echo "OK" || echo "MISSING"
```

Expected: `OK`

**Step 3: Rebuild, nuke, seed, restart**

```bash
mise build            # Recompile with MOD_PLAYERBOTS
# Wait for "ready..."
mise nuke --raw       # Clean slate
mise start            # Fresh DB import
# Wait for "ready..."
mise seed --raw       # Creates acore_playerbots + applies all module SQL
mise stop && mise start   # Restart to load new data
```

**Step 4: Verify playerbots loaded**

```bash
mise cmd '.server info'
```

Expected: server responds (confirming it's running with playerbots compiled in)

```bash
mise cmd '.bot add alliance'
```

Expected: either success or "you must be in-game" error (confirming the bot command is registered)

**Step 5: Commit all task changes together (no file changes here, just a verification step)**

No commit needed — all files were committed in Tasks 1-6.
