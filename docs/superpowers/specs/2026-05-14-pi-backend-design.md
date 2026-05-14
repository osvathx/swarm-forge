# Pi Backend Support

## Goal

Add `pi` ([pi.dev](https://pi.dev/docs/latest/usage)) as a per-role backend in `swarmforge.conf`, alongside the existing `claude`, `codex`, and `none` backends. Pi is a coding-agent CLI similar in shape to codex: a single binary that accepts a positional prompt argument and runs an interactive REPL.

## Scope

In scope:

- Recognize `pi` as a valid value for the agent field in `swarmforge.conf`.
- Check that the `pi` binary is on `PATH` at startup when any role uses it.
- Launch pi inside each pi-backed tmux window with the same bootstrap prompt file used for the other backends.
- Update `README.md` to list `pi` as a supported backend.

Out of scope:

- A configurable per-window extra-args field in `swarmforge.conf`.
- Pi-specific environment-variable plumbing (e.g., `PI_CODING_AGENT_DIR`, `PI_OFFLINE`). Users set these in their own shell environment.
- An auto-accept / unattended-mode flag. Pi has none documented; the operator handles tool-use approvals in the Terminal window if pi prompts.
- A new example swarm directory under `examples/`. Users swap `codex` → `pi` in their own `swarmforge.conf`.

## Touchpoints

All changes live in `swarmforge.sh`. Three small edits, no new files:

1. **Validation allowlist** — `parse_config`, the `case "$agent" in` block (currently around line 183):
   ```sh
   case "$agent" in
     claude|codex|pi|none) ;;
     *)
       echo -e "${RED}Error:${RESET} Unsupported agent '$agent' for role '$role'"
       exit 1
       ;;
   esac
   ```

2. **Dependency check** — `check_backend_dependencies` (currently around line 338). Add one case branch:
   ```sh
   pi) check_dependency pi ;;
   ```

3. **Launch command** — `launch_role`, the `case "$agent" in` block (currently around line 389). Add the pi branch, mirroring codex's pattern (positional prompt + working-directory `cd` prefix):
   ```sh
   pi)
     launch_cmd="export PATH='$SWARM_TOOLS_DIR:$SCRIPT_DIR':\$PATH && cd '$role_worktree' && pi \"\$(cat '$prompt_file')\""
     ;;
   ```

The cleanup-window wrapper (the `if [[ "$index" -eq "${CLEANUP_OWNER_INDEX}" ]]` block immediately after the case statement) appends `swarm-cleanup.sh` to whatever `launch_cmd` was assembled, so the pi branch inherits cleanup behavior with no extra work.

## Behavior

- **Bootstrap prompt.** Pi receives the file written by `write_agent_instruction_file` as its first user message (via `"$(cat '$prompt_file')"`). The file contents are backend-agnostic — they instruct the agent to read `swarmforge/constitution.prompt`, then `swarmforge/<role>.prompt`, and use `notify-agent.sh` for handoffs. No pi-specific phrasing required.
- **Working directory.** Pi has no `-C <dir>` flag; the launcher uses the existing `cd '$role_worktree' &&` prefix (same approach used for claude). Pi resolves its own session and config relative to the cwd at invocation.
- **System prompt.** Pi's `--append-system-prompt <text>` is not used. The bootstrap is sent as a user message only, matching the codex pattern. (Claude is the outlier — it sends the bootstrap both as `--append-system-prompt-file` and as a positional message — but that duplication serves no purpose for our one-shot bootstrap.)
- **Auto-accept.** Pi launches with its built-in defaults. If pi prompts the operator for tool-use approval, the operator handles it in the Terminal window. This is the same posture codex gets today.

## Documentation

- `README.md`: update line 30 ("Supports per-role backends such as `claude`, `codex`, or `none`") to list `pi` alongside the others.
- No other doc changes required; the `swarmforge.conf` syntax description already treats the agent field as an extensible token.

## Verification

Verification is **manual**. No automated tests are added to the repository — the user runs these checks by hand against a real working directory after implementation:

1. Run `swarmforge.sh` against a working directory whose `swarmforge.conf` includes at least one role with `pi` as the agent. Confirm the Terminal window opens, pi starts, and the agent receives and acts on the bootstrap prompt (i.e., it reads the constitution and role prompt).
2. Run `swarmforge.sh` against a `swarmforge.conf` containing an intentionally misspelled agent name (e.g., `pii`). Confirm the validation error still fires — regression check that adding `pi` to the case statement did not accidentally widen the allowlist.
3. Run `swarmforge.sh` against a `swarmforge.conf` with `pi` listed but no `pi` binary on `PATH`. Confirm `check_dependency` exits with the expected "'pi' is required but not installed" error before any tmux session is created.
