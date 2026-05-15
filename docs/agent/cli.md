# `arduino-app-cli` & board access

The CLI is preinstalled on the UNO Q. It is **not** installed on a typical dev machine. As an agent you operate in one of two execution contexts, and you must detect which one before running any deploy/lifecycle command.

## Execution context

| Context | How to detect | How you run commands |
| --- | --- | --- |
| **On-board** (running on the UNO Q Linux side) | `command -v arduino-app-cli` resolves, and `hostname` matches the board name (or `/etc/os-release` mentions Arduino). | Run `arduino-app-cli ...` directly. The app's code is already on the board at `/home/arduino/ArduinoApps/<app-name>/`. |
| **Off-board** (running on a dev machine, e.g. macOS/Linux laptop) | `command -v arduino-app-cli` fails, but `command -v adb` works and `adb devices` lists the board. | Push code with `adb push`, then run lifecycle commands via `adb shell arduino-app-cli ...`. |

If neither path is available, stop and ask the user to either connect the UNO Q over USB-C (so `adb` sees it) or run the agent on the board.

## Off-board: push code to the board (ADB)

From the repo root on your dev machine:

```bash
APP=<app-name>         # the directory name under ArduinoApps; e.g. my-app
DEST=/home/arduino/ArduinoApps/$APP

adb shell "mkdir -p $DEST"
adb push ./ "$DEST/"
adb shell "chown -R arduino:arduino $DEST"
```

To pull state/logs back:

```bash
adb pull /home/arduino/ArduinoApps/$APP ./local-folder
```

## App lifecycle

On-board:

```bash
arduino-app-cli app new "my-app"                          # scaffold a new app
arduino-app-cli app list                                  # list apps + status
arduino-app-cli app start /home/arduino/ArduinoApps/my-app
arduino-app-cli app start user:my-app                     # shortcut
arduino-app-cli app start examples:blink                  # built-in example
arduino-app-cli app stop  /home/arduino/ArduinoApps/my-app
arduino-app-cli app logs  /home/arduino/ArduinoApps/my-app --all
```

Off-board — prefix every command with `adb shell`:

```bash
adb shell arduino-app-cli app list
adb shell arduino-app-cli app start /home/arduino/ArduinoApps/my-app
adb shell arduino-app-cli app stop  /home/arduino/ArduinoApps/my-app
adb shell arduino-app-cli app logs  /home/arduino/ArduinoApps/my-app --all
```

Use `adb shell -t` if you need a TTY (e.g. for streaming logs interactively).

## Bricks (read-only)

```bash
arduino-app-cli brick list
arduino-app-cli brick details arduino:<brick>
```

You add bricks by editing `app.yaml` directly (see `AGENTS.md` §3.2 and §7). The CLI itself cannot add/remove them.

## System

```bash
arduino-app-cli system update
arduino-app-cli system set-name "my-board"
arduino-app-cli system network enable|disable    # SSH on/off
arduino-app-cli system cleanup                   # reclaim disk
```

## Typical agent deploy loop (off-board)

```bash
APP=$(basename "$PWD")
DEST=/home/arduino/ArduinoApps/$APP

adb shell "mkdir -p $DEST"
adb push ./ "$DEST/"
adb shell "chown -R arduino:arduino $DEST"
adb shell arduino-app-cli app stop  "$DEST" 2>/dev/null || true
adb shell arduino-app-cli app start "$DEST"
adb shell arduino-app-cli app logs  "$DEST" --all
```
