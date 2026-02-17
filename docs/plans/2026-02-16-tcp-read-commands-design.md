# Extended TCP Read Commands — Design

## Problem

OpenClaw can query basic bot state via TCP (hp, position, strategy, etc.) and send behavioral commands. But richer information — inventory, spells, quests, gear, line-of-sight — is only available via in-game whisper commands that produce multi-line WoW-formatted output. OpenClaw needs this data in a machine-readable TCP format.

## Solution

Add six new read queries to `PlayerbotAI::HandleRemoteCommand`, following the same pattern as existing queries (`hp`, `position`, `strategy`). Each reads bot state directly and returns a single pipe-delimited plain-text line. No WoW color codes, no item/spell links, no dependency on the action/trigger pipeline.

## Protocol

All queries use the existing TCP protocol: `<query>,<BotName>\n` → single-line response.

### `stats,<BotName>`

Bot resources: gold, bag space, durability, XP.

Format: `<gold>|<free>/<total> bags|<dur>% dur (<repair_cost>)|<xp>%/<rest>% xp`

Gold and repair cost use human-readable format: `4g 3s 2c` (zero parts omitted).

Example: `1g 23s 45c|8/20 bags|87% dur (3s 20c)|45%/120% xp`

XP section omitted at max level.

### `who,<BotName>`

Bot identity summary.

Format: `<race> <gender> <class> <spec>|<level>|<gearscore>|<area>`

Example: `Human F Holy Priest|80|4523|Eversong Woods`

### `los,<BotName>`

Line-of-sight entities.

Format: `targets:<name>,<name>,...|npcs:<name>,...|gos:<name>,...|players:<name>,...|corpses:<name>,...`

Example: `targets:Manawraith,Arcane Patroller|npcs:Innkeeper Jovia|gos:Mailbox|players:|corpses:`

Empty categories still present (distinguishes "none" from "error").

### `quests,<BotName>`

Quest log with completion status.

Format: `<name>=<status>|<name>=<status>|...`

Status is `complete` or `incomplete`. Empty string if no quests.

Example: `The Dwarven Spy=incomplete|Delivering Daffodils=complete`

### `inventory,<BotName>`

All items in bags (aggregated by item ID).

Format: `<name> x<count>|<name> x<count>|...`

Soulbound items get a `(soulbound)` suffix. Empty string if bags are empty.

Example: `Hearthstone x1|Runecloth x12 (soulbound)|Mana Potion x5`

### `spells,<BotName>`

All active non-passive spells for the current spec.

Format: `<name>|<name>|...`

Example: `Flash Heal|Greater Heal|Renew|Shadow Word: Pain|Smite`

## Implementation

All new logic goes into `PlayerbotAI::HandleRemoteCommand` in the existing patch file (`patches/mod-playerbots-tcp-commands.patch`). No new files, no action class dependencies, no threading changes beyond what already exists.

A small `formatMoney(uint32 copper)` helper converts copper to `Xg Ys Zc` format (omitting zero parts).

## What's intentionally excluded

- WoW color codes and clickable links (not useful for TCP consumers)
- Reagent/crafting details for spells (add a `crafts` query later if needed)
- Quest travel details (available via existing `travel` query)
- Per-item durability breakdown (aggregate percentage is sufficient)
- Inventory filtering by item name (dump everything, let the client filter)
- LOS trigger category (internal AI concept, not useful for OpenClaw)
