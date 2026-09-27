# Upgrade record

Read this first before the next upgrade. The newest upgrade is at the top.
The checklist at the end is the reusable part.

## 2026-09-27: Playerbot 2026-07-03 → 2026-09-20

### Versions

| Component | Before | After |
|---|---|---|
| Core base (`upstream/Playerbot`) | `c62dc7377` (2026-07-03) | `7f12e89ee` (2026-09-20) |
| Our `main` | `5250061bb` (tag `pre-upgrade-2026-09-27`) | `0f99b15a1` |
| mod-playerbots | `a119a6de` (2026-07-03) | `7bae1b5c` (2026-09-20) |
| mod-ah-bot | `a680cc1` | `c11d831` |
| mod-aoe-loot | `2ddf6ff` | `57279b6` |
| mod-individual-progression | `b7b6869` | `0180b91` |
| mod-learn-spells | `016b92d` | `016b92d` |
| Images | `:pinned-20260705` (also `:pre-upgrade-20260927`) | `:pinned-20260927` |
| Install directory | `~/code/azerothcore-modplayerbots` (kept for rollback) | `~/code/azerothcore-modplayerbots-2026-09` |

Upstream `azerothcore/azerothcore-wotlk` master was `baefaab94` (2026-09-27).
The Playerbots fork lagged it by 135 commits (merge base 2026-09-20).

The pre-upgrade audit (what changed upstream, risks, Playerbots facts) is in
[docs/upgrades/2026-09-27-audit.md](docs/upgrades/2026-09-27-audit.md).

### Backups

All in `~/backups/azerothcore/2026-09-27/` (mode 700, on the LUKS root;
auth holds password verifiers, so never `/mnt/aux`).

| File | Content |
|---|---|
| `db/acore_{auth,characters,world,playerbots}.sql.gz` | Online dumps, `--single-transaction --routines --triggers --events --hex-blob --databases`, taken with the server running |
| `db-final/*.sql.gz` | Dumps taken with world and auth stopped, just before the switch. **Use these to roll back** |
| `files/config.tar.gz` | `docker-compose.override.yml`, `.env`, `mise.toml`, `mise.local.toml`, `config/secret.key`, `env/dist/etc/`, `patches/`, `.mise/`, `.claude/settings.local.json` |
| `files/module-shas.txt` | Module commits before the upgrade |
| `files/mod-playerbots-live.diff` | The patched module tree diff (equal to the patch file) |
| `files/git-refs.txt` | Local `main` and GitHub `main` before the upgrade |
| `files/Errors.log.baseline` | 40-line `Errors.log` from the old server, for comparison |

Verified: `gzip -t` on each dump, table counts (auth 23, characters 112,
world 316, playerbots 31), and a test restore of auth and characters into
scratch schemas gave 158 accounts and 1,516 characters, the same as live.

Old images are also tagged `:pre-upgrade-20260927`, so no rebuild can
overwrite them. Git tags on GitHub: `pre-upgrade-2026-09-27` (the deployed
`main`) and `pre-upgrade-2026-09-27-github` (GitHub's old `main`, which was
force-pushed over).

### Steps

Before any change:

```bash
B=~/backups/azerothcore/2026-09-27; mkdir -p $B/db $B/files
cd ~/code/azerothcore-modplayerbots
mise exec -- bash -c 'for db in acore_auth acore_characters acore_world acore_playerbots; do
  docker exec -e MYSQL_PWD="$DOCKER_DB_ROOT_PASSWORD" ac-database mysqldump -uroot \
    --single-transaction --routines --triggers --events --hex-blob --set-gtid-purged=OFF \
    --databases $db | gzip -6 > '"$B"'/db/$db.sql.gz; done'
for f in $B/db/*.gz; do gzip -t $f; zcat $f | tail -1; done
tar czf $B/files/config.tar.gz docker-compose.override.yml .env mise.local.toml mise.toml \
  config/secret.key env/dist/etc patches .mise .claude/settings.local.json
for s in worldserver authserver db-import client-data; do
  docker tag acore/ac-wotlk-$s:pinned-20260705 acore/ac-wotlk-$s:pre-upgrade-20260927; done
```

Git (the deployed `main` had been rebased on 2026-07-05 and never pushed):

```bash
git tag -a pre-upgrade-2026-09-27 main -m "..."
git tag -a pre-upgrade-2026-09-27-github origin/main -m "..."
git push origin pre-upgrade-2026-09-27 pre-upgrade-2026-09-27-github
git branch -f Playerbot upstream/Playerbot && git push origin Playerbot   # fast-forward
```

New install next to the old one (the old tree's `env/dist/etc` and
`modules/` are bind-mounted into the running containers, so it stays
untouched):

```bash
cd ~/code
git clone --branch main azerothcore-modplayerbots azerothcore-modplayerbots-2026-09
cd azerothcore-modplayerbots-2026-09
git remote set-url origin git@github.com:tvararu/azerothcore-wotlk.git
git remote add upstream https://github.com/mod-playerbots/azerothcore-wotlk.git
mkdir -p config; cp -p ../azerothcore-modplayerbots/config/secret.key config/
cp -p ../azerothcore-modplayerbots/{docker-compose.override.yml,mise.local.toml} .
mise exec -- git fetch origin; mise exec -- git fetch upstream
mise exec -- git rebase --onto upstream/Playerbot c62dc7377 main
# edit .mise/tasks/setup-modules: mod-playerbots pin -> 7bae1b5c...
mise setup-modules            # clones modules, runs apply-patches (failed, see Problems)
# port the patch, then:
git -C modules/mod-playerbots diff --src-prefix=i/ --dst-prefix=w/ > patches/mod-playerbots/gm-reply-target.patch
git commit ...                # pin bump; patch port
mise exec -- git push --force-with-lease=main:<old origin/main sha> origin main
```

Point the new tree at the same Compose project, so it uses the same
database volume, and at a new image tag:

```bash
# first line of docker-compose.override.yml:
name: azerothcore-modplayerbots
echo "DOCKER_IMAGE_TAG=pinned-20260927" > .env
mise exec -- docker compose config --volumes   # must show azerothcore-modplayerbots_ac-database
```

Build on 8 of 16 cores while the old server runs (the Dockerfile hard-codes
`-j $(nproc)+1`; a buildx builder pinned to 8 CPUs makes `nproc` 8):

```bash
docker buildx create --name acore-8cpu --driver docker-container --driver-opt cpuset-cpus=0-7
BUILDX_BUILDER=acore-8cpu mise exec -- docker compose build \
  ac-db-import ac-authserver ac-client-data-init ac-worldserver
```

Build took 11.5 min on 8 cores (16:35–16:47). `worldserver --version` in the
new image: `rev. 0f99b15a1e0a+`. The builder `acore-8cpu` is kept for the
next upgrade (`docker buildx rm acore-8cpu` to drop it).

Config: a fresh `env/dist/etc` gets only `.dist` files from the image. The
old install also had `.conf` copies (equal to `.dist`) for three modules;
without them a module uses its code defaults. So, as before:

```bash
E=env/dist/etc   # filled from the image's /azerothcore/env/ref/etc
cp $E/authserver.conf.dist $E/authserver.conf
for f in individualProgression mod_aoe_loot mod_learnspells; do
  cp $E/modules/$f.conf.dist $E/modules/$f.conf; done
```

New mod-individual-progression `.dist` changes behaviour:
`EnforceGroupRules` 1 → 0; `ExcludeAccounts*` renamed to `BotAccounts*`
(still `^RNDBOT.*`); health-adjustment keys removed; new keys
`DisableQuestMarkers = 1`, `MaxMonsterSight = 1`. mod_aoe_loot adds
`AOELoot.MailEnable = 0`.

Rehearsal on a scratch copy (isolated network, no published ports; the
live DB was not touched):

```bash
docker network create acore-rehearsal
docker run -d --name acore-rehearsal-db --network acore-rehearsal -e MYSQL_ROOT_PASSWORD=rehearsal mysql:8.4
zcat $B/db/*.sql.gz | docker exec -i ... mysql -uroot         # 69 s for all 4
docker run --rm --network acore-rehearsal -e AC_*_DATABASE_INFO=acore-rehearsal-db;... \
  acore/ac-wotlk-db-import:pinned-20260927                     # 25 s, rc 0
docker run -d --name acore-rehearsal-world --network acore-rehearsal --env-file <live env, DB host swapped> \
  -v <new modules>:ro -v azerothcore-modplayerbots_ac-client-data:...:ro acore/ac-wotlk-worldserver:pinned-20260927
# SOAP from a curl container on the same network
docker rm -fv acore-rehearsal-world acore-rehearsal-db; docker network rm acore-rehearsal
```

Results: all updates applied (auth 6, characters 4, world 315, playerbots 3);
158 accounts and 1,516 characters kept; world initialized in 6 s;
`Errors.log` 1 line (an empty pool, 1072) against 40 on the old server;
SOAP `server info`, `account create` and `account delete` work. One
db-import warning: `2026_01_24_00.sql` is recorded as applied but was
removed upstream (proc system refactor, #24233); harmless.

### Switch-over (done 2026-09-27)

Downtime 17:12–17:23 BST (11 min). No human online at the start.

```bash
cd ~/code/azerothcore-modplayerbots
mise exec -- docker compose stop ac-worldserver ac-authserver
# final dumps -> $B/db-final/ (same mysqldump loop), gzip -t each
# counts -> $B/files/counts-before.txt
mise stop
cd ~/code/azerothcore-modplayerbots-2026-09
mise start                      # failed: ac-client-data-init exit 1, see Problems
mkdir -p env/dist/data
echo "DOCKER_VOL_DATA=./env/dist/data" >> .env
mise start                      # downloaded client data v20.0 (1.2 GB, 9 min), then started
```

### Verification

- Worldserver `rev. 0f99b15a1e0a+`, world initialized in 7 s, no
  `MMAP ... expected v20` lines, `Errors.log` 1 line (empty pool 1072) against
  40 before.
- db-import: auth 6, characters 4, world 315 queries; worldserver applied 3
  playerbots updates.
- Ports 3724, 8085, 7878, 8888, 3306 listening. `mise health` passes.
- SOAP (TCFACTORY): `account create UPGTEST1` and `account delete UPGTEST1` work.
  tuicraft-factory `/health`: auth, world, DB and SOAP up.
- Counts (`$B/files/counts-{before,after}.txt`): accounts 161, characters
  1,519, items 34,857, `character_spell` 159,413 and guilds 20 are unchanged;
  human characters unchanged. `pet_spell` −287 and `pet_spell_cooldown`
  −1,112 are the orphan deletes of `2026_08_15_00.sql`.
  `playerbots_random_bots` −500 are `add` rows, which mod-playerbots deletes
  at every startup (old and new) and writes again as bots log in.
- Peon evals: pending (maintainer).

### Switch-over runbook

Downtime starts at step 2. Expected 10–15 min.

1. Tell the maintainer; wait for the go.
2. Stop world and auth in the **old** tree, keep the DB up:
   `cd ~/code/azerothcore-modplayerbots && mise exec -- docker compose stop ac-worldserver ac-authserver`
3. Final dumps to `$B/db-final/` (same command as above), `gzip -t`, and
   counts: accounts, characters, `pet_spell`, `pet_spell_cooldown`.
4. `mise stop` in the old tree (removes containers, keeps volumes).
5. `cd ~/code/azerothcore-modplayerbots-2026-09 && mise start`. db-import
   applies the pending updates, then auth and world start. Watch
   `docker logs -f ac-db-import`, then `mise logs`.
6. Verify (below).

### Rollback runbook

Schema updates are forward-only: never start the old images on the new
schema. Restore the dumps first.

```bash
cd ~/code/azerothcore-modplayerbots-2026-09 && mise stop
cd ~/code/azerothcore-modplayerbots          # .env still says pinned-20260705
mise exec -- docker compose up -d ac-database
B=~/backups/azerothcore/2026-09-27/db-final
mise exec -- bash -c 'for db in acore_auth acore_characters acore_world acore_playerbots; do
  docker exec -e MYSQL_PWD="$DOCKER_DB_ROOT_PASSWORD" ac-database mysql -uroot -e "DROP DATABASE IF EXISTS $db"
  zcat '"$B"'/$db.sql.gz | docker exec -i -e MYSQL_PWD="$DOCKER_DB_ROOT_PASSWORD" ac-database mysql -uroot
done'
mise start && mise health
```

The restore of all four takes about 70 s (measured in the rehearsal).
Then check counts against `db-final`. If the `:pinned-20260705` tag is ever
lost, retag from `:pre-upgrade-20260927`.

### Problems and fixes

- **The deployed branch was not on GitHub.** `main` was rebased on
  2026-07-05 and never pushed; GitHub had pre-rebase copies. Fixed by tagging
  both states, then `--force-with-lease` (approved by the maintainer).
- **`CLAUDE.md` / `AGENTS.md` conflict.** Upstream added its own
  `AGENTS.md` and made `CLAUDE.md` an `@AGENTS.md` import. Kept ours:
  `CLAUDE.md` is the project doc and `AGENTS.md` is a symlink to it.
  Upstream's agent notes are not imported. Expect this conflict again if
  upstream edits either file.
- **`gm-reply-target.patch` failed at hunk 2.** Upstream `HandleCommands`
  now erases null owners before the command check. Re-added the block after
  that check, dropped a stray blank-line hunk, regenerated the patch.
  (The audit said hunk 3 would not compile. That was wrong: the `Player*`
  overload of `HandleCommand` it patches still exists next to the `Player&`
  one.)

- **Client data v19 → v20.0.** The core bumped the mmap generator to v20
  (#25720, #26697). The new core rejects v19 `.mmtile` files (no pathfinding),
  so `ac-client-data-init` tries to download v20.0. It failed with
  `data.zip: Permission denied`: the `ac-client-data` volume is owned by root
  (filled in February) and the container runs as uid 1000. The rehearsal
  showed the `MMAP ... expected v20` lines, but I missed them. Fixed
  additively: `DOCKER_VOL_DATA=./env/dist/data` in `.env` (a host directory,
  3.1 GB) so the old v19 volume stays for rollback. Cost: 9 min of extra
  downtime for the download. Next time, download the data before the
  switch-over.
- **Tools outside this repo point at the tree by path.**
  `~/srv/tuicraft-factory/sweep.sh` ran `mise -C ~/code/azerothcore-modplayerbots`
  (the old tree). Changed to the new tree after the switch-over. The old copy
  is at `files/sweep.sh.pre-upgrade` in the backup. It still worked because
  both trees use the same containers, but it would have failed when the old
  tree was removed.

## Checklist for the next upgrade

1. Read this file. Check what runs: `docker logs ac-worldserver | grep rev`,
   and that the deployed branch is on GitHub (`git status -sb`).
2. List local-only state: unpushed commits, `git status --ignored`, module
   working trees vs `patches/`.
3. `git fetch upstream`; count commits; read new `data/sql/updates/db_*`
   (look for `DELETE`, `DROP`, `ALTER`) and `*.conf.dist` diffs. Count the rows
   each deleting migration hits.
4. Back up: 4 dumps, config tar, retag images `:pre-upgrade-<date>`, push
   `pre-upgrade-<date>` tags. Test-restore auth + characters into scratch
   schemas and compare counts.
5. New clone next to the old one; copy `config/secret.key`, override (add
   `name: azerothcore-modplayerbots`), `mise.local.toml`; new `.env` tag.
6. Rebase `main` onto `upstream/Playerbot`; bump the mod-playerbots pin to
   the module master of the same day; `mise setup-modules`; fix patches.
7. Push, build on the 8-CPU builder.
8. Compare `VERSION` in `inst_download_client_data`
   (`apps/installer/includes/functions.sh`) with `env/dist/data/data-version`.
   If it changed, download the new data into a new directory before the
   switch-over (run the new `ac-client-data` image against it).
9. Rehearse on a scratch DB copy; grep its `Server.log` for `MMAP`, `ERROR`,
   `expected`.
10. Switch over and verify as above.
11. Point outside tools at the new tree:
    `grep -rn azerothcore-modplayerbots ~/srv`.
12. Keep the old tree and images until the maintainer says they can go.
