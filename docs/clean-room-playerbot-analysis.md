# Feasibility Analysis: Clean Playerbot Patch Layer + Clean Room Module

## Executive Summary

Having cloned and diffed all three repos (upstream AzerothCore, the mod-playerbots/azerothcore-wotlk Playerbot branch, and the mod-playerbots module itself), here’s what I found:

**The patch layer is surprisingly small.** The actual functional changes to AzerothCore’s core game engine (ignoring whitespace reformatting and stale script version drift) total **~2,660 lines across 92 files**, plus **~140 lines across 12 database infrastructure files**, plus **3 new files**. Most changes are 1–5 line accessor additions. The patch is surgical and well-scoped.

**The module is enormous.** mod-playerbots is **190K lines of C++ across 1,172 files**, with 1.5M lines of SQL data (mostly pre-computed travel node paths). A clean room rewrite of equivalent functionality is a multi-year effort. But a strategically scoped v1 that covers the essentials is achievable in months.

**The automation story is strong.** The fork’s changes are highly localized and use `#ifdef MOD_PLAYERBOTS` guards in many places already. A well-designed patch system with CI would handle upstream rebases with minimal manual intervention.

-----

## Part 1: The Surgical Patch Layer

### What actually needs patching

I categorized every modified file between upstream and the Playerbot branch. Stripping away whitespace-only changes and stale raid script divergence (where the fork simply hasn’t merged upstream’s latest Ulduar/Naxx rewrites), the **real** patch breaks down as:

|Subsystem                                               |Files|Functional Lines|Nature of Changes                                                                                                                          |
|--------------------------------------------------------|-----|----------------|-------------------------------------------------------------------------------------------------------------------------------------------|
|Entities (Player, Unit, Item, Creature, GameObject)     |23   |~723            |Public accessor additions, method signature tweaks, `BotCanUseItem()`, `SetMovement()`, `IsInChannel()`, `ResetSpeakTimers()`              |
|Globals (ObjectMgr)                                     |2    |~450            |Bot account/character management queries, equipment cache lookups                                                                          |
|Handlers (Chat, Trade, Character, Bank, Auth, Petitions)|7    |~295            |Hyperlink validation bypass for bot links, bot session handling in trade/character creation                                                |
|Battlegrounds (AB, AV, EY, IC, Arena)                   |9    |~181            |Public getters (`GetAVNodeInfo`, node accessors), bot-specific BG logic                                                                    |
|Scripting (ScriptMgr, ScriptDefines)                    |8    |~166            |`PlayerbotScript` base class, 9 new hooks, `OnPacketReceived`, `OnPlayerAfterUpdate`                                                       |
|Guilds                                                  |2    |~147            |Made `_HasRankRight`, `_GetRankRights`, `_MemberHasTabRights` public; added `HandleSetEmblem`/`HandleSetRankInfo` overloads                |
|Server (WorldSession)                                   |4    |~141            |Constructor `is_bot` parameter, `IsBot()`, `LoginQueryHolder` moved to header, `CharacterCreateInfo` public constructor, `GetPacketQueue()`|
|DataStores                                              |2    |~102            |DBC accessor additions for bot gear/spell lookups                                                                                          |
|Movement                                                |5    |~101            |`MotionMaster` additions for bot pathfinding control                                                                                       |
|Chat                                                    |3    |~80             |Bot chat routing, channel access                                                                                                           |
|AI (SmartAI, PetAI)                                     |5    |~66             |Bot-aware pet behavior, SmartAI extensions                                                                                                 |
|Loot                                                    |2    |~52             |Bot loot distribution hooks                                                                                                                |
|World                                                   |5    |~47             |`IWorld` interface addition, `AddQueryHolderCallback`, DB revision tracking                                                                |
|Database Infrastructure                                 |12   |~140            |`PlayerbotsDatabase` connection pool, `DatabaseLoader` extension, `DBUpdater` hooks                                                        |
|Other (Maps, Pools, Spells, DungeonFinding, etc.)       |14   |~90             |Minor: LFG queue bot checks, pool spawn hooks, spell corrections                                                                           |

**New files added (3):**

- `PlayerbotsDatabase.h` / `PlayerbotsDatabase.cpp` — dedicated database connection with ~100 prepared statements
- `PlayerbotsScript.cpp` — ScriptDefines implementation for the PlayerbotScript base class

**Ifdef guards already present:** 25 `#ifdef MOD_PLAYERBOTS` blocks across the fork, concentrated in WorldSession, MotionMaster, IWorld, World, and database infrastructure. This is a good sign — the fork authors already partially designed for conditional compilation.

### The patch design

The changes fall into **five distinct categories**, which I’d structure as five layered patch files (or a single structured patch with clear sections):

**Layer 1: Hook Infrastructure (~166 lines)**
The cleanest, most upstreamable part. Add `PlayerbotScript` to ScriptMgr, the 9 virtual hooks (`OnPlayerbotUpdate`, `OnPlayerbotCheckLFGQueue`, `OnPlayerbotPacketSent`, etc.), `OnPacketReceived` to ServerScript, `OnPlayerAfterUpdate` to PlayerScript, and database lifecycle hooks to DatabaseScript. These follow existing AzerothCore patterns exactly and could plausibly be PRed upstream.

**Layer 2: WorldSession Bot Support (~141 lines)**
The `is_bot = false` constructor parameter, `IsBot()`, `GetPacketQueue()`, moving `LoginQueryHolder` to the header, and making `CharacterCreateInfo` publicly constructable. This is the most architecturally significant change — it’s what allows fake sessions for bots.

**Layer 3: Database Connection (~140 lines + 2 new files)**
`PlayerbotsDatabase` connection pool, `DatabaseLoader` extension, `DBUpdater` support. This is self-contained and entirely behind `#ifdef MOD_PLAYERBOTS`.

**Layer 4: Accessor Exposure (~1,200 lines across ~50 files)**
The bulk of the patch by file count. Adding public getters to Group (`GetTargetIcon`, `GetRolls`), Unit (`GetFactionReactionTo`, `SendPlaySpellVisual`), Player (`BotCanUseItem`, `SetMovement`, `IsInChannel`), Guild (making private methods public), Battleground (node info accessors), etc. Each individual change is 1–5 lines. These are “please expose this internal state” changes.

**Layer 5: Handler Modifications (~295 lines across 7 files)**
Bot-aware behavior in ChatHandler (hyperlink validation bypass), CharacterHandler (bot character creation), TradeHandler (bot trading), AuthHandler (bot login flow), BankHandler, PetitionsHandler. These are the messiest because they modify control flow rather than just adding accessors.

### Automation strategy

```
upstream (azerothcore/azerothcore-wotlk master)
    ↓ [GitHub Action: triggered on upstream push]
    ↓ git fetch upstream && git rebase upstream/master
    ↓ Apply patch series (git am or quilt)
    ↓ If conflict: open issue with conflict details, halt
    ↓ If clean: compile test (CMake + build)
    ↓ If builds: tag new fork release, push
fork (your-org/azerothcore-wotlk-botready)
    ↓ [modules/ directory]
    ↓ git submodule: your-clean-room-playerbots
working server
```

**Conflict frequency estimate:** Based on upstream’s commit patterns (~200–250 commits/month, 30–43 authors), the patch would conflict roughly **2–4 times per month**, concentrated in:

- Player.h/Player.cpp (most active upstream file)
- WorldSession.h (periodic upstream refactors — the fork already diverges on `TradeStatusInfo`, instance packet handling)
- ScriptMgr.h (new hooks added upstream regularly)

Most conflicts would be trivial (context drift, not semantic conflicts). The WorldSession area is the highest risk because upstream is actively refactoring packet handling (they moved to `WorldPackets::Instance` structs that the fork hasn’t adopted).

**Estimated setup effort:** 2–3 weeks for one developer to:

- Extract the clean patch from the current fork diff
- Structure it into the 5 layers
- Set up GitHub Actions CI (rebase + build + test)
- Write documentation for each patch section
- Create conflict resolution runbook

**Ongoing maintenance:** ~2–4 hours/month resolving conflicts, assuming the patch stays stable.

### Upstreaming strategy

Layers 1 and partially Layer 4 are the most upstreamable. The conversation with AzerothCore would go:

1. **Start with individual hook PRs** (Layer 1). “We’d like `OnPlayerAfterUpdate` for per-tick module processing.” These follow existing patterns and have clear justification.
1. **Then accessor exposure PRs** (Layer 4). “Can `Group::m_targetIcons` have a public getter?” These are low-risk, small, and useful beyond bots.
1. **The WorldSession changes** (Layer 2) are the hardest sell. “We need to construct fake sessions without network connections.” This is where the philosophical discussion lives.
1. **Database infrastructure** (Layer 3) is self-contained behind ifdefs and probably stays in the patch permanently.

Each successful upstream merge shrinks the patch. Realistically, you might get Layer 1 and half of Layer 4 upstream over 6–12 months, reducing the patch from ~2,660 to ~1,500 lines.

-----

## Part 2: The Clean Room Module

### What mod-playerbots actually contains

|Component     |Lines     |Files   |What It Does                                                                                           |
|--------------|----------|--------|-------------------------------------------------------------------------------------------------------|
|**AI/Base**   |64,114    |574     |Action/trigger/value framework, strategy engine, movement, targeting, chat, formations, generic actions|
|**AI/Raid**   |39,729    |138     |Per-boss raid strategies (ICC, Ulduar, Naxx, OS, EoE, etc.) — the most labor-intensive content         |
|**AI/Class**  |26,999    |191     |Per-class combat rotations, talent specs, buff priorities for all 10 WotLK classes                     |
|**AI/Dungeon**|7,871     |154     |Per-dungeon strategies (positioning, boss mechanics, trash handling)                                   |
|**AI/World**  |2,230     |10      |World quest, outdoor PvP, travel strategies                                                            |
|**Bot**       |25,669    |48      |RandomPlayerbotMgr, PlayerbotFactory (character creation/gearing), travel nodes, equipment scoring     |
|**Mgr**       |17,550    |26      |PlayerbotMgr (per-player bot manager), PlayerbotAI (per-bot AI driver), spell repository, config       |
|**Script**    |2,147     |12      |ScriptMgr hook implementations, command handlers                                                       |
|**Util**      |1,704     |9       |Chat helpers, performance profiling, item/quest filtering                                              |
|**Db**        |415       |8       |Database helpers                                                                                       |
|**SQL Data**  |1.5M lines|57 files|Travel node paths (1.4M lines), bot names (100K), equipment cache, speech texts                        |

### Clean room rewrite: what to keep, what to redesign

**Keep the architecture (it’s good):**

- Action/Trigger/Value pattern for bot decision-making
- Strategy stacking (combat + non-combat strategies compose)
- Per-class strategy hierarchy
- The travel node graph concept

**Redesign fundamentally:**

1. **LLM-first decision layer.** The current system is pure rule-based (IF threat > X AND health < Y THEN action Z). Build a hybrid where the rule engine handles frame-by-frame combat (latency-sensitive) but an LLM handles strategic decisions: “should I queue for a BG?”, “what dungeon should I run?”, “how should I respond to this chat?”, “should I help this player?” The LLM doesn’t need to run every tick — it runs asynchronously and sets high-level goals that the rule engine executes.
1. **Provider-agnostic LLM integration.** Abstract behind an interface: `ILLMProvider` with implementations for Ollama (local), Anthropic API, OpenAI API, and a null provider (pure rule-based fallback). Configuration per-bot or per-bot-tier (your main party bots get Claude, random world bots get local llama or rules-only).
1. **Personality and memory system.** Each bot gets a personality seed (race/class/level/faction/random traits) that feeds into LLM system prompts. Conversations and relationships persist in the playerbot database. This is where the LLM integration pays off most — bots that remember you helped them last week and act accordingly.
1. **Modular raid/dungeon strategies.** The current system has 40K lines of hand-coded raid strategies. Instead: define a DSL or data format for encounter mechanics, and generate bot behavior from it. An LLM could even help: “describe the Mimiron encounter phases” → generate strategy data. Still need hand-tuned strategies for hard content, but the 80% case (stand here, don’t stand in fire, interrupt this) can be data-driven.
1. **Clean WorldSession abstraction.** Instead of modifying the WorldSession constructor, create a `BotSession` wrapper that holds the minimal state a bot needs. The patch layer exposes the constructor parameter; the module uses it through a clean factory.

### Phased implementation plan

**Phase 0 — Patch Layer (2–3 weeks)**
Extract and structure the patch from the existing fork. Set up CI. Verify it compiles and runs with zero modules (just the hooks, no bots).

**Phase 1 — Skeleton Module (4–6 weeks)**

- Bot lifecycle: create, login, logout, persist
- WorldSession factory for bot sessions
- Basic AI loop: register for `OnPlayerbotUpdate`, drive per-bot ticks
- Character factory: random name, race, class, basic equipment
- `.bot add` / `.bot remove` commands
- **No combat AI yet.** Bots log in, stand around, follow you.

**Phase 2 — Combat Foundation (6–8 weeks)**

- Action/Trigger/Value framework (clean room from the pattern, not the code)
- Strategy engine with composable strategy stacks
- 3 classes fully implemented: Warrior (tank), Priest (healer), Mage (DPS)
- Basic threat/aggro awareness
- Follow + assist + defend behaviors
- Can complete a normal dungeon with manual guidance

**Phase 3 — Class Completeness (8–12 weeks)**

- Remaining 7 classes: Paladin, Warlock, Rogue, Hunter, Shaman, Druid, Death Knight
- Talent spec awareness (2–3 specs per class)
- Buff/debuff management
- Basic group composition logic

**Phase 4 — LLM Integration (4–6 weeks)**

- `ILLMProvider` interface + Ollama/Anthropic/OpenAI implementations
- Personality generation system
- Chat response system (whisper, party, guild)
- Strategic decision-making (what to do when idle)
- Memory/relationship persistence
- Configuration: which bots get LLM, rate limiting, cost controls

**Phase 5 — World Simulation (6–8 weeks)**

- Random bot population system (bots that exist independently)
- Travel node system (can reuse the SQL data from mod-playerbots — it’s generated, not creative code)
- Quest completion behavior
- Auction house participation
- Guild creation/management

**Phase 6 — Raid/Dungeon Strategies (ongoing)**

- Data-driven encounter strategy format
- Hand-tuned strategies for major raids
- This is where 40K lines of the original lives — it’s never “done”

### Total effort estimate

|Phase          |Duration  |FTE|Notes                    |
|---------------|----------|---|-------------------------|
|0: Patch Layer |2–3 weeks |1  |Extraction + CI          |
|1: Skeleton    |4–6 weeks |1  |Foundation               |
|2: Combat      |6–8 weeks |1  |Core gameplay            |
|3: Classes     |8–12 weeks|1–2|Parallelizable per class |
|4: LLM         |4–6 weeks |1  |Your differentiator      |
|5: World Sim   |6–8 weeks |1  |Making it feel alive     |
|6: Raid/Dungeon|Ongoing   |1+ |Content, not architecture|

**Solo developer (you, part-time ~4hrs/day alongside NHS contract):**

- Phase 0–2 (usable MVP): ~4–5 months
- Phase 0–4 (LLM-enabled, all classes): ~8–10 months
- Phase 0–5 (full world simulation): ~12–14 months

**With 1 additional contributor:**

- MVP in ~2–3 months
- Full feature parity minus raid strategies in ~6–8 months

### What you CAN’T shortcut

- **The 10 class implementations** (~27K lines in original). Every class has unique spell priorities, cooldown management, resource systems. You can make this cleaner and more data-driven, but someone still has to encode “Frostbolt is the filler, Ice Lance procs on Fingers of Frost, use Icy Veins at pull” for every spec.
- **Encounter-specific strategies** (~40K lines in original). “On Mimiron Phase 2, spread for Rocket Strike, stack for Laser Barrage, kill Frost Bombs during Phase 4 transition.” This is content work, not engineering. It’s also what makes bots actually useful in raids vs. standing in fire.
- **The travel node graph** (1.4M lines SQL). This is pre-computed pathfinding data for the entire game world. You can likely reuse the existing data (it’s generated from mmaps, not creative work) or regenerate it with a tool.

### What you CAN shortcut (LLM-assisted development)

- **Class rotations**: Feed the class spell list + WotLK theorycraft to an LLM, generate initial rotation logic, then hand-tune.
- **Encounter strategies**: Describe the boss fight in natural language, have the LLM generate the strategy data structure, review and correct.
- **Bot chat/personality**: This is literally what LLMs are for.
- **Equipment scoring**: LLM can analyze stat weights per spec and generate gear preference logic.

-----

## Part 3: Risk Assessment

### Low risk

- Patch extraction and CI setup — straightforward git mechanics
- AI framework design — well-understood patterns, mod-playerbots proves the architecture works
- LLM integration — you have direct experience with this

### Medium risk

- **Upstream AzerothCore refactors.** If they restructure WorldSession or ScriptMgr significantly, the patch needs major rework. Monitor their roadmap.
- **Combat AI parity.** Getting bots to “not be stupid” in dungeons requires extensive testing and tuning. The original has years of community bug reports baked in.
- **Eluna incompatibility persists.** Your patch inherits the same virtual method signature issues. If you want Eluna support, you need to design the hooks differently (this is solvable but adds scope).

### High risk

- **Scope creep.** 190K lines of original module is a gravitational well. Define your MVP ruthlessly and ship Phase 2 before expanding.
- **Solo maintainer burnout.** The patch + module + upstream tracking + LLM integration is a lot of surface area for one person working 4 hours/day. Phase 0–2 is fine solo; beyond that, you’ll want help.

-----

## Recommendation

**Do it, but phase aggressively.** The patch layer is a 2–3 week project that immediately provides value — it gives you a clean, automated, documented foundation that the current mod-playerbots fork lacks. Even if you never write a line of the clean room module, a well-maintained patch layer with CI is useful.

For the module, the LLM integration is your genuine differentiator. The WoW emulation community has had rule-based bots for 15 years. Nobody has shipped bots that can hold a conversation, remember relationships, and make strategic decisions via an LLM. Phase 4 is where you should be sprinting toward — Phases 1–3 are just the necessary foundation to get there.

Start with the patch. Ship it as a public repo. Write a single blog post. See if anyone shows up to help with Phase 3 (class implementations) while you focus on the LLM layer.
