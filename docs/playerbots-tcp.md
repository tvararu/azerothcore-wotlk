# TCP Bot Interface (port 8888)

Line-based TCP protocol for querying and commanding playerbots. One request per line (`\n`-terminated), one response per line.

```bash
echo "hp,Xiara" | nc localhost 8888       # read
echo "cmd,Xiara,Deity,follow" | nc localhost 8888  # write
```

## Commands (write)

```
cmd,<BotName>,<MasterName>,<command>\n
```

Master must be online. GM masters have full access to all bots. Returns `ok` on success.

The command field accepts any bot chat command documented in `PLAYERBOTS.md`. Examples:

```
cmd,Xiara,Deity,follow
cmd,Xiara,Deity,stay
cmd,Xiara,Deity,attack
cmd,Xiara,Deity,flee
cmd,Xiara,Deity,summon
cmd,Xiara,Deity,reset
cmd,Xiara,Deity,co +dps -tank
cmd,Xiara,Deity,nc +loot
cmd,Xiara,Deity,autogear
cmd,Xiara,Deity,stats
cmd,Xiara,Deity,who
cmd,Xiara,Deity,do attack my target
```

Compound commands use `\\` separator:

```
cmd,Xiara,Deity,co +passive\\nc +stay
```

## Queries (read)

```
<query>,<BotName>\n
```

| Query | Response format | Example response |
|---|---|---|
| `hp` | `HP%` or `HP% / TargetHP%` | `100%` or `85% / 42%` |
| `state` | `combat`, `dead`, or `non-combat` | `non-combat` |
| `position` | `X Y Z MapId Orientation \|Zone\|` | `10355.3 -6365.5 35.4 530 -0.17 \|Eversong Woods\|` |
| `target` | target name or empty | `Manawraith` |
| `tpos` | target's `X Y Z MapId Orientation` or empty | `10360.1 -6370.2 35.0 530 2.1` |
| `movement` | last move destination `X Y Z MapId Orientation` | `10355.0 -6365.0 35.4 530 0.0` |
| `strategy` | `Strategies: name1, name2, ...` | `Strategies: buff, dps assist, follow, nc` |
| `action` | last executed action name | `follow` |
| `values` | full AI context dump (long) | `(large key=value output)` |
| `travel` | travel destination, status, time left | `Destination = Grind: ... Status = travel Expire in 120s` |
| `budget` | gold breakdown by purpose | `Current money: 1g 23s ... repair \| 50s / 1g` |
| `stats` * | gold, bags, durability, XP | `1g 23s 45c\|8/20 bags\|87% dur (3s 20c)\|45%/120% xp` |
| `who` * | race, gender, spec, level, gearscore, area | `Blood Elf F Holy priest\|80\|4523\|Eversong Woods` |
| `los` * | line-of-sight entities by category | `targets:Mana Wyrm\|npcs:Innkeeper\|gos:\|players:Deity\|corpses:` |
| `quests` * | quest log with completion status | `The Dwarven Spy=incomplete\|Delivering Daffodils=complete` |
| `inventory` * | all bag items (aggregated by ID) | `Hearthstone x1 (soulbound)\|Runecloth x12\|Mana Potion x5` |
| `spells` * | active non-passive spells (filtered) | `Flash Heal\|Greater Heal\|Shadow Word: Pain\|Smite` |

Queries marked * are non-standard extensions (not in upstream mod-playerbots). The `cmd` write protocol and name-based bot lookup are also non-standard. Upstream only supports GUID-based read queries.

## Errors

| Response | Meaning |
|---|---|
| `bot not found` | No online player with that name |
| `not a bot` | Player exists but isn't a bot |
| `master not found` | Master player not online |
| `usage: cmd,BotName,MasterName,command` | Malformed cmd request |
| `invalid request: ...` | Could not parse request |

## Backwards compatibility

Old GUID format still works for read queries: `hp,12345` falls back to GUID lookup if name lookup fails.
