# TCP Bot Commands Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend mod-playerbots' TCP command server to accept behavioral commands and name-based bot lookup, delivered as a patch file.

**Architecture:** Rewrite `RandomPlayerbotMgr::HandleRemoteCommand` to support name-based lookup via `ObjectAccessor::FindPlayerByName` and a new `cmd,BotName,MasterName,command` protocol for behavioral commands routed through `PlayerbotAI::HandleCommand`. Maintained as a git patch applied by `mise setup-modules`.

**Tech Stack:** C++ (AzerothCore/mod-playerbots), git patch, bash (mise task)

**Design doc:** `docs/plans/2026-02-16-tcp-bot-commands-design.md`

---

### Task 1: Modify HandleRemoteCommand in mod-playerbots

**Files:**
- Modify: `modules/mod-playerbots/src/Bot/RandomPlayerbotMgr.cpp:3414-3435`

**Step 1: Replace `HandleRemoteCommand` implementation**

The current function (lines 3414-3435) parses `command,GUID` and only looks up random bots. Replace it with:

```cpp
std::string const RandomPlayerbotMgr::HandleRemoteCommand(std::string const request)
{
    // Split request on commas (up to 4 parts for cmd protocol)
    std::vector<std::string> parts;
    std::string::size_type start = 0;
    std::string::size_type pos;
    while ((pos = request.find(',', start)) != std::string::npos)
    {
        parts.push_back(request.substr(start, pos - start));
        start = pos + 1;
    }
    parts.push_back(request.substr(start));

    if (parts.size() < 2)
        return "invalid request: " + request;

    // Behavioral command: cmd,BotName,MasterName,command text
    if (parts[0] == "cmd")
    {
        if (parts.size() < 4)
            return "usage: cmd,BotName,MasterName,command";

        Player* bot = ObjectAccessor::FindPlayerByName(parts[1]);
        if (!bot)
            return "bot not found";

        PlayerbotAI* botAI = GET_PLAYERBOT_AI(bot);
        if (!botAI)
            return "not a bot";

        Player* master = ObjectAccessor::FindPlayerByName(parts[2]);
        if (!master)
            return "master not found";

        // Rejoin remaining parts as command text (in case command contains commas)
        std::string commandText = parts[3];
        for (size_t i = 4; i < parts.size(); ++i)
            commandText += "," + parts[i];

        botAI->HandleCommand(CHAT_MSG_WHISPER, commandText, master);
        return "ok";
    }

    // Read query: command,BotName (with GUID fallback for backwards compat)
    std::string const& command = parts[0];
    std::string const& botIdentifier = parts[1];

    Player* bot = ObjectAccessor::FindPlayerByName(botIdentifier);
    if (!bot)
    {
        // Fall back to old GUID-based lookup
        uint32 guidCounter = atoi(botIdentifier.c_str());
        if (guidCounter)
        {
            ObjectGuid guid = ObjectGuid::Create<HighGuid::Player>(guidCounter);
            bot = GetPlayerBot(guid);
        }
    }

    if (!bot)
        return "bot not found";

    PlayerbotAI* botAI = GET_PLAYERBOT_AI(bot);
    if (!botAI)
        return "not a bot";

    return botAI->HandleRemoteCommand(command);
}
```

No new `#include` needed — `ObjectAccessor`, `GET_PLAYERBOT_AI`, and `PlayerbotAI` are all already available in this file.

**Step 2: Verify the edit compiles conceptually**

Check that:
- `ObjectAccessor::FindPlayerByName` takes `std::string const&` — yes (`ObjectAccessor.h:81`)
- `GET_PLAYERBOT_AI` is a macro for `sPlayerbotsMgr.GetPlayerbotAI(object)` — yes (`Playerbots.h:30`)
- `botAI->HandleCommand(uint32 type, std::string const text, Player* fromPlayer)` — yes (`PlayerbotAI.cpp:911`)
- `CHAT_MSG_WHISPER` is available — yes (via `SharedDefines.h`, already included transitively)

---

### Task 2: Generate the patch file

**Files:**
- Create: `patches/mod-playerbots-tcp-commands.patch`

**Step 1: Generate the patch from the module's git diff**

```bash
cd modules/mod-playerbots
git diff src/Bot/RandomPlayerbotMgr.cpp > ../../patches/mod-playerbots-tcp-commands.patch
```

**Step 2: Verify patch contents**

```bash
cat patches/mod-playerbots-tcp-commands.patch
```

Confirm it shows the diff for `HandleRemoteCommand` only.

**Step 3: Restore the module to upstream state**

```bash
cd modules/mod-playerbots
git checkout src/Bot/RandomPlayerbotMgr.cpp
```

**Step 4: Verify patch applies cleanly**

```bash
cd modules/mod-playerbots
git apply --check ../../patches/mod-playerbots-tcp-commands.patch
git apply ../../patches/mod-playerbots-tcp-commands.patch
```

---

### Task 3: Update mise setup-modules to apply patch

**Files:**
- Modify: `.mise/tasks/setup-modules`

**Step 1: Add patch application after mod-playerbots clone/pull**

After the module clone/pull loop, add a section that applies patches. Insert before the "Fixups" comment block:

```bash
# Apply patches
PATCHES_DIR="$(git rev-parse --show-toplevel)/patches"
PLAYERBOTS_DIR="$MODULES_DIR/mod-playerbots"
PLAYERBOTS_PATCH="$PATCHES_DIR/mod-playerbots-tcp-commands.patch"
if [[ -f "$PLAYERBOTS_PATCH" && -d "$PLAYERBOTS_DIR" ]]; then
  if git -C "$PLAYERBOTS_DIR" apply --check "$PLAYERBOTS_PATCH" 2>/dev/null; then
    git -C "$PLAYERBOTS_DIR" apply "$PLAYERBOTS_PATCH"
    echo "[mod-playerbots] Applied TCP commands patch"
  else
    echo "[mod-playerbots] Patch already applied or conflicts — skipping"
  fi
fi
```

The `--check` makes it idempotent: if the patch is already applied (or conflicts), it skips gracefully.

---

### Task 4: Build and test

**Step 1: Rebuild the server**

```bash
mise build
```

This recompiles with the patched mod-playerbots source. Watch for compilation errors in `RandomPlayerbotMgr.cpp`.

**Step 2: Start the server**

```bash
mise start
```

Wait for worldserver to finish loading (check `mise logs` for "AzerothCore is ready").

**Step 3: Test read queries with netcat**

Log into the game client. Add a bot with `.bot add <name>`. Then from a terminal:

```bash
echo "hp,<BotName>" | nc localhost 8888
```

Expected: a response like `100%` or `100% / 50%`.

```bash
echo "strategy,<BotName>" | nc localhost 8888
```

Expected: list of active strategies.

**Step 4: Test behavioral commands**

```bash
echo "cmd,<BotName>,<YourCharName>,follow" | nc localhost 8888
```

Expected: `ok` response, and in-game the bot should start following your character.

```bash
echo "cmd,<BotName>,<YourCharName>,stay" | nc localhost 8888
```

Expected: `ok` response, bot stops following.

**Step 5: Test error cases**

```bash
echo "cmd,FakeName,Deity,follow" | nc localhost 8888
```

Expected: `bot not found`

```bash
echo "cmd,<BotName>,FakeMaster,follow" | nc localhost 8888
```

Expected: `master not found`

```bash
echo "hp,<YourCharName>" | nc localhost 8888
```

Expected: `not a bot` (your real character isn't a bot)

**Step 6: Test backwards compatibility**

If you have the GUID of a random bot, verify the old format still works:

```bash
echo "hp,12345" | nc localhost 8888
```

---

### Task 5: Commit everything

**Step 1: Commit patch file and setup-modules change**

```bash
git add patches/mod-playerbots-tcp-commands.patch .mise/tasks/setup-modules
git commit -m "feat: extend playerbot TCP server with behavioral commands

Add a patch for mod-playerbots that extends the TCP command server
(port 8888) to accept behavioral commands and name-based bot lookup.
Protocol: cmd,BotName,MasterName,command for writes, query,BotName for
reads. Applied automatically by mise setup-modules."
```

---

### Task 6: Update PLAN.md

**Files:**
- Modify: `docs/PLAN.md`

**Step 1: Update Phase 3 to mention the TCP command interface**

Add a note to the Phase 3 section about the TCP command server being extended for external control (OpenClaw integration).

**Step 2: Commit**

```bash
git add docs/PLAN.md
git commit -m "docs: update PLAN.md with TCP bot command interface"
```
