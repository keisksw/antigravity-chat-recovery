# Antigravity Chat History Disappearance — Complete Recovery Guide

> **Verified on:** Antigravity 1.20.6 / macOS 15.7.4 Sequoia (Darwin arm64)
> **Author:** Documented from a real recovery session (2026-03-27)
> **Scope:** 26 missing chat sessions recovered with zero data loss using the ".pb injection" method
> **License:** Public domain. Share, adapt, or submit as a GitHub issue reference.
> **Credits:** Recovery technique adapted from [AG-Chat-Recovery](https://github.com/Artemis43/AG-Chat-Recovery) by Artemis43
> **Disclaimer:** This guide involves manual file manipulation. Always backup your data first. Use at your own risk.

---

## Table of Contents

1. [What Happened — The Bug Explained](#1-what-happened--the-bug-explained)
2. [Antigravity Data Architecture](#2-antigravity-data-architecture)
3. [Data Inventory — What Survives and What Is Lost](#3-data-inventory--what-survives-and-what-is-lost)
4. [Environment Setup](#4-environment-setup)
5. [Recovery Method: .pb Injection](#5-recovery-method-pb-injection)
   - [5.1 How It Works](#51-how-it-works)
   - [5.2 Prerequisites](#52-prerequisites)
   - [5.3 Step-by-Step Procedure (6 Phases)](#53-step-by-step-procedure-6-phases)
6. [What Happens After Recovery](#6-what-happens-after-recovery)
7. [Safety Protocols — What NOT to Do](#7-safety-protocols--what-not-to-do)
8. [Identifying Missing Conversations](#8-identifying-missing-conversations)
9. [Identifying Dummy Chats by Birth Time](#9-identifying-dummy-chats-by-birth-time)
10. [The v4 Accident — A Cautionary Tale](#10-the-v4-accident--a-cautionary-tale)
11. [What Cannot Be Recovered](#11-what-cannot-be-recovered)
12. [Methods That Do NOT Work](#12-methods-that-do-not-work)
13. [Prevention — How to Avoid Losing Chats](#13-prevention--how-to-avoid-losing-chats)
14. [Using This Guide With an AI Agent](#14-using-this-guide-with-an-ai-agent)
15. [Investigation Notes — Unresolved Issues](#15-investigation-notes--unresolved-issues)
16. [Frequently Asked Questions](#16-frequently-asked-questions)
17. [Appendix: Reference Commands](#17-appendix-reference-commands)

---

## 1. What Happened — The Bug Explained

### The Symptom

After updating Antigravity (known to occur in versions **1.18.x, 1.19.x, and 1.20.x**), after a crash, or seemingly at random — the chat history sidebar appears empty or shows far fewer conversations than expected. You can start new chats, but past conversations are invisible in the UI.

This is a **widely reported bug** acknowledged by Antigravity's engineering team. The community has tracked it since at least version 1.18.x. Multiple workarounds exist (see Section 12 for methods that do NOT work, and Section 5 for one that does).

### The Root Cause

Antigravity maintains two **independent** data stores:

1. **Conversation data** — individual `.pb` (protobuf) files, each containing an entire chat session
2. **Index** — a registration list that tells the UI which conversations to display

When the bug strikes, **the `.pb` files remain perfectly intact on disk**, but the index loses some or all of its registration entries. The UI only shows conversations that exist in the index — it does **not** scan the `conversations/` directory dynamically.

### Why the UI Doesn't "Just Find" the Files

There is no filesystem-scan mechanism on startup. Even if 100 `.pb` files are present, Antigravity will silently ignore any `.pb` that doesn't have a corresponding entry in the index. That's why the files survive but the conversations are invisible.

---

## 2. Antigravity Data Architecture

Understanding where Antigravity stores data is critical for any recovery attempt.

### File System Layout

```
~/.gemini/antigravity/
├── conversations/
│   ├── {UUID-1}.pb          # Full chat content (protobuf binary)
│   ├── {UUID-2}.pb
│   └── ...
├── brain/
│   ├── {UUID-1}/            # AI-generated artifacts, plans, walkthroughs
│   │   ├── task.md
│   │   ├── implementation_plan.md
│   │   └── ...
│   └── {UUID-2}/
├── annotations/
│   └── {UUID}.pbtxt         # Last-viewed timestamp only
└── state.vscdb              # Local state (NOT the main index)

~/Library/Application Support/Antigravity/User/globalStorage/
└── state.vscdb              # Main SQLite DB containing the index
```

### Key Index Entries (inside `state.vscdb`)

| Key | Format | Description |
|---|---|---|
| `antigravityUnifiedStateSync.trajectorySummaries` | Protobuf/base64 | **Primary UI index** — determines which chats appear in sidebar |
| `chat.ChatSessionStore.index` | JSON | Session-level index; often found empty (`{"version":1,"entries":{}}`) after the bug |

### What Each `.pb` File Contains

A `.pb` file is a self-contained binary protobuf holding the **complete** conversation:
- All user messages and AI responses
- Timestamps for each message
- Model metadata
- Tool call records
- Conversation settings

File sizes typically range from **50KB** (a single question-answer) to **10MB+** (long multi-hour sessions).

### The `brain/` Directory

Artifacts created during "task mode" (implementation plans, walkthroughs, task checklists) are stored in `~/.gemini/antigravity/brain/{UUID}/`. These are plain-text files. Even if the `.pb` is lost, the `brain/` directory may contain useful artifacts.

> **Important:** `brain/` files are conversation-specific and may be auto-deleted after 7 days (configurable). Copy important artifacts elsewhere.

### The `annotations/` Directory

Files in `~/.gemini/antigravity/annotations/{UUID}.pbtxt` contain only a `last_user_view_time` timestamp. Their presence means Antigravity has "seen" the conversation, but they do **not** control whether a chat appears in the sidebar.

---

## 3. Data Inventory — What Survives and What Is Lost

### Step 1: Establish your baseline

Before doing anything else, count what you have:

```bash
# Count all .pb files (total conversation data on disk)
ls ~/.gemini/antigravity/conversations/*.pb | wc -l

# List them with sizes, sorted largest first
ls -lhS ~/.gemini/antigravity/conversations/*.pb
```

Then open Antigravity and count the chats visible in the sidebar. The difference is your **recovery target**.

### What Survives (Recoverable) ✅

- All `.pb` files that have not been overwritten or deleted
- Full conversation content: messages, AI responses, timestamps, tool calls
- Files from any date — age doesn't matter
- `brain/` artifacts (task.md, implementation plans, etc.)

### What Is Lost After Recovery ❌

- The **original conversation UUID** — after recovery, the chat lives under a new UUID (the dummy's UUID)
- **Workspace association** — the recovered chat appears in whichever workspace the dummy was created in
- **Conversation title** — Antigravity regenerates it; may differ from the original
- Any `.pb` that was **physically overwritten** or deleted before backup
- Sessions lost to physical drive failure without any form of backup

### ⚠️ THE CRITICAL FIRST ACTION: BACKUP

```bash
# Do this BEFORE any recovery attempt. Non-negotiable.
cp -a ~/.gemini/antigravity/conversations/ ~/Desktop/conversations_backup_$(date +%Y%m%d)/
```

This single command saves every conversation to your Desktop. If anything goes wrong during recovery, you can always restore from this backup:

```bash
# Emergency rollback (restores ALL conversations to pre-recovery state)
cp -a ~/Desktop/conversations_backup_$(date +%Y%m%d)/* ~/.gemini/antigravity/conversations/
```

---

## 4. Environment Setup

### Dedicated Recovery Workspace

Create a dedicated workspace for this operation. This keeps recovered chats organized and prevents confusion with active work.

```bash
mkdir -p ~/Projects/chat-recovery
```

Then in Antigravity:
1. Open **Agent Manager** (or Workspaces panel)
2. Click **"+ Open Workspace"** or **"Add Workspace"**
3. Select `~/Projects/chat-recovery`

> **Why a dedicated workspace?**
> - Dummy chats created here stay isolated from your real workspaces
> - After recovery, all recovered conversations appear under this workspace — easy to find
> - You can delete this workspace after confirming all recoveries are successful

### Create Recovery Tracking Files

Inside `~/Projects/chat-recovery/`, create two files:

**`catalog.md`** — checklist of conversations to recover:

```markdown
## Recovery Checklist

- [ ] `{uuid-prefix}` ({size}) — {description from brain/ or memory}
- [ ] `{uuid-prefix}` ({size}) — {description}
...
```

**`recovery-plan.md`** — batch progress tracking table:

```markdown
## Progress

| Batch | Count | Dummies Created | UUIDs Identified | Mapping Approved | cp Executed | Verified |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1 | 5 | — | — | — | — | — |
| 2 | 5 | — | — | — | — | — |
...
```

---

## 5. Recovery Method: .pb Injection

### 5.1 How It Works

When Antigravity starts a new chat, it:
1. Generates a fresh UUID
2. Creates a new `.pb` file at `~/.gemini/antigravity/conversations/{UUID}.pb`
3. Registers that UUID in `trajectorySummaries` (the index)

The key insight: **the index only cares about the UUID, not the content of the `.pb` file.**

By creating a "dummy" chat (which gets registered in the index) and then overwriting its `.pb` with the content of a missing chat's `.pb`, the next Antigravity restart will display the missing conversation's full content — now accessible via the dummy's UUID.

```
[Missing chat .pb]  ──cp──►  [Dummy chat .pb]  ──restart──►  [Chat visible in UI!]
(has content,                 (in index,                       (index entry + real content
 not in index)                 empty content)                   = fully restored chat)
```

### 5.2 Prerequisites

Before starting, confirm all of these:

- [ ] **Backup completed** (see Section 3)
- [ ] **Dedicated workspace created** and added to Antigravity
- [ ] **Catalog prepared** — list of missing conversation UUIDs with sizes
- [ ] **Antigravity is running** — you need it open to create dummy chats
- [ ] **Terminal access** — to run `stat` and `cp` commands

### 5.3 Step-by-Step Procedure (6 Phases)

This process runs in **batches of 5** to keep operations manageable and human-reviewed.

---

#### PHASE 0: Prepare your catalog

List all `.pb` files with sizes and dates:

```bash
stat -f "%SB %z %N" -t "%Y-%m-%d %H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb | sort
```

Compare against UUIDs visible in Antigravity's sidebar. Those present on disk but absent from the UI are your recovery targets.

> **Tip:** Check `~/.gemini/antigravity/brain/{UUID}/` for each missing UUID. If artifacts exist there (task.md, walkthroughs), they'll help you identify what each conversation was about.

---

#### PHASE 1: Create dummy chats (Human action required)

In the **chat-recovery workspace**, open **5 new chats** and send this exact message to each:

```
This is a data recovery placeholder. Please output a single dot character only.
```

> **Why this exact message?** A minimal prompt keeps the `.pb` file small (≈130–145KB), creating a clear size contrast with real conversations. The AI responds with just `.`, maximizing the speed of creating 5 dummies back-to-back.

**Write down the start time and end time** (e.g., "14:32–14:33"). You'll use this in Phase 2.

> **Speed tip:** Open a new conversation → paste the prompt → press Enter → don't wait for the response → immediately open the next new conversation. You can create 5 dummies in under 30 seconds.

---

#### PHASE 2: Identify dummy UUIDs

Use **birth time** (macOS file creation time) to find the files created during your noted window:

```bash
stat -f "%SB %N" -t "%H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb | sort | tail -20
```

Look for a cluster of files born within your time window.

**⚠️ Verification checklist — ALL 3 must pass before proceeding:**

1. ✅ **Count matches** — exactly 5 files (or however many you created)
2. ✅ **Time window** — all birth times fall within your noted window ±2 minutes
3. ✅ **No collisions** — none of these UUIDs appear in your recovery target list

**If count does not match → STOP immediately and investigate.** Do not proceed until the discrepancy is resolved.

---

#### PHASE 3: Create mapping and review

Before executing any `cp`, create an explicit mapping table:

```
Batch 1 Mapping (5 items):
[1] Source: {missing-uuid-1}.pb (3.3MB, "Chat topic description")
    → Destination: {dummy-uuid-1}.pb (birth=14:32:15)
[2] Source: {missing-uuid-2}.pb (1.4MB, "Another chat topic")
    → Destination: {dummy-uuid-2}.pb (birth=14:32:28)
...
```

**Review this mapping yourself, or have your AI agent present it to you.** Verify that:
- Source UUIDs match your catalog
- Destination UUIDs are the dummies you just created
- No real conversations are in the "destination" column

**Do not proceed without explicitly confirming the mapping is correct.**

---

#### PHASE 4: Execute copy — one file at a time

Run each copy as a **separate, individual command**. Never chain them with `&&` or `;`.

```bash
# Copy 1
cp ~/.gemini/antigravity/conversations/{MISSING_UUID_1}.pb \
   ~/.gemini/antigravity/conversations/{DUMMY_UUID_1}.pb

# Copy 2 (run separately after confirming Copy 1 succeeded)
cp ~/.gemini/antigravity/conversations/{MISSING_UUID_2}.pb \
   ~/.gemini/antigravity/conversations/{DUMMY_UUID_2}.pb
```

After each `cp`, confirm success (exit code 0). If any fails, stop and investigate.

---

#### PHASE 5: Restart and verify

1. **Quit Antigravity completely** — use ⌘Q (macOS) or close from Dock. Simply closing the window is NOT enough.
2. **Relaunch Antigravity**
3. Open the **chat-recovery workspace**
4. Open each recovered conversation and confirm the content is correct
5. Update your `catalog.md` checklist: `- [ ]` → `- [x]`
6. Update your `recovery-plan.md` progress table

---

#### PHASE 6: Repeat per batch

Return to Phase 1 for the next batch of 5. Continue until all conversations are recovered.

---

## 6. What Happens After Recovery

### The Core Concept: "Recovery Slots"

The elegance of .pb injection lies in its simplicity: you create **empty "recovery slots"** (dummy chats) that Antigravity has registered in its index, then fill those slots with the actual conversation data. Antigravity doesn't verify that the content matches what was originally in that slot — it simply reads whatever `.pb` data is present.

Think of it like labeled filing cabinet drawers: Antigravity only opens drawers that are on its inventory list. If a drawer is on the list but you swap its contents, Antigravity happily reads the new contents when you open that drawer.

### Observed Behavior After Recovery

After restarting Antigravity with overwritten `.pb` files, the following behavior was observed in practice:

| Behavior | Expected | Actual (Observed) |
| --- | --- | --- |
| **Chat content** | Fully restored | ✅ **Fully restored** — all messages, responses, tool calls visible |
| **Conversation title** | Auto-updated from content | ⚠️ **Not updated** — title remains as the dummy's title (e.g., "Dummy chat data"). Updated for the very first recovery only; all subsequent recoveries kept the dummy title. Reason unknown |
| **Workspace association** | Auto-classified to original workspace | ❌ **Not auto-classified** — chat stays in whatever workspace the dummy was created in. This is why we create a dedicated recovery workspace |
| **Sidebar timestamp** | Updated to reflect content | ❌ **Not updated** — timestamp remains frozen at the dummy's creation time, even after sending new messages in the recovered chat |
| **Completion notification** | 🔵 unread indicator after agent finishes | ❌ **Not shown** — when continuing work in a recovered chat, the blue unread indicator does not appear when the agent completes a task. Reason unknown |
| **Continued use** | Normal operation | ⚠️ **Partially functional** — you can send additional messages and the agent responds normally, but the UI metadata issues above persist |
| **Agent's conversation summary** | Shows original topic | ⚠️ **Shows dummy title** — in the agent's "recent 10 conversations" context, recovered chats appear with their dummy titles, not the original content-based titles |

### What This Means in Practice

1. **Your data is safe** — message content is 100% restored and fully usable
2. **The UI metadata is broken** — titles, timestamps, notifications, and workspace assignment do not auto-heal (with the possible exception of the very first recovery in a session)
3. **This is why a dedicated workspace matters** — since recovered chats don't auto-classify back to their original workspaces, having them all in `chat-recovery` keeps them organized and findable
4. **If Antigravity patches the index bug in the future** — the original `.pb` files (still on disk under their original UUIDs) might suddenly reappear in the UI, potentially creating duplicates of recovered conversations

### What Is Preserved vs What Is Lost

| Preserved ✅ | Lost/Broken ❌ |
| --- | --- |
| All message content (user + AI) | Original conversation UUID |
| Message timestamps (original send/receive times) | Auto-generated title (stays as dummy title) |
| Tool call records | Workspace association (stays in recovery workspace) |
| Attached images/files within the `.pb` | Sidebar timestamp (frozen at dummy creation time) |
| `brain/` artifacts (separate from `.pb`) | Unread/notification indicators |
| Ability to continue the conversation | Agent-visible conversation summary title |

---

## 7. Safety Protocols — What NOT to Do

These rules were established after a **real incident** (the "v4 accident", Section 10) that caused permanent, unrecoverable data loss of 3 conversations.

| ❌ Prohibited | ✅ Safe Alternative | Why |
|---|---|---|
| Identify dummies by **file size** | Identify by **birth time** only | Short real chats have similar sizes to dummies |
| **Bulk copy** with loops or scripts | Human-reviewed mapping → individual `cp` | One wrong target destroys real data |
| Chain commands with `&&` or `;` | Run each `cp` as a separate command | Errors in chained commands cascade silently |
| Run automation **without backup** | Always backup first | One mistake without backup = permanent loss |
| Assume small file = dummy | Cross-check against known target UUIDs | Real conversations of 55KB exist |
| Use `ls -t` (modification time) | Use `stat -f "%SB"` (birth time) | `cp` changes modification time; birth time is immutable |
| Let AI agent auto-execute | Require explicit human approval for every batch | Prevents autonomous misidentification |

---

## 8. Identifying Missing Conversations

### Option A: Simple count comparison

```bash
# Files on disk
ls ~/.gemini/antigravity/conversations/*.pb | wc -l
# Then count chats visible in Antigravity's sidebar
# Difference = approximate number of missing conversations
```

### Option B: Read the SQLite index directly

```bash
# Extract registered UUIDs from the primary index
sqlite3 "~/Library/Application Support/Antigravity/User/globalStorage/state.vscdb" \
  "SELECT key, length(value) FROM ItemTable WHERE key LIKE '%chat%' OR key LIKE '%trajectory%';"
```

For a full UUID list from the session store:

```bash
sqlite3 ~/.gemini/antigravity/state.vscdb \
  "SELECT value FROM ItemTable WHERE key = 'chat.ChatSessionStore.index';" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for entry in data.get('entries', []):
    print(entry.get('sessionId',''))
"
```

Cross-reference against filenames in `~/.gemini/antigravity/conversations/`. Files on disk but NOT in the index are your recovery targets.

### Option C: Identify conversations from brain/ artifacts

If you don't remember what each missing UUID contained, check for artifacts:

```bash
# List all brain directories (one per conversation)
ls -la ~/.gemini/antigravity/brain/

# Check a specific conversation's artifacts
ls ~/.gemini/antigravity/brain/{UUID}/
cat ~/.gemini/antigravity/brain/{UUID}/task.md
```

### Option D: Community tools

An open-source tool [AG-Chat-Recovery](https://github.com/Artemis43/AG-Chat-Recovery) automates index comparison and offers a web UI for recovery. Verify you are using a trusted, well-reviewed repository before running any external script. **Always backup first regardless of which tool you use.**

---

## 9. Identifying Dummy Chats by Birth Time

This is the **core safety mechanism** of the recovery process. Getting this wrong causes data loss (see Section 10).

### What is birth time?

macOS (HFS+/APFS) records when a file was **first created**. This is called **birth time** (`Btime` or `SB` in stat). It has two critical properties:

1. **Immutable by `cp`** — when you `cp` over a file, the modification time updates but the birth time does NOT
2. **Precise to the second** — you can identify exactly when Antigravity created each `.pb`

### View birth times

```bash
# All .pb files, sorted by birth time (oldest first)
stat -f "%SB %N" -t "%H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb | sort
```

### Filter to a time window

```bash
# Files born between 22:30 and 22:35
stat -f "%SB %N" -t "%H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb \
  | grep "^22:3[0-5]" | sort
```

### What dummies look like in the output

Dummy chats created in rapid succession form a visible **cluster** — typically 5 files with birth times seconds apart, all ~130–145KB:

```
22:21:09 /path/to/8f2f6276-45e8-4985-85ad-6ece432e23ab.pb   ← dummy 1
22:21:20 /path/to/079762d6-fff7-4a25-8fc3-459e250d9fef.pb   ← dummy 2
22:21:36 /path/to/a2b73712-3727-48a7-81aa-162c00c1bea8.pb   ← dummy 3
22:21:40 /path/to/1754a146-7bf7-48de-bec4-6f9d4092b650.pb   ← dummy 4
22:21:52 /path/to/d82423f8-d225-42c8-b2cc-422b55855c14.pb   ← dummy 5
```

### Why NOT to use modification time

When you run `cp source.pb destination.pb`, the destination file's modification time is updated to "now". Its birth time remains unchanged (the original creation time of the dummy). Using `ls -t` (which sorts by modification time) would show recently-copied files, not recently-created ones — leading to misidentification.

---

## 10. The v4 Accident — A Cautionary Tale

During an earlier, automated attempt to recover conversations, an AI agent created a bulk restore script called "v4". This script identified dummy chats by **file size** alone (files under 200KB were assumed to be dummies). The script ran without human review.

### What Went Wrong

Three real conversations — which happened to be **short exchanges under 200KB** — were misidentified as dummies and overwritten with dummy data.

| UUID | Original Size | Content | Outcome |
|---|---|---|---|
| `{uuid-1}` | 55KB | A brief verification exchange | ❌ **Permanently overwritten** |
| `{uuid-2}` | 159KB | A short test session | ❌ **Permanently overwritten** |
| `{uuid-3}` | 157KB | A quick system check | ❌ **Permanently overwritten** |

No Time Machine backup and no APFS snapshot existed at the time. These three conversations are **gone forever**.

### Why This Happened

1. **File size ≠ dummy status.** Real conversations can be small. A single question + answer might be only 50KB.
2. **No human review checkpoint.** The script ran autonomously — no one verified the mapping before execution.
3. **No backup was taken first.** A simple `cp -a` beforehand would have saved everything.
4. **Overconfidence.** The earlier v1–v3 scripts had worked, creating false confidence that v4's optimization (using size instead of birth time) was safe.

### The Lessons

1. **Identify dummies by birth time, never by size.**
2. **Present a mapping table to a human before every batch of `cp` commands.**
3. **Never let any script or AI agent run `cp` on `.pb` files without explicit human approval.**
4. **Always take a full backup before starting. This rule has zero exceptions.**
5. **Slow down. Excitement and momentum cause careless mistakes.** This recovery process is boring by design.

---

## 11. What Cannot Be Recovered

Even with the .pb injection method, some things are permanently gone:

| What | Why | Workaround |
|---|---|---|
| Overwritten `.pb` files | Binary data is replaced; no undo without snapshots | Check `tmutil listlocalsnapshots /` for APFS snapshots |
| Original conversation UUID | UUID is hardcoded in the index entry | Cosmetic only; chat content is unaffected |
| Workspace association | Controlled by index, not `.pb` content | Minor; chat appears in recovery workspace instead |
| Auto-generated title | Antigravity regenerates based on content | Usually close enough; sometimes different wording |

### If You Have Time Machine or APFS Snapshots

Before concluding that data is lost, check:

```bash
# List available local APFS snapshots
tmutil listlocalsnapshots /

# Browse a snapshot (if available)
# Replace {date} with an available snapshot date
mount_apfs -s "com.apple.TimeMachine.{date}" /dev/diskXsY /tmp/snapshot_mount
ls /tmp/snapshot_mount/.gemini/antigravity/conversations/
```

If an older snapshot exists from before the data loss, you can copy `.pb` files directly from it.

---

## 12. Methods That Do NOT Work

The following approaches were tested during this recovery session and confirmed to be **ineffective** for this specific bug:

| Method | Why It Fails |
|---|---|
| **Reopening the workspace** (File → Open Folder) | The index is corrupt/empty; simply opening a folder doesn't rebuild it |
| **Agent Manager "tapping" method** | Clicking through Agent Manager sessions doesn't trigger re-indexing |
| **Editing `chat.ChatSessionStore.index` directly** | Antigravity **overwrites** this on restart; manual edits are discarded |
| **Editing `trajectorySummaries` in `state.vscdb`** | Same problem — overwritten on restart |
| **Downgrading Antigravity** | Some users report success ([source: Reddit](https://reddit.com)), but this was not verified in our case and may cause other issues |
| **Deleting `state.vscdb`** | Antigravity creates a fresh empty one — doesn't help recover missing entries |

> **Why can't we just edit the database?** Antigravity maintains its own in-memory copy of the index and writes it to disk on shutdown. Any manual database edits are overwritten when Antigravity restarts with its (still-broken) in-memory state.

---

## 13. Prevention — How to Avoid Losing Chats

While the underlying bug is in Antigravity's code and will hopefully be patched, these practices minimize damage when it inevitably recurs:

### 1. Keep conversations short

Conversations approaching **15MB** (verified threshold from community reports) may trigger indexing issues more frequently. Split long sessions into new conversations.

### 2. Save important artifacts outside Antigravity

Don't rely on chat history as your only record. Regularly export:
- Key decisions → project `docs/` directory
- Research findings → knowledge base files
- Implementation plans → version-controlled repositories

### 3. Watch for early warning signs

If a `.pb` file's modification timestamp stops updating for **30+ minutes** despite active conversation, this may indicate a save/sync issue. Consider starting a new conversation.

### 4. Periodic backups

Set up a simple periodic backup:

```bash
# Add to crontab: daily backup at 3am
0 3 * * * cp -a ~/.gemini/antigravity/conversations/ ~/Backups/antigravity_conversations_$(date +\%Y\%m\%d)/
```

### 5. Enable Time Machine

If you're on macOS and not using Time Machine, **enable it now**. It would have saved the 3 conversations lost in the v4 accident.

---

## 14. Using This Guide With an AI Agent

If you're using Antigravity's built-in AI agent (or any AI assistant) to help with recovery, share this document at the beginning of the session.

### Suggested prompt to start a recovery session

```
Please read ~/Projects/chat-recovery/antigravity-chat-recovery-guide.md fully.
Then read catalog.md and recovery-plan.md.
We are performing .pb injection recovery for Antigravity chat history.
Follow the safety protocols in Section 7 strictly.
Do not execute any cp command without my explicit approval.
I will create dummy chats and tell you the time window.
You identify the UUIDs, present the mapping, and wait for my "go".
Update the progress table after each step, not in bulk.
```

### What the agent should do

- ✅ Read this entire guide before taking any action
- ✅ Use `stat -f "%SB"` for birth time identification (macOS-specific)
- ✅ Present a mapping table for human review before each batch
- ✅ Execute `cp` commands **one at a time**, not in loops or chains
- ✅ Update the tracking table (recovery-plan.md) **after each step**, not all at once
- ✅ Halt immediately if dummy count doesn't match expected count

### What the agent must NOT do

- ❌ Select dummy UUIDs based on file size
- ❌ Execute multiple `cp` commands without human approval for each batch
- ❌ Run any bulk script on the conversations directory
- ❌ Assume any conversation is expendable
- ❌ Skip the backup step
- ❌ Get excited or rush — methodical pace prevents errors

---

## 15. Investigation Notes — Unresolved Issues

> This section documents unresolved behaviors observed after recovery. These are not blockers — **chat content recovery works fully** — but they represent incomplete aspects of the recovery that may be solvable with community input. If you have insights or solutions for any of these, contributions are welcome.

### Background: What Information Does the AI Agent Receive?

To understand why these issues matter, it helps to know what context Antigravity provides to its AI agent at each conversation turn:

| Context Provided | Description |
| --- | --- |
| **Recent conversation summaries** | Titles and brief descriptions of the most recent ~10 chat sessions, automatically generated from conversation content |
| **Active document state** | Which file the user has open, cursor position |
| **Workspace information** | Which workspace is currently active |
| **Conversation history** | Full message history of the *current* chat |

The "recent conversation summaries" are important: they allow the agent to understand what the user has been working on across sessions. They are generated and updated by Antigravity based on the `.pb` content.

### Issue: Agent-Visible Summaries Not Updated After Recovery

After recovering 26 conversations via .pb injection, we verified the agent's "recent conversation summaries" context. **None of the recovered conversations appeared in this list**, despite being accessible and functional in the UI.

This means:

- The agent cannot "see" or reference any recovered conversation in its cross-session context
- If the agent is asked "what did we discuss recently?", recovered sessions are invisible to it
- Summaries remain frozen as whatever was generated for the dummy chat (e.g., "Dummy chat data")

**Likely cause:** Antigravity generates conversation summaries at chat creation time and updates them based on specific triggers (possibly when the user opens a chat, or at set intervals). After .pb injection, the content changes but the summary-generation pipeline does not re-run.

**Unverified hypothesis:** Opening each recovered chat manually might trigger a summary regeneration. This was not tested during our recovery session.

### Issue: UI Metadata Frozen After Recovery

Several pieces of UI metadata remain frozen at their dummy-creation values and do not update, even after sending new messages in the recovered conversation:

| Metadata | Expected Behavior | Observed After Recovery |
| --- | --- | --- |
| **Sidebar title** | Updates to reflect conversation content | Remains as dummy title (first recovery was an exception — title updated once, then all subsequent recoveries kept dummy titles; reason unknown) |
| **Sidebar timestamp** | Updates on new activity | Frozen at dummy creation time |
| **Unread indicator** (🔵) | Appears when agent completes a task | Does not appear |
| **Workspace classification** | May re-classify based on content | Does not re-classify |

Recovered conversations remain **fully functional** for continued use (you can send messages, the agent responds normally), but the sidebar treats them as if they are still the original dummy chats.

### Why This Wasn't Addressed in This Recovery

The primary goal of this recovery was to **restore access to chat content** — the messages, AI responses, tool call records, and conversation history that were invisible due to the index bug. That goal was achieved: all 26 conversations are accessible and their content is intact.

The metadata issues (titles, timestamps, notifications, agent summaries) are UI-level inconveniences that do not affect the recovered data itself. Solving them likely requires understanding how Antigravity's internal summary pipeline triggers — which may involve reverse-engineering the application's update logic or waiting for an official fix.

### Call for Community Input

If you have found a way to force Antigravity to re-generate conversation summaries or update sidebar metadata after .pb injection, please share your findings. Specifically:

- Is there a way to trigger `trajectorySummaries` re-computation for a specific UUID?
- Does opening each recovered chat in sequence cause summary regeneration?
- Are there undocumented API endpoints or developer tools within Antigravity that can force a metadata refresh?

Any solution here would benefit everyone using the .pb injection recovery method.

---

## 16. Frequently Asked Questions

**Q: Will the recovered chats look exactly like the originals?**
A: The content (messages, AI responses, tool calls, timestamps) is **100% preserved**. What changes: the UUID in the sidebar, the workspace label, and the auto-generated title (which is usually similar but may differ in wording).

**Q: Can I recover chats that are weeks or months old?**
A: Yes. As long as the `.pb` file exists on disk and hasn't been overwritten, age doesn't matter. We recovered conversations from several days prior without issue.

**Q: How do I know if my missing chats are actually recoverable?**
A: Run `ls ~/.gemini/antigravity/conversations/` and look for the UUIDs. If the `.pb` file exists and its size is greater than ~1KB, the content is almost certainly intact. You can also check the file size — a real conversation is typically 50KB to 10MB+.

**Q: What if Antigravity still doesn't show the chat after restart?**
A: Verify the `cp` succeeded by checking the destination file size: `ls -lh ~/.gemini/antigravity/conversations/{DUMMY_UUID}.pb`. If the size still shows ≈130KB (original dummy size), the copy didn't work. Check for typos in the UUID. Also ensure you **quit** Antigravity (⌘Q), not just closed the window.

**Q: Can I use this method on Windows?**
A: Yes, the method works identically. The directory locations and commands differ:

```powershell
# Windows: Conversations directory
~\.gemini\antigravity\conversations\

# List files with creation times (PowerShell)
Get-ChildItem "$env:USERPROFILE\.gemini\antigravity\conversations\*.pb" |
  Select-Object Name, CreationTime, Length | Sort-Object CreationTime

# Copy command
Copy-Item "$env:USERPROFILE\.gemini\antigravity\conversations\{MISSING}.pb" `
  "$env:USERPROFILE\.gemini\antigravity\conversations\{DUMMY}.pb"
```

**Q: Can I use this on Linux?**
A: The `.pb` injection method works on any OS. For birth time on Linux (ext4 with kernel 4.11+):

```bash
stat --format='%W %n' ~/.gemini/antigravity/conversations/*.pb | sort
```

Note: Not all Linux filesystems record birth time. If unavailable, use modification time with extreme caution and smaller batch sizes.

**Q: Is there an automated tool?**
A: [AG-Chat-Recovery](https://github.com/Artemis43/AG-Chat-Recovery) offers automated recovery via a web UI. However, **any automated tool carries risk** — review its source code and always backup first. This guide's manual approach with human review at every step is slower but safer.

**Q: Why doesn't Antigravity just re-scan the conversations folder?**
A: Unknown. A filesystem-scan-on-startup would be the simplest fix. The issue has been reported to Antigravity's engineering team. Until a patch is released, manual recovery is the only reliable option.

**Q: How many conversations can I recover in one session?**
A: We recovered 26 conversations in 5 batches (5-6 per batch) in approximately 45 minutes. The bottleneck is creating dummy chats and restarting Antigravity, not the `cp` commands.

**Q: Will recovered conversations count against any storage limits?**
A: No. The `.pb` files already existed on disk. The `cp` command creates an additional copy (the dummy's `.pb` now has the recovered content), but the original `.pb` also remains. You can delete the original orphaned `.pb` after confirming recovery, or keep it as an extra backup.

---

## 17. Appendix: Reference Commands

### System information

```bash
# macOS version
sw_vers

# Antigravity version
defaults read /Applications/Antigravity.app/Contents/Info.plist CFBundleShortVersionString
```

### Conversation directory

```bash
# macOS / Linux
~/.gemini/antigravity/conversations/

# Windows
%USERPROFILE%\.gemini\antigravity\conversations\
```

### Count .pb files on disk

```bash
ls ~/.gemini/antigravity/conversations/*.pb | wc -l
```

### Full backup (do this first!)

```bash
cp -a ~/.gemini/antigravity/conversations/ ~/Desktop/conversations_backup_$(date +%Y%m%d)/
```

### List .pb files with birth times (macOS)

```bash
stat -f "%SB %z %N" -t "%Y-%m-%d %H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb | sort
```

### Filter by time window

```bash
# Replace HH:MM with your target window
stat -f "%SB %N" -t "%H:%M:%S" \
  ~/.gemini/antigravity/conversations/*.pb | grep "^HH:M" | sort
```

### Single recovery copy

```bash
cp ~/.gemini/antigravity/conversations/{MISSING_UUID}.pb \
   ~/.gemini/antigravity/conversations/{DUMMY_UUID}.pb
```

### Read the SQLite index keys

```bash
sqlite3 ~/.gemini/antigravity/state.vscdb \
  "SELECT key, length(value) FROM ItemTable WHERE key LIKE '%chat%';"
```

### Check for APFS snapshots

```bash
tmutil listlocalsnapshots /
```

### Emergency rollback

```bash
cp -a ~/Desktop/conversations_backup_*/* ~/.gemini/antigravity/conversations/
```

---

*This guide was written from a real recovery session (2026-03-27). Environment: Antigravity 1.20.6, macOS 15.7.4 Sequoia, Apple Silicon (iMac). 26 chat sessions recovered via .pb injection with zero additional data loss after the v4 accident (3 conversations permanently lost due to automated misidentification). Recovery technique adapted from [AG-Chat-Recovery](https://github.com/Artemis43/AG-Chat-Recovery).*
