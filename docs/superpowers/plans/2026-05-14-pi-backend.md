# Pi Backend Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `pi` as a per-role backend in `swarmforge.conf`, alongside `claude`, `codex`, and `none`.

**Architecture:** Three small edits to `swarmforge.sh` (validation allowlist, dependency check, launch-command case branch) plus two doc lines in `README.md`. Pi launches just like codex: positional prompt argument, `cd` into the worktree first, no extra flags. Verification is performed manually by the operator — no automated tests added.

**Tech Stack:** zsh, tmux, the `pi` CLI ([pi.dev](https://pi.dev/docs/latest/usage)).

**Spec:** `docs/superpowers/specs/2026-05-14-pi-backend-design.md`

---

### Task 1: Wire pi backend into swarmforge.sh

**Files:**
- Modify: `swarmforge.sh` (three small edits in three different functions)

Three edits land in one commit because they form one logical unit — none of them works without the others (the allowlist rejects `pi` without edit 1, startup errors before dependency-check without edit 2, and launch silently no-ops without edit 3).

- [ ] **Step 1: Add `pi` to the agent-validation allowlist in `parse_config`**

In `swarmforge.sh`, find this block (inside `parse_config`, around line 183):

```sh
    case "$agent" in
      claude|codex|none) ;;
      *)
        echo -e "${RED}Error:${RESET} Unsupported agent '$agent' for role '$role'"
        exit 1
        ;;
    esac
```

Replace it with:

```sh
    case "$agent" in
      claude|codex|pi|none) ;;
      *)
        echo -e "${RED}Error:${RESET} Unsupported agent '$agent' for role '$role'"
        exit 1
        ;;
    esac
```

- [ ] **Step 2: Add `pi` to `check_backend_dependencies`**

In `swarmforge.sh`, find this block (around lines 338-346):

```sh
check_backend_dependencies() {
  local i
  for (( i = 1; i <= ${#AGENTS[@]}; i++ )); do
    case "${AGENTS[$i]}" in
      claude) check_dependency claude ;;
      codex) check_dependency codex ;;
    esac
  done
}
```

Replace it with:

```sh
check_backend_dependencies() {
  local i
  for (( i = 1; i <= ${#AGENTS[@]}; i++ )); do
    case "${AGENTS[$i]}" in
      claude) check_dependency claude ;;
      codex) check_dependency codex ;;
      pi) check_dependency pi ;;
    esac
  done
}
```

- [ ] **Step 3: Add the `pi` launch branch in `launch_role`**

In `swarmforge.sh`, find this block (inside `launch_role`, around lines 389-396):

```sh
  case "$agent" in
    claude)
      launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && claude --append-system-prompt-file '$prompt_file' --permission-mode acceptEdits -n 'SwarmForge ${display}' \"\$(cat '$prompt_file')\""
      ;;
    codex)
      launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && codex -C '$role_worktree' \"\$(cat '$prompt_file')\""
      ;;
  esac
```

Replace it with:

```sh
  case "$agent" in
    claude)
      launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && claude --append-system-prompt-file '$prompt_file' --permission-mode acceptEdits -n 'SwarmForge ${display}' \"\$(cat '$prompt_file')\""
      ;;
    codex)
      launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && codex -C '$role_worktree' \"\$(cat '$prompt_file')\""
      ;;
    pi)
      launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && pi \"\$(cat '$prompt_file')\""
      ;;
  esac
```

Note: no `--append-system-prompt` flag and no `-C` flag are used. Pi has no working-directory flag; the leading `cd '$role_worktree' &&` handles that. The bootstrap file is sent only as the positional user message, mirroring the codex pattern (see spec §Behavior).

- [ ] **Step 4: Sanity-check the script still parses**

Run: `zsh -n swarmforge.sh`
Expected: exit code 0, no output. (`-n` performs a syntax-only check without executing.)

If this fails, re-read the three edited blocks and look for misplaced quotes, missing `;;`, or accidentally truncated lines.

- [ ] **Step 5: Commit**

```bash
git add swarmforge.sh
git commit -m "Add pi as a per-role agent backend"
```

---

### Task 2: Update README to list pi as a supported backend

**Files:**
- Modify: `README.md` (two lines mentioning the backend list)

The README mentions the backend set twice — both need updating to keep them in sync. One commit covers both edits.

- [ ] **Step 1: Update the Core Features bullet**

In `README.md`, find this line (around line 19):

```markdown
- Supports per-role backends such as `claude`, `codex`, or `none`
```

Replace it with:

```markdown
- Supports per-role backends such as `claude`, `codex`, `pi`, or `none`
```

- [ ] **Step 2: Update the Backend Selection Per Role bullet**

In `README.md`, find this line (around line 30):

```markdown
- **Backend Selection Per Role** — A role can launch `claude`, `codex`, or no agent at all.
```

Replace it with:

```markdown
- **Backend Selection Per Role** — A role can launch `claude`, `codex`, `pi`, or no agent at all.
```

- [ ] **Step 3: Confirm both mentions now match**

Run: `grep -n 'backends\|Backend Selection' README.md`
Expected: both matching lines now contain `pi`. If either is missing pi, redo the step that missed it.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "List pi as a supported backend in README"
```

---

### Task 3: Hand off for manual verification

**Files:** none.

Per the spec (§Verification), no automated tests are added. The operator runs these checks by hand against a real working directory after the implementation tasks land. This task is the handoff — list the three checks for the operator, then stop.

- [ ] **Step 1: Report the manual verification checklist to the user**

Tell the user implementation is complete and ask them to run these three checks themselves:

1. **Happy path.** In a working directory whose `swarmforge/swarmforge.conf` includes at least one window line with `pi` as the agent (e.g., `window coder pi coder`), run `swarmforge.sh`. Confirm the Terminal window opens, pi starts, and the agent acts on the bootstrap prompt (reads `swarmforge/constitution.prompt` and `swarmforge/<role>.prompt`).

2. **Allowlist regression.** Edit a `swarmforge/swarmforge.conf` to use an intentionally misspelled agent name (e.g., `window coder pii coder`). Run `swarmforge.sh`. Confirm it exits with the validation error `Error: Unsupported agent 'pii' for role 'coder'`.

3. **Missing-binary check.** With `swarmforge.conf` listing `pi` for at least one window, run `swarmforge.sh` in a shell whose `PATH` does **not** include the `pi` binary (e.g., `PATH=/usr/bin:/bin swarmforge.sh`). Confirm it exits with `Error: 'pi' is required but not installed` before any tmux session is created.

Do not run these checks yourself — wait for the user to confirm or report failures.
