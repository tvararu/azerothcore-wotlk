# TCP Bot Commands: Bidirectional Command Server

## Problem

OpenClaw (an external AI agent) can reach the game server via SOAP and run GM commands like `.pinfo` and `.gps`, but has no way to control playerbots. Bot commands are received through in-game chat hooks (whisper, group, guild, channel) which require a real player client connection. There is no console or SOAP command to send a whisper.

mod-playerbots has an existing TCP command server (port 8888) designed for monitoring — it accepts read-only status queries (position, HP, strategy, etc.) but cannot issue behavioral commands like follow, attack, or strategy changes.

## Solution

Extend the existing TCP command server to:

1. Accept character names instead of raw GUIDs (friendlier for external tools).
2. Find any bot (random or on-demand) via the global `PlayerbotsMgr` registry instead of only random bots via `RandomPlayerbotMgr`.
3. Route behavioral commands through the existing `PlayerbotAI::HandleCommand` path, which queues them for the bot's AI engine — the same path used by in-game whispers.

## Protocol

The server listens on the port configured by `AiPlayerbot.CommandServerPort` (default 8888). Each request is a single line terminated by `\n`. The server responds with one line per request.

### Read queries

```
<query>,<BotName>\n
```

Supported queries: `state`, `position`, `tpos`, `movement`, `target`, `hp`, `strategy`, `action`, `values`, `travel`, `budget`.

Examples:

```
state,Botname
hp,Botname
strategy,Botname
```

### Behavioral commands

```
cmd,<BotName>,<MasterName>,<command text>\n
```

The master player must be online. The bot's security system checks the master's permissions (GMs get full access). The command text supports the full ~130 command vocabulary: `follow`, `attack`, `stay`, `co +dps`, `nc +heal`, etc.

Examples:

```
cmd,Botname,Deity,follow
cmd,Botname,Deity,attack
cmd,Botname,Deity,co +dps -tank
```

Response is `ok` on success, or an error string (`bot not found`, `master not found`, `not a bot`).

### Backwards compatibility

The old GUID-based format (`command,12345`) still works — if the second field parses as a number and no player is found by that name, it falls back to GUID lookup via `RandomPlayerbotMgr::GetPlayerBot`.

## Changes

### mod-playerbots (patched)

**`src/Bot/RandomPlayerbotMgr.cpp` — `HandleRemoteCommand`**

Replace the current implementation (~20 lines) with:

1. Split request on commas.
2. If the request starts with `cmd,` and has 4 fields: parse `botName`, `masterName`, `commandText`. Look up both players via `ObjectAccessor::FindPlayerByName`. Get the bot's AI via `PlayerbotsMgr::instance().GetPlayerbotAI(bot)`. Call `botAI->HandleCommand(CHAT_MSG_WHISPER, commandText, masterPlayer)`. Return `ok`.
3. Otherwise (read query): 2 fields, `command` and `botName`. Look up bot by name via `ObjectAccessor::FindPlayerByName`, then get AI via `PlayerbotsMgr::instance().GetPlayerbotAI(bot)`. Fall back to GUID lookup if name lookup fails. Delegate to `botAI->HandleRemoteCommand(command)`.

**`src/Bot/PlayerbotAI.cpp` — `HandleRemoteCommand`**

No changes. Stays read-only.

**`src/Bot/Cmd/PlayerbotCommandServer.cpp`**

No changes. TCP server, threading, session handling all stay as-is.

### This repo

**`patches/mod-playerbots-tcp-commands.patch`**

The diff for `RandomPlayerbotMgr.cpp`, generated with `git diff` from within the module directory. Checked into version control.

**`.mise/tasks/setup-modules`**

After cloning/pulling mod-playerbots, apply the patch:

```bash
git -C "$target" apply "$REPO_ROOT/patches/mod-playerbots-tcp-commands.patch"
```

Idempotent: check `git apply --check` first, skip if already applied.

## Threading model

No changes to the existing model. The TCP server runs on its own thread (spawned by `PlayerbotCommandServer::Start`). For read queries, `HandleRemoteCommand` reads bot state that the world thread writes — this is the existing behavior, already safe enough for monitoring purposes.

For behavioral commands, `HandleCommand` pushes onto the bot's `chatCommands` queue (a `std::list<ChatCommandHolder>`). The world thread's `PlayerbotAI::HandleCommands` drains this queue on the bot's update tick. This producer/consumer split is the same mechanism used by in-game chat — the TCP thread is just another producer.

## Security

The bot's existing `PlayerbotSecurity::CheckLevelFor` runs inside `HandleCommand`. It requires a valid `Player*` with a `WorldSession`. GMs get `PLAYERBOT_SECURITY_ALLOW_ALL`. For on-demand bots, only the bot's master has full access. For random bots, group membership and gear score checks apply.

SOAP authentication protects the SOAP interface. The TCP command server has no authentication — it binds to `0.0.0.0` by default. This is an existing condition, not introduced by this change. Access should be restricted to localhost or Tailscale.

## Usage from OpenClaw

```bash
# Query bot state
echo "hp,Botname" | nc localhost 8888

# Send commands
echo "cmd,Botname,Deity,follow" | nc localhost 8888
echo "cmd,Botname,Deity,attack" | nc localhost 8888
echo "cmd,Botname,Deity,co +dps -tank" | nc localhost 8888
```

## Future considerations

- The TCP server could be extended with authentication if exposed beyond Tailscale.
- Bot responses (the whisper replies bots send back) are not captured by this interface. OpenClaw can poll state via read queries instead.
- If upstream mod-playerbots changes `HandleRemoteCommand`, the patch may need regeneration. The change is localized to one function in one file.
