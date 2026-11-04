# agent-quit

Skill `/q` for Claude Code (and other agents): do the last task, then close the terminal tab (Ghostty).

`/q commit and push` or `commit and push /q` → the agent does it without questions, shows a short status notification in Ghostty, and the tab disappears.

- `/q` is lowercase only, at the very start or very end of the prompt.
- `skill/q/SKILL.md` — the skill (handles the leading `/q`).
- `skill/q/close-tab` — finds the shell that owns the tab and sends it SIGHUP; also the hook handlers. Agent-agnostic.
- Two global hooks make it work for a trailing `/q` too (a skill is not loaded in that case).

## Install (instructions for an agent)

Run from the repo root. Requires macOS, Ghostty, `jq`, `python3`.

1. Link the skill:
   ```sh
   mkdir -p ~/.claude/skills
   ln -sfn "$PWD/skill/q" ~/.claude/skills/q
   chmod +x skill/q/close-tab
   ```
2. Add the hooks to `~/.claude/settings.json` (back it up first: `cp ~/.claude/settings.json ~/.claude/settings.json.bak`). Merge, don't overwrite: keep existing hooks, append these entries to `hooks.UserPromptSubmit` and `hooks.Stop` unless an entry with the same command is already there.
   ```json
   {
     "hooks": {
       "UserPromptSubmit": [
         { "hooks": [{ "type": "command", "command": "$HOME/.claude/skills/q/close-tab hook-prompt" }] }
       ],
       "Stop": [
         { "hooks": [{ "type": "command", "command": "$HOME/.claude/skills/q/close-tab hook-stop" }] }
       ]
     }
   }
   ```
   Or with a script:
   ```sh
   python3 - <<'EOF'
   import json, os
   p = os.path.expanduser('~/.claude/settings.json')
   d = json.load(open(p)) if os.path.exists(p) else {}
   cmd = '$HOME/.claude/skills/q/close-tab'
   for event, arg in (('UserPromptSubmit', 'hook-prompt'), ('Stop', 'hook-stop')):
       entries = d.setdefault('hooks', {}).setdefault(event, [])
       full = f'{cmd} {arg}'
       if not any(h.get('command') == full for e in entries for h in e.get('hooks', [])):
           entries.append({'hooks': [{'type': 'command', 'command': full}]})
   json.dump(d, open(p, 'w'), indent=2, ensure_ascii=False)
   EOF
   ```
3. Ghostty config: set `confirm-close-surface = false`, otherwise Ghostty asks before closing. Allow Ghostty notifications in macOS System Settings → Notifications to see the status.
4. Restart Claude Code (hooks and skills are read at session start; `/hooks` also reloads hooks).
5. Verify without closing anything: `Q_DRY_RUN=1 ~/.claude/skills/q/close-tab mark` prints the shell it would close. Then try `/q say hi` in a throwaway session.

Codex (no Stop hook; the skill runs `close-tab now 3 "<status>"` itself):
```sh
ln -sfn "$PWD/skill/q" ~/.codex/skills/q
```

## How it works

- `UserPromptSubmit` → `close-tab hook-prompt`: prompt starts or ends with `/q` → arm the tab and tell the agent to finish without questions and answer with one status line; otherwise disarm.
- `Stop` → `close-tab hook-stop`: if armed, send the first line of the final answer as an OSC 777 notification to the tab, then SIGHUP the shell 2 seconds later.
- If the agent needs the user, it runs `close-tab cancel` and the tab stays.

A closed tab can be restored with Cmd+Z within Ghostty's `undo-timeout`.
