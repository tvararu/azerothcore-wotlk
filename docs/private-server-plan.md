# AzerothCore Server Setup Plan

## Overview

Personal solo WotLK 3.3.5a server on Arch Linux (128GB RAM), using the mod-playerbots
fork of AzerothCore for future bot support. Blizzlike 1x rates. Accessible via Tailscale.

## Phase 1: Basic Server (do this first, get in-game)

### 1. Build and start containers

```bash
cd ~/code/azerothcore-modplayerbots
docker compose up -d --build
```

First build compiles from source (~15-30 min depending on CPU). Services:
- `ac-database` — MySQL 8.4 (port 3306)
- `ac-db-import` — one-shot schema/data import
- `ac-client-data-init` — downloads maps/vmaps/mmaps/dbc (~several GB)
- `ac-worldserver` — game server (port 8085, SOAP 7878)
- `ac-authserver` — login server (port 3724)

Monitor progress: `docker compose logs -f`

### 2. Create GM account

```bash
docker attach ac-worldserver
# In the console:
account create <username> <password>
account set gmlevel <username> 3 -1
# Detach: Ctrl+P, Ctrl+Q (NEVER Ctrl+C — it kills the server)
```

### 3. Configure realmlist for Tailscale access

Get Tailscale IP:
```bash
tailscale ip -4
```

Update realmlist in the database:
```bash
mysql -h127.0.0.1 -uroot -ppassword -e \
  "UPDATE acore_auth.realmlist SET address='<TAILSCALE_IP>';"
```

### 4. Client setup

Edit `realmlist.wtf` in the WoW 3.3.5a client `Data/` directory:
```
set realmlist <TAILSCALE_IP>
```

### 5. Firewall / Tailscale

Ports needed (only over Tailscale, not public internet):
- 3724 (authserver)
- 8085 (worldserver)

If using Tailscale ACLs, ensure the playing device can reach these ports on this machine.

---

## Phase 2: Extra Modules

Clone into `modules/` directory, then rebuild:

```bash
cd ~/code/azerothcore-modplayerbots/modules

# AoE loot — loot all nearby corpses at once
git clone https://github.com/azerothcore/mod-aoe-loot.git

# Auto-learn spells on level up
git clone https://github.com/azerothcore/mod-learn-spells.git

# Progressive content unlock (T4 before T5, etc.)
git clone https://github.com/azerothcore/mod-individual-progression.git
```

Then rebuild:
```bash
cd ~/code/azerothcore-modplayerbots
docker compose up -d --build
```

**Note:** Verify these modules compile against the playerbots fork. The fork has
modified core APIs — if a module fails to build, check for API signature mismatches.
`mod-individual-progression` is the most likely to have issues since it hooks deeply
into game systems.

---

## Phase 3: Playerbots (done)

On-demand bots enabled. Random bot autologin is off — use `.bot add <name>` to summon bots manually. Managed by `mise setup-modules` (clones `mod-playerbots/mod-playerbots`), `mise init` (sets `AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "0"`), and `mise seed` (creates `acore_playerbots` DB + applies SQL).

To enable random world bots later, set `AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "1"` in `docker-compose.override.yml` and tune `AC_AI_PLAYERBOT_MIN_RANDOM_BOTS` / `AC_AI_PLAYERBOT_MAX_RANDOM_BOTS`. With 128GB RAM, hundreds of bots are comfortable.

---

## Phase 4: Ollama Bot Buddy (someday, maybe)

Repo cloned at `~/code/mod-ollama-bot-buddy`. Requires:
- Ollama running locally with a model loaded
- Playerbots enabled (Phase 3)
- Module cloned into `modules/` and rebuilt

Not a priority — just a fun experiment for later.

---

## Phase 5: Future Considerations

- **Backups**: Set up cron/systemd timer for mysqldump of `acore_auth`,
  `acore_characters`, `acore_playerbots` (world DB regenerates from source)
- **Systemd service**: If you want auto-start on boot
- **Upstream merges**: The playerbots fork merges upstream every 2-5 days.
  Periodically `git pull` and rebuild to get fixes
- **Config tuning**: XP rates, loot rules, etc. go in `docker-compose.override.yml`
  using `AC_` prefixed env vars (dots become underscores, camelCase splits)

---

## Server Style

| Setting | Value |
|---|---|
| Rates | 1x blizzlike |
| Fork | mod-playerbots/azerothcore-wotlk (Playerbot branch) |
| Host | This Arch box, Docker |
| Access | Tailscale (remote) |
| RAM | 128GB (no constraints) |
| Client | 3.3.5a enUS, ready |
| Lifecycle | Manual `docker compose up -d` / `down` |

---

## Quick Reference

```bash
# Start server
cd ~/code/azerothcore-modplayerbots && docker compose up -d

# Stop server
cd ~/code/azerothcore-modplayerbots && docker compose down

# View logs
docker compose logs -f ac-worldserver

# Attach to worldserver console
docker attach ac-worldserver

# Rebuild after changes
docker compose up -d --build
```
