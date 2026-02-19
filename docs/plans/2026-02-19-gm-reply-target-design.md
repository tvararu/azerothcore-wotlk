# GM Reply Target — Design

## Problem

When a GM whispers a command (e.g. `los`) to a bot that has a different master, the security gate passes (GMs get `PLAYERBOT_SECURITY_ALLOW_ALL`) but every response goes to the master via `TellMaster()`, not back to the GM. The GM can control the bot but is blind to the output.

## Solution

Add a transient `replyTarget` pointer to `PlayerbotAI`. When a GM who is not the master issues a command, set the reply target before dispatch and clear it after. `TellMasterNoFacing` checks the reply target and duplicates the message to the GM.

This covers every command that uses `TellMaster` with zero changes to individual action classes.

## Delivery

The patch is stored as a git-format `.patch` file in `patches/mod-playerbots/` and applied automatically by `mise setup-modules`. This keeps the change reproducible across fresh clones and module updates.

## Changes

### PlayerbotAI.h

- Add `Player* replyTarget = nullptr;` in the `protected:` section near `Player* master;` (line 631)
- Add `GetReplyTarget()` / `SetReplyTarget()` accessors near `GetMaster()` (line 532)

### PlayerbotAI.cpp — two dispatch sites

1. **`HandleCommands()`** (lines 527-558) — set `replyTarget` before `ParseChatCommand()`, clear after
2. **`HandleCommand()`** (line 911+) — set `replyTarget` around immediate-execution paths (`DoSpecificAction`, etc.) that bypass the queue

### PlayerbotAI.cpp — TellMasterNoFacing (line 2824)

After `master->SendDirectMessage(&data)` on line 2854, build a duplicate packet and send to `replyTarget` if set, not the master, and still in world.

## Patch infrastructure

- `patches/mod-playerbots/gm-reply-target.patch` — git-format patch
- `.gitignore` negation: `!patches/**/*.patch`
- New `mise apply-patches` task applies all `patches/<module>/*.patch` to `modules/<module>/`
- `mise setup-modules` calls `mise apply-patches` at the end

## Invariants

- Master always receives the message (copy to GM, not redirect)
- Non-GM players still get rejected by the security check
- No changes to any action class
- `replyTarget` is cleared after each command dispatch, no state leakage
- `replyTarget->IsInWorld()` guard prevents crash if GM disconnects mid-command

## Pinned SHA workflow

mod-playerbots is pinned to a specific commit in `setup-modules`. The patch targets that commit. When bumping the pin, re-run `mise setup-modules`; if the patch fails to apply, regenerate it against the new SHA.
