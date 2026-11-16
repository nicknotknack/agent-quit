# agent-quit

`/q` for Claude Code: do the last task, show a one-line status notification, close the terminal tab.

```
/q commit and push
commit and push /q
```

Requires macOS, [Ghostty](https://ghostty.org), `jq`.

## Install

```sh
git clone https://github.com/nicknotknack/agent-quit ~/.agent-quit
mkdir -p ~/.claude/skills
ln -sfn ~/.agent-quit/skill/q ~/.claude/skills/q
```

Add the hooks to `~/.claude/settings.json` (merge with existing `hooks`):

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

In the Ghostty config set `confirm-close-surface = false`, then restart Claude Code.

Check: `Q_DRY_RUN=1 ~/.claude/skills/q/close-tab now` prints the shell it would close.

## How it works

- `UserPromptSubmit`: a prompt that starts or ends with `/q` arms the tab; any other prompt disarms it.
- `Stop`: if armed, the first line of the answer goes to a macOS notification and the tab's shell gets `SIGHUP` 2 s later.
- If the agent needs you, it runs `close-tab cancel` and the tab stays.

Undo a closed tab with `Cmd+Z` (Ghostty `undo-timeout`).
