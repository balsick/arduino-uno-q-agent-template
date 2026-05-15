# Bridge — Python ↔ MCU RPC

The Bridge is the RPC layer between Python (`python/main.py`, Linux side) and the C++ sketch (`sketch/sketch.ino`, MCU side). It is provided by the `Arduino_RouterBridge` library on the MCU and by `arduino.app_utils.Bridge` / `App` on Python.

## Python API (`from arduino.app_utils import App, Bridge`)

| Call | Purpose |
| --- | --- |
| `Bridge.call(name, *args)` | Synchronously invoke a function exposed by the MCU via `Bridge.provide_safe` / `Bridge.provide`. Returns the MCU's return value. |
| `Bridge.provide(name, fn)` | Expose a Python function so the MCU can invoke it with `Bridge.call(name, ...)`. |
| `App.provide(name, fn)` | Alias used in some docs — equivalent to `Bridge.provide` for the calling-from-MCU case. |
| `App.run(user_loop=None)` | Start the app. With `user_loop`, the given function runs repeatedly. Without it, only bricks and Bridge callbacks drive the app. MUST be the last line. |

## C++ API (`#include <Arduino_RouterBridge.h>`)

| Call | Purpose |
| --- | --- |
| `Bridge.begin()` | Initialize the Bridge. Call in `setup()`. |
| `Bridge.provide_safe(name, fn)` | Expose a C++ function callable from Python. Runs in `loop()` context — **use this by default**. |
| `Bridge.provide(name, fn)` | Same as above but runs in the Bridge thread. Only use if you know what you're doing (no hardware access, no interrupts). |
| `Bridge.call(name, ...)` | Synchronously invoke a Python function exposed via `Bridge.provide`. |
| `Bridge.notify(name, ...)` | Fire-and-forget call into Python. No return value, doesn't block. Use this for high-frequency telemetry (sensor samples, etc.). |
| `Monitor.begin()` / `Monitor.print()` / `Monitor.println()` | Logging into the App Lab Serial Monitor. `Serial.print` works too on Zephyr core ≥ 0.55.0. |

## Type matching

Arguments are typed across the wire. Mismatches **fail silently**.

| Python | C++ |
| --- | --- |
| `int` | `int` (32-bit) |
| `float` | `float` |
| `bool` | `bool` |
| `str` | `const char*` / `String` |

Always declare the C++ function signature explicitly and pass matching types from Python.

## Anti-patterns (cause deadlocks / hangs)

1. Calling `Bridge.call(...)` (in either direction) from inside a function that was exposed via `Bridge.provide*`.
2. Calling `Monitor.print(...)` inside a `Bridge.provide*` callback on the MCU.
3. Long `delay()` (MCU) or `time.sleep()` (Python) inside loops or callbacks — the Bridge can't service requests while blocked.
4. Hand-installing the `Arduino_RouterBridge` library on Zephyr core ≥ 0.55.0 (already bundled).

## Pattern: Python → MCU command

```python
# python/main.py
from arduino.app_utils import App, Bridge

def loop():
    Bridge.call("set_brightness", 128)

App.run(user_loop=loop)
```

```cpp
// sketch/sketch.ino
#include <Arduino_RouterBridge.h>

void set_brightness(int level) {
  analogWrite(LED_BUILTIN, level);
}

void setup() {
  Monitor.begin();
  pinMode(LED_BUILTIN, OUTPUT);
  Bridge.begin();
  Bridge.provide_safe("set_brightness", set_brightness);
}

void loop() {}
```

## Pattern: MCU → Python telemetry (high-frequency)

Use `Bridge.notify` from the MCU to push samples without blocking.

```cpp
// sketch/sketch.ino
unsigned long last = 0;

void loop() {
  unsigned long now = millis();
  if (now - last >= 16) {                 // ~62.5 Hz
    last = now;
    float x = readAccelX();
    float y = readAccelY();
    float z = readAccelZ();
    Bridge.notify("on_sample", x, y, z);  // no return, no block
  }
}
```

```python
# python/main.py
from arduino.app_utils import App, Bridge

def on_sample(x: float, y: float, z: float):
    # buffer / process
    ...

Bridge.provide("on_sample", on_sample)
App.run()
```

## Pattern: Bidirectional

```python
# python/main.py
from arduino.app_utils import App, Bridge

def on_button(state: int):
    print(f"Button: {state}")
    Bridge.call("ack", state)   # OK: not inside a provide callback? -> NO, this IS inside one.

Bridge.provide("on_button", on_button)
App.run()
```

The example above is **wrong**: `Bridge.call("ack", ...)` is invoked inside the handler exposed via `Bridge.provide("on_button", ...)`. Move the outbound call to a separate flow (e.g., a periodic loop or a queue consumed elsewhere) to avoid deadlocking the Bridge.
