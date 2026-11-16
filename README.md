# agent-quit

`/q` for Claude Code: do the last task, show the result as a Ghostty notification, close the terminal tab.

```
commit and push /q
```

## Requirements

- macOS
- [Ghostty](https://ghostty.org) terminal
- `jq`: a command-line JSON tool the hooks use to read Claude Code's hook input (`brew install jq`)

## Install with Claude

Paste this into Claude Code:

```
Install agent-quit for me: follow the "Manual install" section of https://github.com/nicknotknack/agent-quit/blob/master/README.md step by step, then verify it.
```

## Manual install

1. Install `jq` if it is missing: `brew install jq`.
2. Clone and link the skill:
   ```sh
   git clone https://github.com/nicknotknack/agent-quit ~/.agent-quit
   mkdir -p ~/.claude/skills
   ln -sfn ~/.agent-quit/skill/q ~/.claude/skills/q
   ```
3. Add the hooks to `~/.claude/settings.json`. Back the file up first. Merge with existing `hooks`, don't overwrite them, and skip an entry that is already there:
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
4. Ghostty config (`~/.config/ghostty/config`): set `confirm-close-surface = false` so the tab closes without asking.
5. Allow Ghostty notifications: System Settings → Notifications → Ghostty.
6. Verify: `Q_DRY_RUN=1 ~/.claude/skills/q/close-tab now` prints the shell it would close.
7. Restart Claude Code.

## How it works

- `UserPromptSubmit` hook: a prompt that starts or ends with `/q` arms the tab; any other prompt disarms it.
- `Stop` hook: if armed, the first line of the answer is shown as a Ghostty notification, and the tab's shell gets `SIGHUP` 2 s later.
- If the agent needs you, it runs `close-tab cancel` and the tab stays.

Ghostty doesn't show notifications for the tab you are looking at. Undo a closed tab with `Cmd+Z`.
