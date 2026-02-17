# Playerbots

Full wiki: https://github.com/mod-playerbots/mod-playerbots/wiki

Playerbots are programmed to respond to chat commands. There are two types:

- **Altbots** — manually created characters you control, logged in with `.playerbots bot add`
- **Rndbots** — auto-generated bots that self-gear, level, and roam the world autonomously

## Client Addons

Install to `~/games/wotlk/Interface/AddOns/`.

- **[MultiBot](https://github.com/Wishmaster117/MultiBot)** — UI for controlling playerbots (use the Wishmaster117 fork, the original is inactive)
- **[CompactRaidFrame](https://gitlab.com/Tsoukie/compactraidframe-3.3.5)** — better raid frames for managing bot groups
- **[DBM-Warmane](https://github.com/Zidras/DBM-Warmane)** — Deadly Boss Mods for 3.3.5a

## Companion Server Modules

- **[mod-junk-to-gold](https://github.com/noisiver/mod-junk-to-gold)** — autosell gray items
- **[mod-player-bot-level-brackets](https://github.com/DustinHendrickson/mod-player-bot-level-brackets)** — control random bot level distribution (only useful with random bots enabled)
- **[mod-ollama-chat](https://github.com/DustinHendrickson/mod-ollama-chat)** — LLM-powered bot chat via Ollama (early stage, see Phase 4 in PLAN.md)

## Bot Management

| Command | Action |
|---------|--------|
| `.playerbots bot add <name1,name2>` | Log in altbots from your account or linked accounts |
| `.playerbots bot remove <name1,name2>` | Log out altbots |
| `.playerbots bot add *` | Log in all altbots in party/raid |
| `.playerbots bot remove *` | Log out all altbots in party/raid |
| `.playerbots bot addaccount <name>` | Log in all characters on an account |
| `.playerbots bot addclass <class>` | Summon a rndbot of that class (use "dk" for Death Knight) |
| `.playerbots bot list` | List your altbots |
| `.playerbots bot init=rare <name1,name2>` | Respawn bot with talents and rare gear (gearing is bugged) |

### Account Linking

Link accounts so you can control altbots across multiple accounts.

| Command | Action |
|---------|--------|
| `.playerbots account setKey <key>` | Define a security key for your account |
| `.playerbots account link <ACCOUNT> <key>` | Link another account using its security key |
| `.playerbots account linkedAccounts` | Show linked accounts |
| `.playerbots account unlink <ACCOUNT>` | Remove a linked account |

## Movement & Combat

| Command | Action |
|---------|--------|
| `follow` | Follow you |
| `stay` | Hold position |
| `attack` | Attack your target |
| `flee` | Run to you ignoring threats |
| `grind` | Attack anything visible |
| `summon` | Teleport bot to you |
| `reset` | Stop current actions |
| `leave` | Leave party |
| `release` | Release spirit when dead |
| `revive` | Revive at spirit healer |
| `disperse set <yards>` | Spread bots apart by N yards |
| `disperse disable` | Reset disperse to default |
| `give leader` | Pass party/raid leadership to master |

### Group Targeting Syntax

Commands can be prefixed with `@` selectors to target specific bots:

- `@group1 follow`, `@group2 attack` — by raid group number
- `@tank`, `@dps`, `@heal`, `@ranged`, `@rangeddps`, `@meleedps` — by role
- `@warrior`, `@paladin`, `@shaman`, etc. — by class name
- Combined: `@Group1,4` or `@group2-5,8`

### Automatic Reactions

Bots automatically respond to player actions:

- Accept quest — bots accept too
- Talk to quest giver — bots turn in completed quests
- Use meeting stone — bots teleport
- Use game object — bots use it too
- Open trade window — bots show inventory
- Invite to party/raid — bots accept
- Ready check — bots report readiness
- Mount/unmount — bots mirror you
- Enter dungeon portal — bots follow

## Target Selection (RTSC/RTI)

**RTSC** (Real-Time Strategy Control) lets you save locations and direct bots to them using the "aedm" spell. **RTI** (Raid Target Icons) focuses bots on marked targets.

| Command | Action |
|---------|--------|
| `rtsc` | Enable RTSC, receive "aedm" spell |
| `rtsc cancel` | Disable RTSC, remove spell |
| `rtsc save <#>` | Save a location after using aedm |
| `rtsc unsave <#>` | Clear a saved location |
| `rtsc go <#>` | Send bots to saved location |
| `rtsc go save` | Return bots to saved position |
| `[name/group] rtsc toggle` | Enable/disable point-click targeting for specific bots |
| `rti <icon>` | Set prioritized target (skull, cross, circle, star, square, triangle, diamond, moon) |
| `attack rti target` | Attack RTI target |
| `rti cc <icon>` | Set crowd-control target icon (default: moon) |

## Strategies

Bots use combat (co) and non-combat (nc) strategies. Toggle them with:

```
co +strategy1,-strategy2,~strategy3
nc +strategy1,-strategy2,~strategy3
co ?    -- list active combat strategies
nc ?    -- list active non-combat strategies
```

### Combat Strategies

| Strategy | Description |
|----------|-------------|
| `tank` | Threat-generating abilities |
| `tank assist` | Make tank pull mobs |
| `tank face` | Prevent target from facing ranged players |
| `dps` | DPS abilities |
| `cc` | Crowd control abilities |
| `assist` | Single target focus |
| `aoe` | Multiple target focus |
| `boost` | Use big cooldowns |
| `threat` | Avoid grabbing threat |
| `grind` | Attack any visible target |
| `heal` | Party healing focus |
| `focus` | Single target, avoid multi-target spells |
| `avoid aoe` | Dodge harmful area abilities |
| `save mana` | Healers use efficient spells when mana is low |
| `healer dps` | Healers cast damage spells with sufficient mana |
| `behind` | Move to target's rear flank |
| `passive` | Do nothing (useful for pull macros) |

### Class-Specific Combat Strategies

| Class | Strategies |
|-------|------------|
| Druid | `bear`, `cat`, `caster` (talent-based) |
| Hunter | `trap weave` (drop explosive traps during rotation) |
| Mage | `frost`, `fire` (talent-based), `firestarter` (melee with instant flamestrike) |
| Shaman | `<totem name>` (add totem to call of the elements) |
| Warlock | `meta melee` (demonology default, melee with metamorphosis) |

### Non-Combat Strategies

| Strategy | Description |
|----------|-------------|
| `food` | Start/stop eating/drinking |
| `pvp` | Toggle PvP mode |
| `loot` | Enable looting (GM-level required for rndbots) |
| `stay` | Hold position |
| `follow` | Follow master |
| `passive` | Passive mode |

### Class-Specific Non-Combat Strategies

| Class | Strategies |
|-------|------------|
| Paladin | Blessings: `bdps`, `bmana`, `bstats`, `bhealth`. Auras: `rfire`, `rfrost`, `rshadow`, `baoe`, `barmor`, `bcast`, `bspeed` |
| Priest | `rshadow` (shadow protection) |
| Hunter | Aspects: `bdps`, `bspeed`, `bmana`, `rnature` |
| Warlock | Pets: `imp`, `voidwalker`, `succubus`, `felhunter`, `felguard`. Soulstone: `ss master`, `ss self`, `ss tank`, `ss healer` |

### Raid-Specific Strategies

Automatically applied upon instance entry:

| Strategy | Instance | Notes |
|----------|----------|-------|
| `moltencore` | Molten Core | All bosses |
| `bwl` | Blackwing Lair | Auto Onyxia Scale Cloak buff, suppression device disabling, Brood Affliction clearing |
| `aq20` | Ruins of Ahn'Qiraj | Ossirian primarily |
| `naxx` | Naxxramas | Most bosses |
| `voa` | Vault of Archavon | Up to Emalon |
| `wotlk-os` | Obsidian Sanctum | OS+2 functional |
| `wotlk-eoe` | Eye of Eternity | Malygos |
| `uld` | Ulduar | All bosses except Algalon |
| `onyxia` | Onyxia's Lair | — |
| `icc` | Icecrown Citadel | Heroic mode in progress |

## Gear & Talents

| Command | Action |
|---------|--------|
| `autogear` | Auto-equip best available gear |
| `maintenance` | Enable learning spells, consumables, enchants, repairs |
| `talents` | Check current spec |
| `talents spec list` | View available specs |
| `talents spec <name>` | Switch to a spec |
| `talents apply <link>` | Apply talent build from a talent link |
| `glyphs` | List equipped glyphs with item links |
| `glyph equip <GlyphID1...>` | Apply specific glyphs |
| `reset botAI` | Reset bot settings |

## Spells

| Command | Action |
|---------|--------|
| `spells` | List bot's spells |
| `cast <spell>` | Cast a spell |
| `cast <spell> on <player>` | Cast on a specific target |
| `ss +<id>` | Exclude a spell from use |
| `ss -<id>` | Remove spell exclusion |
| `ss reset` | Clear all spell exclusions |
| `trainer` | Show learnable spells from nearby trainer |
| `trainer learn` | Learn all available spells from selected trainer |

## Items & Loot

| Command | Action |
|---------|--------|
| `nc +loot` | Enable looting |
| `ll all` | Loot everything |
| `ll normal` | Loot except bind-on-pickup |
| `ll gray` | Loot gray items only |
| `ll quest` | Loot quest items only |
| `ll skill` | Loot herbalism/mining/skinning items |
| `ll <item>` | Add item to loot list |
| `ll -<item>` | Remove item from loot list |
| `e <item>` | Equip |
| `ue <item>` | Unequip |
| `u <item>` | Use |
| `u <item> <target>` | Use on target |
| `open items` | Open inventory items with loot |
| `destroy <item>` | Destroy item |
| `roll <item>` | Roll for upgrade |
| `roll` | All bots roll |
| `s <item>` | Sell |
| `s *` | Sell all grey items |
| `s vendor` | Sell all vendorable items |
| `b <item>` | Buy |
| `2g 3s 5c` | Give you gold |
| `bank <item>` | Deposit in bank |
| `bank -<item>` | Withdraw from bank |
| `gb <item>` | Deposit in guild bank |
| `gb -<item>` | Withdraw from guild bank |

## Quests

| Command | Action |
|---------|--------|
| `quests` | Show quest summary |
| `quests all` | List all quests with links |
| `accept <quest>` | Accept a quest |
| `accept *` | Accept all quests from NPC |
| `drop <quest>` | Abandon a quest |
| `r <item>` | Choose quest reward |
| `<quest>` | Show quest objectives |
| `talk` | Interact with targeted NPC |
| `u <game object>` | Use game object |

## Pet Commands

| Command | Action |
|---------|--------|
| `pet aggressive` | Aggressive stance |
| `pet passive` | Passive stance |
| `pet defensive` | Defensive stance |
| `pet stance` | Display current stance |
| `pet attack` | Pet attack target |
| `pet follow` | Pet follow master |
| `pet stay` | Pet stay in place |

### Hunter Taming

| Command | Action |
|---------|--------|
| `tame` | Tame help |
| `tame name "<name>"` | Summon a tameable pet by name |
| `tame id "<id>"` | Summon by creature ID |
| `tame family` | Tame family help |
| `tame family "<family>"` | Randomly summon pet of family |
| `tame rename "<new name>"` | Rename current pet |

## Miscellaneous

| Command | Action |
|---------|--------|
| `stats` | Show stat summary |
| `who` | Display bot race, spec, talents, class, level, item level, zone |
| `who <profession>` | Show profession skill level |
| `los` | List visible objects, items, creatures, NPCs |
| `home` | Set hearth at innkeeper |
| `/w <botname> help` | Bot help menu |

### Overrides

Force a bot to perform an action immediately:

| Command | Action |
|---------|--------|
| `do attack` | Attack target |
| `do attack my target` | Attack player's target |
| `do follow` | Follow immediately |
| `do stay` | Stay immediately |

## Macros

### Pull Macro

Let the tank build aggro before DPS engages (requires `/in` command from Slashin or ElvUI addon):

```
/p @dps co +passive
/p @heal co +passive
/p @tank attack
/in 8 /p @dps co -passive
/in 8 /p @heal co -passive
```

### Bloodlust Control

```
-- Disable bloodlust/heroism:
/p @shaman ss +2825,32182

-- Re-enable:
/p @shaman ss -2825,32182
```

### Movement Modes

**Flee mode** — bots follow you away from danger:
```
/p reset
/p nc -stay,+follow,+passive
/p co +passive
/p do follow
```

**Assist mode** — bots follow and attack your target:
```
/p nc -stay,+follow,-passive
/p co -passive
/p do follow
```

**Stay mode** — bots hold position while assisting:
```
/p nc -follow,+stay,-passive
/p co +passive
/p do stay
```

### Focus Specific Creature

```
/target Web Wrap
/stopmacro [noharm][dead]
/script SetRaidTarget("target", 8)
```

## Console Commands

Server-side commands run via the worldserver console or SOAP (`mise cmd`).

| Command | Action |
|---------|--------|
| `playerbot pmon toggle` | Enable/disable performance monitor |
| `playerbot pmon stack` | Show cumulative performance data |
| `playerbot pmon tick` | Show cycle performance averages |
| `playerbot pmon reset` | Reset performance monitor |
| `playerbot rndbot stats` | Print rndbot statistics |
| `playerbot rndbot reload` | Reload playerbots.conf |
| `playerbot rndbot update` | Trigger full tick |
| `playerbot rndbot init` | Re-roll rndbots |
| `playerbot rndbot clear` | Reset rndbots to starting level |
| `playerbot rndbot level` | Level up all rndbots by 1 |
| `playerbot rndbot refresh` | Revive, reset AI, re-gear while keeping level |
| `playerbot rndbot teleport` | Teleport rndbots to appropriate leveling area |
| `playerbot rndbot reset` | Clear acore_playerbots table (requires restart) |
| `playerbot rndbot change_strategy` | Re-roll grinding vs RPG behavior |
| `playerbot rndbot revive` | Revive, refresh, teleport (**BUGGED**: doubles rndbots) |
| `playerbot rndbot grind` | Conditional teleport (**BUGGED**: crashes server) |
| `playerbot bot self` | Toggle yourself as a bot |
| `playerbot bot initself` | Re-roll your character (use carefully) |

## Configuration Reference

Config file: `env/dist/etc/modules/playerbots.conf` (copied from `.conf.dist` on first boot).

### Key Settings

```ini
# General
AiPlayerbot.Enabled = 1
AiPlayerbot.AllowPlayerBots = 1
AiPlayerbot.AllowGuildBots = 1

# Random bots
AiPlayerbot.RandomBotAccountPrefix = "rndbot"
AiPlayerbot.RandomBotMinLevel = 1
AiPlayerbot.RandomBotMaxLevel = 80
AiPlayerbot.RandomBotMaps = 0,1,530,571
AiPlayerbot.RandombotStartingLevel = 5
AiPlayerbot.RandomBotFixedLevel = 0
AiPlayerbot.DisableRandomLevels = 0
AiPlayerbot.AutoTeleportForLevel = 1
AiPlayerbot.SyncLevelWithPlayers = 0
AiPlayerbot.RandomBotMaxLevelChance = 0.01
AiPlayerbot.DeleteRandomBotAccounts = 0

# Gear (quality: 1=normal, 2=uncommon, 3=rare, 4=epic, 5=legendary)
AiPlayerbot.AutoGearQualityLimit = 4
AiPlayerbot.AutoGearScoreLimit = 0
AiPlayerbot.AutoGearCommand = 1
AiPlayerbot.MaintenanceCommand = 1

# Questing
AiPlayerbot.SyncQuestWithPlayer = 1
AiPlayerbot.AutoDoQuests = 1

# Chat (all disabled = quiet bots)
AiPlayerbot.EnableBroadcasts = 0
AiPlayerbot.RandomBotTalk = 0
AiPlayerbot.RandomBotEmote = 0
AiPlayerbot.RandomBotSuggestDungeons = 0
AiPlayerbot.EnableGreet = 0
AiPlayerbot.ToxicLinksRepliesChance = 0
AiPlayerbot.ThunderfuryRepliesChance = 0
AiPlayerbot.GuildRepliesRate = 0
AIPlayerbot.GuildFeedback = 0
AiPlayerbot.RandomBotSayWithoutMaster = 0
```

### Distance Settings (yards)

```ini
AiPlayerbot.FarDistance = 20.0
AiPlayerbot.SightDistance = 75.0
AiPlayerbot.SpellDistance = 28.5
AiPlayerbot.ShootDistance = 26.0
AiPlayerbot.ReactDistance = 150.0
AiPlayerbot.GrindDistance = 75.0
AiPlayerbot.HealDistance = 38.5
AiPlayerbot.LootDistance = 25.0
AiPlayerbot.FleeDistance = 8.0
AiPlayerbot.TooCloseDistance = 5.0
AiPlayerbot.MeleeDistance = 1.5
AiPlayerbot.FollowDistance = 1.5
AiPlayerbot.WhisperDistance = 6000.0
AiPlayerbot.ContactDistance = 0.5
AiPlayerbot.AoeRadius = 10
AiPlayerbot.RpgDistance = 200
AiPlayerbot.AggroDistance = 22
```

### Update Intervals

```ini
AiPlayerbot.RandomBotUpdateInterval = 20
AiPlayerbot.RandomBotCountChangeMinInterval = 1800
AiPlayerbot.RandomBotCountChangeMaxInterval = 7200
AiPlayerbot.MinRandomBotInWorldTime = 3600
AiPlayerbot.MaxRandomBotInWorldTime = 1209600
AiPlayerbot.MinRandomBotRandomizeTime = 7200
AiPlayerbot.MaxRandomBotRandomizeTime = 1209600
AiPlayerbot.RandomBotsPerInterval = 60
AiPlayerbot.MinRandomBotReviveTime = 60
AiPlayerbot.MaxRandomBotReviveTime = 300
AiPlayerbot.MinRandomBotTeleportInterval = 3600
AiPlayerbot.MaxRandomBotTeleportInterval = 18000
AiPlayerbot.RandomBotInWorldWithRotationDisabled = 31104000
```

### Performance: Bot Activity Profiles

Bots apply 12 sequential rules to determine if they are active (always active in BGs/instances/combat, near real players, in guilds with real players, on friend lists, etc.). The `BotActiveAlone` setting controls what percentage of remaining bots stay active:

**Profile 1** — best for high bot counts:
```ini
AiPlayerbot.BotActiveAlone = 10
AiPlayerbot.botActiveAloneSmartScale = 1
AiPlayerbot.botActiveAloneSmartScaleWhenMinLevel = 1
AiPlayerbot.botActiveAloneSmartScaleWhenMaxLevel = 80
```

**Profile 2** — default, 100% activity with auto-adjustment:
```ini
AiPlayerbot.BotActiveAlone = 100
AiPlayerbot.botActiveAloneSmartScale = 1
```

**Profile 3** — all active regardless of performance:
```ini
AiPlayerbot.BotActiveAlone = 100
AiPlayerbot.botActiveAloneSmartScale = 0
```

### Database Performance

```ini
PlayerbotsDatabase.WorkerThreads = 1
PlayerbotsDatabase.SynchThreads = 2
```

### MySQL Tuning (example for 64GB RAM)

```ini
innodb_buffer_pool_size = 32G
innodb_io_capacity = 500
innodb_io_capacity_max = 2500
innodb_use_fdatasync = ON
innodb_buffer_pool_instances = 12
innodb_log_buffer_size = 32M
binlog_expire_logs_seconds = 432000
transaction_isolation = "READ-COMMITTED"
```

Optional: `skip-log-bin` reduces writes by 75-90% but carries risk.

### Recommended Hardware

- Memory: 16GB minimum, 32GB+ preferred
- CPU cores: 4 minimum, 6+ preferred
- CPU speed: 3000MHz minimum, 4400MHz+ preferred

Use `.server info` to check server latency — should stay under 70-80ms.

### Worldserver Settings for Playerbots

```ini
Quests.IgnoreAutoAccept = 1
PreloadAllNonInstancedMapGrids = 0
SetAllCreaturesWithWaypointMovementActive = 0
DontCacheRandomMovementPaths = 0
MapUpdate.Threads = 4    # or 6
MapUpdateInterval = 10
MinWorldUpdateTime = 1
PlayerLimit = 0
LeaveGroupOnLogout.Enabled = 1
```

## Raid Completion Status

### Vanilla

| Raid | Status | Notes |
|------|--------|-------|
| Molten Core | Completable | Strategies for all bosses |
| Blackwing Lair | Completable | Auto Onyxia Scale Cloak, suppression devices, Brood Affliction |
| Zul'Gurub | Completable | No strategies needed, basic knowledge sufficient |
| Ruins of Ahn'Qiraj | Completable | Strategy for Ossirian only |
| Ahn'Qiraj (AQ40) | WIP | No strategies implemented |
| Naxxramas (60) | Not supported | — |

### Burning Crusade

| Raid | Status | Notes |
|------|--------|-------|
| Karazhan | Completable | All bosses except Chess Event (manual) |
| Magtheridon's Lair | Completable | Strategies implemented |
| Gruul's Lair | Completable | Both bosses covered |
| Serpentshrine Cavern | Partial | Strategy in PR, some bosses need work |
| Tempest Keep | Not completable | Cannot progress past A'lar |
| Hyjal Summit | Completable | No strategies but defeatable |
| Black Temple | Partial | Council and Illidan lack strategies |
| Zul'Aman | Unknown | — |
| Sunwell Plateau | Unknown | — |

### Wrath of the Lich King

| Raid | Status | Notes |
|------|--------|-------|
| Naxxramas | Completable | Most bosses have strategies |
| Vault of Archavon | WIP | Needs more strategies |
| Obsidian Sanctum | Completable | Up to OS+2 |
| Eye of Eternity | Completable | Malygos strategy |
| Ulduar | WIP | Up to Yogg-Saron covered |
| Trial of the Crusader | WIP | Needs strategies |
| Onyxia's Lair | Completable | Strategy for Onyxia |
| Icecrown Citadel | Completable | Heroic mode in progress |
| Ruby Sanctum | Unknown | — |

## Raid Strategy Tips

A few highlights — see the [full raid strategy guide](https://github.com/mod-playerbots/mod-playerbots/wiki/Playerbot-Raid-Strategy-Guide) for boss-by-boss details.

**General tips:**
- Bots auto-apply raid strategies on instance entry
- Use `@ranged disperse set <yards>` to spread ranged bots for AoE mechanics
- Use skull/cross marks (RTI) to control kill order
- Use RTSC to position bots for specific mechanics (e.g., Nefarian P2, Netherspite beams)
- Bloodlust/heroism can be disabled with spell exclusion macros and re-enabled for burn phases

**Karazhan highlights:**
- Attumen: bots stack behind boss within Berserker Charge minimum range
- Maiden: tank moves boss to healer during Repentance to break stun with Holy Ground
- Shade of Aran: bots freeze during Flame Wreath, run to edge for Magnetic Pull
- Netherspite: tank bots block red beam, DPS block blue (rotate at 24 stacks), healers block green
- Prince: Enfeebled bots run from Shadow Nova, avoid Infernal Hellfire

**Naxxramas:** Use `/ra @ranged disperse set 20` on Anub'Rekhan to avoid Impale.

**Vault of Archavon:** Set Wintergrasp control with `.bf switch 1` and `.bf timer 1 0h00m01s`.

## Clearing All Random Bots (SQL)

To fully remove all random bots from the database (nuclear option):

```sql
-- acore_playerbots
DELETE FROM playerbots_random_bots;
DELETE FROM playerbots_account_type;

-- acore_characters: delete bot characters and all related data
-- (cascades through 30+ tables: achievements, inventory, pets, quests, etc.)
DELETE FROM characters WHERE account IN (
    SELECT id FROM acore_auth.account WHERE username LIKE 'RNDBOT%'
);

-- acore_auth
DELETE FROM account WHERE username LIKE 'RNDBOT%';
```

Alternatively, use the console command `playerbot rndbot reset` (requires server restart).
