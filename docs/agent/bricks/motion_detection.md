# `motion_detection` brick

Real-time motion / gesture classification from accelerometer samples, using a pre-trained or Edge-Impulse-trained model. Source: `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/motion_detection`.

## Install

User must add the **Motion Detection** brick from the App Lab GUI. The sketch must read accelerometer data and forward samples to Python via the Bridge. If using a Modulino Movement, the user must add the **Modulino** sketch library via the App Lab GUI.

## Python API

```python
from arduino.app_bricks.motion_detection import MotionDetection

motion = MotionDetection(confidence=0.4)
```

| Method | Signature | Purpose |
| --- | --- | --- |
| `accumulate_samples` | `(sample: tuple[float, float, float])` | Append one `(x, y, z)` sample (in **m/s²**) to the sliding-window buffer. Call once per accelerometer reading. |
| `on_movement_detection` | `(label: str, cb: Callable[[dict], None])` | Register a callback fired when the model classifies a motion with confidence ≥ `confidence`. `label` is the model class name (e.g., `"updown"`). The callback receives the full classification dict. |

The brick handles buffering, sliding window, and inference internally. You only feed samples and react to detections.

## Sketch-side pattern

Sample the accelerometer at the rate the model was trained on (Arduino's default model: **62.5 Hz → 16 ms period**). Push samples to Python with `Bridge.notify` (fire-and-forget, doesn't block the loop).

```cpp
#include <Arduino_RouterBridge.h>
#include <Modulino.h>

ModulinoMovement movement;
unsigned long last = 0;
const long interval = 16;  // ms — matches 62.5 Hz training rate

void setup() {
  Monitor.begin();
  Bridge.begin();
  Modulino.begin(Wire1);
  while (!movement.begin()) {
    delay(1000);
  }
}

void loop() {
  unsigned long now = millis();
  if (now - last < interval) return;
  last = now;

  if (movement.update() != 1) return;
  float x = movement.getX();   // values in g
  float y = movement.getY();
  float z = movement.getZ();
  Bridge.notify("record_sensor_movement", x, y, z);
}
```

## Minimal example (`python/main.py`)

```python
from arduino.app_utils import App, Bridge
from arduino.app_bricks.motion_detection import MotionDetection

motion = MotionDetection(confidence=0.4)

def record_sensor_movement(x: float, y: float, z: float):
    # convert g -> m/s^2 (model expects SI units)
    motion.accumulate_samples((x * 9.81, y * 9.81, z * 9.81))

Bridge.provide("record_sensor_movement", record_sensor_movement)

def on_updown(classification: dict):
    print(f"updown detected: {classification}")

motion.on_movement_detection("updown", on_updown)

App.run()
```

## Alternative sample sources

`accumulate_samples` only needs `(x, y, z)` floats in m/s². The Modulino is the typical source, but **any** stream works:

- A phone streaming over [`remote_sensor`](remote_sensor.md) (keys `accelerometerX/Y/Z`).
- An I²C IMU read by the sketch and forwarded via `Bridge.notify`.

The brick is sensor-agnostic; only the sample rate has to match what the model was trained on.

## Models

The default framework model classifies `updown` (vertical shake) and similar simple gestures at **62.5 Hz**. The public Edge-Impulse project used by the framework is:

`https://studio.edgeimpulse.com/public/497631/latest`

Open it to see the full class list, retrain with your own data, or export a fresh model. When you swap the model, double-check the **sample rate** and **class labels** — both must match the new project.

## Gotchas

- **Units**: source may be in **g**; the brick expects **m/s²**. Multiply by `9.81` (as above).
- **Sample rate**: must match what the model was trained on. Default 62.5 Hz (16 ms period). Wrong rate → garbage classifications.
- Use `Bridge.notify` (not `Bridge.call`) from the MCU for sample push — fire-and-forget, won't stall the loop at 62.5 Hz.
- Argument types must match (`float` ↔ `float`). Don't send `int`s.
- Available class labels depend on the deployed model. Register one `on_movement_detection` per label you care about.
- `confidence` is a hard threshold — callbacks fire only when the model's score for the label meets it.
- Don't call `Bridge.call(...)` inside `record_sensor_movement` (it is itself a `Bridge.provide` handler — deadlock). It IS safe to call `Bridge.call(...)` from a `RemoteSensor.on_datapoint` callback feeding samples in.
