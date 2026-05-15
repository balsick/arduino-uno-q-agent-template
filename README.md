# arduino-uno-q-agent-template

A starter template for building **Arduino UNO Q** (Arduino App Lab) applications
**with an AI coding agent** (Copilot CLI, Claude Code, Cursor, etc.).

This repo ships only the pieces an agent needs to bootstrap a real project:

```
AGENTS.md                # the rules. Always in the agent's context.
CLAUDE.md                # symlink → AGENTS.md
docs/agent/              # on-demand reference docs the agent loads per task
  bridge.md              # Python ↔ MCU Bridge (RPC, callbacks, types)
  cli.md                 # arduino-app-cli + on-board vs off-board (adb) workflow
  frontend.md            # Web UI conventions (HTML/CSS/JS, Socket.IO, vendoring)
  bricks/                # one file per brick (web_ui, motion_detection, …)
    _template.md         # copy this when adding a new brick doc
.gitignore
```

No `python/`, `sketch/`, `assets/` or `app.yaml` — the agent creates those for
your specific app.

## How to use this template

1. Click **"Use this template"** on GitHub (or `gh repo create --template
   balsick/arduino-uno-q-agent-template ...`).
2. Open the new repo in your agent of choice. The agent will read `AGENTS.md`
   (or `CLAUDE.md`) automatically.
3. Ask for what you want to build, e.g. _"a web UI that streams accelerometer
   data from the UNO Q and persists peaks to SQLite"_.
4. The agent scaffolds `python/`, `sketch/`, `assets/`, `app.yaml`,
   `sketch/sketch.yaml`, picks the right bricks, and deploys to the board.

## What the agent is allowed to do

- **Edit `app.yaml` directly**, including the `bricks:` list.
- **Edit `sketch/sketch.yaml` directly**, including the `libraries:` list.
- **Deploy & run the app** via `arduino-app-cli` (on the board) or
  `adb push` + `adb shell arduino-app-cli` (from your dev machine). See
  [`docs/agent/cli.md`](docs/agent/cli.md).
- **Adopt an undocumented brick** by scraping
  [`arduino/app-bricks-examples`](https://github.com/arduino/app-bricks-examples)
  and [`arduino/app-bricks-py`](https://github.com/arduino/app-bricks-py), then
  drafting a doc under `docs/agent/bricks/<brick>.md` — after asking you to
  confirm.

## How the template self-improves

Whenever the agent has to document a previously-unknown brick (see `AGENTS.md`
§8), it also opens a pull request against this template repository
(`balsick/arduino-uno-q-agent-template`) adding the same brick doc. Over time
the template accumulates coverage of every brick the community uses.

## Deploy cheat-sheet

**On-board** (agent running on the UNO Q's Linux side):

```bash
arduino-app-cli app start /home/arduino/ArduinoApps/<app>
arduino-app-cli app logs  /home/arduino/ArduinoApps/<app> --all
```

**Off-board** (agent running on your laptop, UNO Q on USB-C):

```bash
APP=$(basename "$PWD")
DEST=/home/arduino/ArduinoApps/$APP
adb shell "mkdir -p $DEST"
adb push ./ "$DEST/"
adb shell "chown -R arduino:arduino $DEST"
adb shell arduino-app-cli app start "$DEST"
adb shell arduino-app-cli app logs  "$DEST" --all
```

Open the Web UI (if any) from any browser on the same network:

```
http://<board-name>.local:7000/
```

## Hardware

- Arduino UNO Q (Zephyr core ≥ 0.55.0)
- USB-C cable (for `adb` from a dev machine)

## Conventions

- Code comments and docs in English.
- Python deps in `python/requirements.txt` only.
- Third-party JS/CSS vendored under `assets/js/vendor/` (or
  `assets/css/vendor/`); no CDN links at runtime.
- Keep changes minimal and focused — no speculative refactors.

## License

This template carries no code. Add your own `LICENSE` when you derive a
project from it.
