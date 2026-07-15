---
name: naoqi-2-5-10-7
description: Use when writing, debugging, or reviewing code for a NAO or Pepper robot on NAOqi OS 2.5.10.7 — both Python against the NAOqi SDK (ALProxy or qi framework, ALMotion/ALTextToSpeech/ALVideoDevice/ALMemory/ALRobotPosture/etc, event subscriptions, sync vs async calls, deployment) and Choregraphe .xar behaviors/boxes, including the strict constraints of the Choregraphe virtual robot (no python-script boxes; keyframe animation only).
---

# NAOqi 2.5.10.7

Guidance for writing Python against the NAOqi SDK targeting **NAOqi OS 2.5.10.7** (NAO / Pepper robots).

## Hard constraint: Python 2.7 only

The NAOqi 2.5.x Python SDK (`pynaoqi`) only works with **Python 2.7**. There is no
Python 3 build for this version line. Before writing code:

- Confirm the target interpreter is 2.7 (`python2 --version` / a 2.7 virtualenv).
- Use `print "x"` statement syntax, not `print("x")` — though the latter still
  parses in 2.7 as a no-op tuple print for single args, don't rely on it for
  anything more complex.
- `from __future__ import print_function` is fine if consistency with 3.x-style
  code is wanted, but don't introduce Python 3 syntax (f-strings, `async def`,
  type hints beyond comments) — it will fail to import.
- The SDK must be on `PYTHONPATH`, e.g.
  `export PYTHONPATH=${PYTHONPATH}:/path/to/pynaoqi-python2.7/lib`.

## Two ways to connect

### 1. `ALProxy` (simple, most common for scripts)

```python
from naoqi import ALProxy

ip = "192.168.1.42"   # robot's IP
port = 9559            # default NAOqi port

tts = ALProxy("ALTextToSpeech", ip, port)
tts.say("Hello, human.")
```

- Raises `RuntimeError` on connection failure or bad method calls — wrap
  calls that might fail (robot offline, module not present) in `try/except RuntimeError`.
- Every method is available as-is (blocking/synchronous) and also via
  `.post` for a fire-and-forget async call that returns an ID:

```python
motion = ALProxy("ALMotion", ip, port)
motion.setStiffnesses("Body", 1.0)
task_id = motion.post.moveTo(0.5, 0, 0)   # non-blocking
motion.wait(task_id, 0)                    # block until it finishes
```

### 2. `qi` framework / session (used by Choregraphe-generated behaviors, more modern style)

```python
import qi
import sys

def main():
    app = qi.Application(sys.argv, url="tcp://%s:%d" % (ip, port))
    app.start()
    session = app.session()

    tts = session.service("ALTextToSpeech")
    tts.say("Hello from the qi framework.")

    app.run()   # blocks; needed if subscribing to events/signals

if __name__ == "__main__":
    main()
```

- `session.service("ModuleName")` returns the same kind of proxy object as `ALProxy`.
- Prefer this style for **behaviors that run on the robot itself** (installed via
  Choregraphe / `qicli`) or that need to register their own service back to NAOqi.
- Use `ALProxy` for **standalone scripts run from a dev machine** talking to the
  robot over the network — it's less boilerplate.

Don't mix idioms within one script without a reason — pick one connection style per script.

## Common modules and what they're for

| Module | Purpose |
|---|---|
| `ALMotion` | Joint control, walking (`moveTo`, `moveToward`), stiffness, `angleInterpolation`/`angleInterpolationWithSpeed` |
| `ALRobotPosture` | Named postures: `goToPosture("StandInit", 0.5)` |
| `ALTextToSpeech` | `say(text)`, `setLanguage`, `setVolume` |
| `ALAudioPlayer` | Play sound files from the robot's filesystem |
| `ALVideoDevice` | Camera access: `subscribe`, `getImageRemote`, `unsubscribe` |
| `ALFaceDetection` | Face detection events via `ALMemory` |
| `ALMemory` | Central key/value blackboard + event subscription |
| `ALNavigation` | Autonomous navigation / mapping (Pepper) |
| `ALTracker` | Track a target (face, sound, object) with head/body motion |
| `ALBehaviorManager` | Run/stop behaviors installed on the robot |

## Event subscription pattern (ALMemory)

Events (face detected, word recognized, touch sensors, etc.) go through `ALMemory`.
This requires a broker/callback module, not a plain script-level function:

```python
from naoqi import ALProxy, ALBroker, ALModule

class ReactionModule(ALModule):
    def __init__(self, name):
        ALModule.__init__(self, name)
        self.memory = ALProxy("ALMemory")
        self.memory.subscribeToEvent(
            "FaceDetected", name, "onFaceDetected"
        )

    def onFaceDetected(self, key, value, message):
        print "Face detected:", value

broker = ALBroker("myBroker", "0.0.0.0", 0, ip, port)
reaction = ReactionModule("reaction")
# keep the process alive (e.g. a loop or app.run()) or the broker dies with it
```

- `ALBroker` must stay alive for the duration the module listens for events —
  don't let the script exit immediately after subscribing.
- Always unsubscribe (`self.memory.unsubscribeToEvent(...)`) in cleanup/`finally`
  if the module is long-lived and re-subscribes across restarts, to avoid
  duplicate callbacks.

## Gotchas specific to 2.5.10.7

- **Stiffness before motion**: joints won't move if stiffness is 0. Call
  `motion.setStiffnesses("Body", 1.0)` (or the relevant chain name) before
  posture/motion commands, and consider setting it back to 0 on cleanup so
  the robot doesn't hold a rigid pose indefinitely (heat/battery).
- **Blocking vs `.post`**: default calls block until the action completes.
  For anything that takes real time (walking, posture changes, long TTS),
  decide deliberately whether the script should block or use `.post` +
  `motion.wait(id, timeoutMs)`.
- **IP/port must be reachable**: NAOqi listens on port `9559` by default;
  robot and dev machine must be on the same network/VPN. Connection
  failures surface as `RuntimeError: ALProxy::ALProxy` — that's a
  connectivity issue, not an API misuse issue.
- **Deploying scripts to the robot** (vs. running from a dev machine) means
  the robot's own Python 2.7 and installed `naoqi` module are used — no pip
  installs of third-party packages are available unless manually placed on
  the robot's filesystem.
- **Choregraphe boxes (physical robot)**: Python code embedded in Choregraphe
  boxes runs inside NAOqi's process and receives `self` with
  `self.getParentBroker()` already wired up — don't create a new `ALBroker`
  inside a box's script. **This only holds on a physical robot** — on the
  Choregraphe virtual robot, a box that contains a python script at all fails
  to load (see the section below).

## Virtual robot (Choregraphe) constraints

When the target is the **Choregraphe virtual robot** (no physical NAO/Pepper),
a large part of the API silently does nothing or hard-fails. Confirm which
target you're on before writing behaviors.

- **No box may contain a python script.** Any Choregraphe box whose
  `<script language="4">` holds real code — *including the default
  `class MyClass(GeneratedClass)` boilerplate that stock library boxes ship
  with* — crashes on load with:

  ```
  NameError: name 'ALBehavior' is not defined
  FMBox::createPythonModule
  ```

  The trigger is the box creating a python module *at all*, not anything the
  script does. This reproduces on stock, unedited boxes (e.g. the "Say" box and
  the library "Goto Posture" box).

- **Verified-working pattern:** keyframed animation boxes with an **empty**
  script and a `<Timeline>` of `ActuatorCurve` keys:

  ```xml
  <script language="4"><content><![CDATA[]]></content></script>
  ```

  When pulling a box out of the Choregraphe library, **strip its `MyClass`
  boilerplate script** before wiring it in, or the behavior won't load.

- **Confirmed broken on the virtual robot** — do not rely on these (flag to the
  user before using any `ALProxy`-based box):
  - `ALTextToSpeech` / stock "Say" box — no audio out
  - `ALRobotPosture` / stock "Goto Posture" box — so there's **no way to send
    the robot to `StandInit`**; animations must start from whatever pose the
    robot is already in
  - `ALVideoDevice` / camera / vision
  - real touch / sonar / FSR sensors
  - audio input / speech recognition

- Don't assume any `ALProxy`-based box works on the virtual robot; the
  keyframe-animation boxes above are the only verified-safe building block.

## Reference docs and library paths

- **Docs version matters.** Use `http://doc.aldebaran.com/2-5/` as the
  documentation root for NAOqi 2.5.x — *not* `2-1` or `2-4`, which describe
  older SDK/Choregraphe versions and can give wrong API/behavior details.
- **Predefined Choregraphe behaviors/libraries** ship at (Suite 2.5, Windows):
  `C:\Program Files (x86)\Softbank Robotics\Choregraphe Suite 2.5\share\choregraphe\libraries`
  Check here for an existing box/animation before writing a new one — but strip
  any embedded script before using it on the virtual robot (see above).

## When helping with this code

- Ask/confirm the robot's IP and whether the script runs from a dev machine
  or is deployed to the robot, since that decides `ALProxy` vs `qi` session
  style and whether a broker is needed.
- Keep error handling scoped to real failure modes (robot unreachable,
  module not present, stiffness not set) rather than generic try/except
  around everything.
- Don't suggest Python 3 syntax, f-strings, or `asyncio` — none of it runs
  under the 2.5.10.7 SDK.
