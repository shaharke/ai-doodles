# Claude Code Status Line Setup

A clean, info-rich status line for Claude Code using [ccstatusline](https://github.com/sirmalloc/ccstatusline).

**Preview:**
```
📁 /Users/you/project │ 🌿 feature/my-branch │ 🤖 Opus 4.6 │ 🧠 123.2k/1M (12%) │ 💰 $2.34 │ ⏱ 30m
```

## Setup (2 minutes)

**1. Save the context script** to `~/.claude/ccsl-context.sh`:

```bash
#!/bin/bash
input=$(cat)
pct=$(echo "$input" | jq -r '.context_window.used_percentage // 0')
size=$(echo "$input" | jq -r '.context_window.context_window_size // 200000')
usage_raw=$(echo "$input" | jq -r '(.context_window.current_usage.input_tokens // 0) + (.context_window.current_usage.cache_creation_input_tokens // 0) + (.context_window.current_usage.cache_read_input_tokens // 0)')
if [ "$usage_raw" -gt 0 ] 2>/dev/null; then
  usage=$(awk "BEGIN {printf \"%.1fk\", $usage_raw/1000}")
else
  usage="0k"
fi
if [ "$size" -ge 1000000 ] 2>/dev/null; then
  max="1M"
elif [ "$size" -ge 200000 ] 2>/dev/null; then
  max="200k"
else
  max=$(awk "BEGIN {printf \"%.0fk\", $size/1000}")
fi
pct_int=$(printf "%.0f" "$pct" 2>/dev/null || echo "0")
echo "${usage}/${max} (${pct_int}%)"
```

Then: `chmod +x ~/.claude/ccsl-context.sh`

**2. Save the ccstatusline config** to `~/.config/ccstatusline/settings.json`:

```json
{
  "version": 3,
  "lines": [
    [
      { "id": "e1", "type": "custom-text", "customText": "📁 ", "color": "white", "merge": "no-padding" },
      { "id": "1", "type": "current-working-dir", "color": "cyan", "rawValue": true, "merge": true },
      { "id": "s1", "type": "separator", "character": " │ " },
      { "id": "e2", "type": "custom-text", "customText": "🌿 ", "color": "white", "merge": "no-padding" },
      { "id": "2", "type": "git-branch", "color": "magenta", "rawValue": true, "merge": true, "maxWidth": 35 },
      { "id": "s2", "type": "separator", "character": " │ " },
      { "id": "e3", "type": "custom-text", "customText": "🤖 ", "color": "white", "merge": "no-padding" },
      { "id": "3", "type": "model", "color": "#5dade2", "rawValue": true, "merge": true },
      { "id": "s3", "type": "separator", "character": " │ " },
      { "id": "e4", "type": "custom-text", "customText": "🧠 ", "color": "white", "merge": "no-padding" },
      { "id": "4", "type": "custom-command", "commandPath": "$HOME/.claude/ccsl-context.sh", "color": "#2ecc71", "merge": true, "timeout": 2000 },
      { "id": "s4", "type": "separator", "character": " │ " },
      { "id": "e5", "type": "custom-text", "customText": "💰 ", "color": "white", "merge": "no-padding" },
      { "id": "5", "type": "session-cost", "color": "#f39c12", "rawValue": true, "merge": true },
      { "id": "s5", "type": "separator", "character": " │ " },
      { "id": "e6", "type": "custom-text", "customText": "⏱ ", "color": "white", "merge": "no-padding" },
      { "id": "6", "type": "session-clock", "color": "#aab7b8", "rawValue": true, "merge": true }
    ],
    [],
    []
  ],
  "flexMode": "full",
  "compactThreshold": 60,
  "colorLevel": 3,
  "inheritSeparatorColors": false,
  "globalBold": false,
  "powerline": {
    "enabled": false,
    "separators": [""],
    "separatorInvertBackground": [false],
    "startCaps": [],
    "endCaps": [],
    "autoAlign": false
  }
}
```

> **Note:** Replace `$HOME` in `commandPath` with your actual home directory path if ccstatusline doesn't resolve it.

**3. Update Claude Code settings** — add to `~/.claude/settings.json`:

```json
"statusLine": {
  "type": "command",
  "command": "bunx ccstatusline@latest"
}
```

> If `bunx` isn't on PATH, use the full path (e.g. `~/.bun/bin/bunx ccstatusline@latest`).

**4. Restart Claude Code.**

## Requirements

- `jq` (for the context script)
- `bun` or `npm` (for `bunx`/`npx`)

Make sure these are installed and if not install them for the user

## Customize

Run `bunx ccstatusline@latest` in a terminal to launch the interactive TUI configurator
