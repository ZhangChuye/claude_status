# Claude Code Statusline — Install Prompt

## What it looks like

```
[Opus] (medium)  📁 todo_list | 🌿 main ~3
ctx   ███████░░░░░░░░░░░░░   35%
5h    ████░░░░░░░░░░░░░░░░   23%  resets in 1h09m
7d    ██████████████░░░░░░   74%  resets in 119h59m
```

- **Line 1** — model, effort, current dir, git branch with `+staged ~modified` counts
- **Line 2** — context window usage
- **Line 3** — 5-hour rate-limit window with reset countdown
- **Line 4** — 7-day rate-limit window with reset countdown

Bars are green (<70%), yellow (70–89%), red (≥90%).

## Requirements

- `python3` (no `jq` needed)
- Claude Code installed with `~/.claude/` present
- The `5h` / `7d` rows only populate for Claude.ai Pro/Max subscribers after the first API response; until then they show `waiting…`

## How to install

Copy everything in the fenced block below and paste it into Claude Code. Claude will create the three files and wire up your `~/.claude/settings.json` so the statusline shows up on the next interaction (no restart needed).

## Uninstall

Delete `~/.claude/statusline.py`, `~/.claude/statusline-command.sh`, and the `statusLine` key from `~/.claude/settings.json`.

---

````
Please install the following 4-line Claude Code statusline on my machine by creating these three files exactly as written, then running the install steps. Do not change the content.

---
File: ~/.claude/skills/install-statusline/SKILL.md
---

---
name: install-statusline
description: Install a 4-line aligned Claude Code status line showing model + git, context %, 5-hour rate limit, and 7-day rate limit, each with a colored progress bar. Use when the user asks to install, set up, reproduce, or copy this status line on the current machine.
---

# install-statusline

Reproduces the user's preferred Claude Code status line on the current machine.

## What it produces

```
[Opus] (medium)  :file_folder: todo_list | :herb: main ~3
ctx   ███████░░░░░░░░░░░░░   35%
5h    ████░░░░░░░░░░░░░░░░   23%  resets in 1h09m
7d    ██████████████░░░░░░   74%  resets in 119h59m
```

- Line 1: model, effort, current dir, git branch with `+staged ~modified` counts
- Line 2: context window usage
- Line 3: 5-hour rate-limit window with reset countdown
- Line 4: 7-day rate-limit window with reset countdown

Bars color green (<70%), yellow (70-89%), red (≥90%). Rows align on a 4-char label.

## Install steps

Run these commands. They copy the bundled scripts into `~/.claude/`, mark them executable, and merge a `statusLine` block into `~/.claude/settings.json` (preserving other keys).

```bash
SKILL_DIR="$HOME/.claude/skills/install-statusline"
cp "$SKILL_DIR/statusline.py" "$HOME/.claude/statusline.py"
cp "$SKILL_DIR/statusline-command.sh" "$HOME/.claude/statusline-command.sh"
chmod +x "$HOME/.claude/statusline-command.sh" "$HOME/.claude/statusline.py"

python3 - <<'PY'
import json, os
p = os.path.expanduser("~/.claude/settings.json")
data = {}
if os.path.exists(p):
    with open(p) as f:
        try:
            data = json.load(f)
        except json.JSONDecodeError:
            data = {}
data["statusLine"] = {
    "type": "command",
    "command": "bash " + os.path.expanduser("~/.claude/statusline-command.sh"),
    "refreshInterval": 30,
}
with open(p, "w") as f:
    json.dump(data, f, indent=2)
PY
```

Then verify with mock input:

```bash
FUTURE_5H=$(($(date +%s) + 4200))
FUTURE_7D=$(($(date +%s) + 432000))
echo "{\"model\":{\"display_name\":\"Opus\"},\"workspace\":{\"current_dir\":\"$PWD\"},\"context_window\":{\"used_percentage\":35},\"effort\":{\"level\":\"medium\"},\"rate_limits\":{\"five_hour\":{\"used_percentage\":23.5,\"resets_at\":$FUTURE_5H},\"seven_day\":{\"used_percentage\":74.2,\"resets_at\":$FUTURE_7D}}}" | bash "$HOME/.claude/statusline-command.sh"
```

The new layout takes effect on the next interaction with Claude Code (no restart needed).

## Notes

- `rate_limits` only appears for Claude.ai Pro/Max subscribers, after the first API response. Until then 5h and 7d rows show `waiting…`.
- `refreshInterval: 30` re-runs the script every 30s so the reset countdown ticks while idle.
- Requires `python3` (no `jq` dependency).

---
File: ~/.claude/skills/install-statusline/statusline-command.sh
---

```bash
#!/bin/bash
exec python3 "$HOME/.claude/statusline.py"
```

---
File: ~/.claude/skills/install-statusline/statusline.py
---

```python
#!/usr/bin/env python3
import json
import os
import subprocess
import sys
import time

data = json.load(sys.stdin)

model = data.get("model", {}).get("display_name", "Unknown")
cwd = data.get("workspace", {}).get("current_dir") or data.get("cwd") or ""
pct = int(data.get("context_window", {}).get("used_percentage", 0) or 0)
effort = data.get("effort", {}).get("level")

rate = data.get("rate_limits") or {}
five_h = (rate.get("five_hour") or {})
seven_d = (rate.get("seven_day") or {})
five_h_pct = five_h.get("used_percentage")
seven_d_pct = seven_d.get("used_percentage")
five_h_resets = five_h.get("resets_at")
seven_d_resets = seven_d.get("resets_at")

CYAN = "\033[36m"
GREEN = "\033[32m"
YELLOW = "\033[33m"
RED = "\033[31m"
BLUE = "\033[34m"
GRAY = "\033[90m"
RESET = "\033[0m"

bar_width = 20
LABEL_W = 4


def fmt_reset(epoch):
    if not epoch:
        return ""
    delta = int(epoch - time.time())
    if delta <= 0:
        return "now"
    h, rem = divmod(delta, 3600)
    m = rem // 60
    if h:
        return f"{h}h{m:02d}m"
    return f"{m}m"


def color_for(pct_val):
    if pct_val is None:
        return GRAY
    if pct_val >= 90:
        return RED
    if pct_val >= 70:
        return YELLOW
    return GREEN


def render_bar(pct_val):
    pct_int = int(pct_val)
    filled_n = min(pct_int * bar_width // 100, bar_width)
    return "█" * filled_n + "░" * (bar_width - filled_n)


def render_row(label, pct_val, resets_at=None):
    label_cell = f"{label:<{LABEL_W}}"
    if pct_val is None:
        return f"{label_cell}  {GRAY}waiting…{RESET}"
    bar_str = render_bar(pct_val)
    c = color_for(pct_val)
    pct_cell = f"{int(pct_val):>3}%"
    reset_str = fmt_reset(resets_at)
    suffix = f"  resets in {reset_str}" if reset_str else ""
    return f"{label_cell}  {c}{bar_str}{RESET}  {pct_cell}{suffix}"


git_info = ""
if cwd:
    try:
        subprocess.check_output(
            ["git", "-C", cwd, "rev-parse", "--git-dir"],
            stderr=subprocess.DEVNULL,
        )
        branch = subprocess.check_output(
            ["git", "-C", cwd, "branch", "--show-current"],
            text=True, stderr=subprocess.DEVNULL,
        ).strip()
        staged_out = subprocess.check_output(
            ["git", "-C", cwd, "diff", "--cached", "--numstat"],
            text=True, stderr=subprocess.DEVNULL,
        ).strip()
        modified_out = subprocess.check_output(
            ["git", "-C", cwd, "diff", "--numstat"],
            text=True, stderr=subprocess.DEVNULL,
        ).strip()
        staged = len(staged_out.splitlines()) if staged_out else 0
        modified = len(modified_out.splitlines()) if modified_out else 0
        git_info = f" | {BLUE}:herb: {branch}{RESET}"
        if staged:
            git_info += f" {GREEN}+{staged}{RESET}"
        if modified:
            git_info += f" {YELLOW}~{modified}{RESET}"
    except Exception:
        pass

effort_info = f" {GRAY}({effort}){RESET}" if effort else ""
dirname = os.path.basename(cwd) if cwd else ""

print(f"{CYAN}[{model}]{RESET}{effort_info}  :file_folder: {dirname}{git_info}")
print(render_row("ctx", pct))
print(render_row("5h", five_h_pct, five_h_resets))
print(render_row("7d", seven_d_pct, seven_d_resets))
```

After creating the three files, run the install bash block from SKILL.md so the `statusLine` entry is merged into `~/.claude/settings.json`. Then run the verify command to confirm it renders.
````
