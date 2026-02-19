# GM Reply Target Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Let GMs see bot command responses when they whisper commands to bots owned by other players.

**Architecture:** Transient `replyTarget` pointer on `PlayerbotAI`, set/cleared around command dispatch. `TellMasterNoFacing` duplicates messages to the reply target. Delivered as a git-format `.patch` applied automatically by `mise setup-modules`.

**Tech Stack:** C++, git format-patch, mise tasks (bash)

---

### Task 1: Patch infrastructure — gitignore and directories

**Files:**
- Modify: `.gitignore:49`
- Create: `patches/mod-playerbots/.gitkeep`

**Step 1: Add gitignore negation**

In `.gitignore`, after line 50 (`*.diff`), add:

```
!patches/**/*.patch
```

This overrides the `*.patch` ignore rule for the `patches/` directory.

**Step 2: Create patches directory**

```bash
mkdir -p patches/mod-playerbots
touch patches/mod-playerbots/.gitkeep
```

**Step 3: Commit**

```bash
git add .gitignore patches/
git commit -m "chore: add patches/ directory for reproducible module patches"
```

---

### Task 2: Create mise apply-patches task

**Files:**
- Create: `.mise/tasks/apply-patches`

**Step 1: Write the task**

```bash
#!/usr/bin/env bash
#MISE description="Apply local patches to cloned modules"
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
PATCHES_DIR="$REPO_ROOT/patches"

if [[ ! -d "$PATCHES_DIR" ]]; then
  echo "No patches/ directory found, nothing to do."
  exit 0
fi

applied=0
failed=0

for module_dir in "$PATCHES_DIR"/*/; do
  module="$(basename "$module_dir")"
  target="$REPO_ROOT/modules/$module"

  if [[ ! -d "$target/.git" ]]; then
    echo "[$module] Module not cloned at modules/$module, skipping patches"
    continue
  fi

  for patch in "$module_dir"*.patch; do
    [[ -f "$patch" ]] || continue
    name="$(basename "$patch")"

    if git -C "$target" apply --check "$patch" 2>/dev/null; then
      echo "[$module] Applying $name"
      git -C "$target" apply "$patch"
      ((applied++))
    elif git -C "$target" apply --check --reverse "$patch" 2>/dev/null; then
      echo "[$module] $name already applied, skipping"
    else
      echo "[$module] FAILED to apply $name — patch may need regeneration"
      ((failed++))
    fi
  done
done

echo ""
echo "Patches applied: $applied, already applied: skipped, failed: $failed"

if [[ $failed -gt 0 ]]; then
  echo "Some patches failed. Regenerate them against the current pinned SHA."
  exit 1
fi
```

**Step 2: Make it executable**

```bash
chmod +x .mise/tasks/apply-patches
```

**Step 3: Commit**

```bash
git add .mise/tasks/apply-patches
git commit -m "feat: add mise apply-patches task for reproducible module patches"
```

---

### Task 3: Integrate apply-patches into setup-modules

**Files:**
- Modify: `.mise/tasks/setup-modules`

**Step 1: Add apply-patches call at the end**

After the existing final `echo` line ("All modules ready..."), add:

```bash
echo ""
echo "Applying local patches..."
mise apply-patches
```

**Step 2: Commit**

```bash
git add .mise/tasks/setup-modules
git commit -m "chore: run apply-patches automatically after setup-modules"
```

---

### Task 4: Write the C++ patch — PlayerbotAI.h

**Files:**
- Modify: `modules/mod-playerbots/src/Bot/PlayerbotAI.h:532` (accessors)
- Modify: `modules/mod-playerbots/src/Bot/PlayerbotAI.h:631` (field)

All edits in this task and the next are temporary — they exist only to generate the patch file.

**Step 1: Add accessors after GetMaster() (line 532)**

After line 533 (`Player* FindNewMaster();`), insert:

```cpp
    Player* GetReplyTarget() { return replyTarget; }
    void SetReplyTarget(Player* target) { replyTarget = target; }
```

**Step 2: Add field after Player* master (line 631)**

After line 631 (`Player* master;`), insert:

```cpp
    Player* replyTarget = nullptr;
```

---

### Task 5: Write the C++ patch — PlayerbotAI.cpp HandleCommands

**Files:**
- Modify: `modules/mod-playerbots/src/Bot/PlayerbotAI.cpp:536` (HandleCommands)

**Step 1: Set replyTarget before dispatch in HandleCommands()**

After line 541 (closing brace of the `if (!owner)` block), before the `const std::string& command` line, insert:

```cpp

        if (owner && owner != master &&
            owner->GetSession()->GetSecurity() >= SEC_GAMEMASTER)
        {
            SetReplyTarget(owner);
        }
```

**Step 2: Clear replyTarget after dispatch**

After line 555 (closing brace of the `if (!helper.ParseChatCommand...)` block), before the `it = chatCommands.erase(it);` line, insert:

```cpp

        SetReplyTarget(nullptr);
```

---

### Task 6: Write the C++ patch — PlayerbotAI.cpp HandleCommand

**Files:**
- Modify: `modules/mod-playerbots/src/Bot/PlayerbotAI.cpp:987` (HandleCommand immediate paths)

**Step 1: Set replyTarget before the immediate-execution if/else chain**

After line 985 (closing brace + `return;` of the raid warning block), before the `if ((filtered.size() > 2 ...` line, insert:

```cpp

    if (fromPlayer && fromPlayer != master &&
        fromPlayer->GetSession()->GetSecurity() >= SEC_GAMEMASTER)
    {
        SetReplyTarget(fromPlayer);
    }
```

**Step 2: Clear replyTarget at end of HandleCommand**

Before line 1047 (the closing `}` of HandleCommand), insert:

```cpp

    SetReplyTarget(nullptr);
```

---

### Task 7: Write the C++ patch — PlayerbotAI.cpp TellMasterNoFacing

**Files:**
- Modify: `modules/mod-playerbots/src/Bot/PlayerbotAI.cpp:2854`

**Step 1: Duplicate message to replyTarget**

After line 2854 (`master->SendDirectMessage(&data);`), insert:

```cpp

        if (replyTarget && replyTarget != master && replyTarget->IsInWorld())
        {
            WorldPacket replyData;
            ChatHandler::BuildChatPacket(replyData,
                type == CHAT_MSG_ADDON ? CHAT_MSG_PARTY : type,
                type == CHAT_MSG_ADDON ? LANG_ADDON : LANG_UNIVERSAL,
                bot, nullptr, text.c_str());
            replyTarget->SendDirectMessage(&replyData);
        }
```

---

### Task 8: Generate the patch file

**Step 1: Generate patch from module working tree**

```bash
cd modules/mod-playerbots
git diff > ../../patches/mod-playerbots/gm-reply-target.patch
```

**Step 2: Verify the patch is non-empty and well-formed**

```bash
wc -l patches/mod-playerbots/gm-reply-target.patch
head -20 patches/mod-playerbots/gm-reply-target.patch
```

Expected: ~80-100 lines, starts with `diff --git`.

**Step 3: Revert the module working tree**

```bash
cd modules/mod-playerbots
git checkout -- .
```

**Step 4: Test round-trip — apply the patch**

```bash
mise apply-patches
```

Expected output:
```
[mod-playerbots] Applying gm-reply-target.patch
Patches applied: 1, already applied: skipped, failed: 0
```

**Step 5: Test idempotence — run again**

```bash
mise apply-patches
```

Expected output:
```
[mod-playerbots] gm-reply-target.patch already applied, skipping
```

**Step 6: Revert module for clean state, then remove .gitkeep**

```bash
cd modules/mod-playerbots
git checkout -- .
rm patches/mod-playerbots/.gitkeep
```

**Step 7: Commit the patch file**

```bash
git add patches/mod-playerbots/gm-reply-target.patch
git rm --cached patches/mod-playerbots/.gitkeep 2>/dev/null || true
git commit -m "feat: add GM reply target patch for mod-playerbots

When a GM whispers a command to a bot with a different master, the bot
now sends a copy of its response back to the GM in addition to the
master. Applied automatically by mise setup-modules."
```

---

### Task 9: Final apply and verification

**Step 1: Run setup-modules to test the full flow**

```bash
mise setup-modules
```

Expected: modules clone/update, then patches apply.

**Step 2: Verify the patched code is in place**

```bash
grep -n "replyTarget" modules/mod-playerbots/src/Bot/PlayerbotAI.h
grep -n "replyTarget" modules/mod-playerbots/src/Bot/PlayerbotAI.cpp
```

Expected: accessors + field in .h, set/clear + duplicate send in .cpp.

**Step 3: Delete the old design sketch**

The original `docs/gm-reply-target-patch.md` is superseded by the design doc and this plan.

```bash
rm docs/gm-reply-target-patch.md
git add docs/gm-reply-target-patch.md
git commit -m "chore: remove superseded patch sketch"
```
