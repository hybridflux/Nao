# Pepper Local LLM — Project Plan & Feasibility Recipe

Giving Pepper the ability to understand spoken/typed questions and respond
conversationally, using an **open-source small language model (SLM) running
locally** — ideally on Pepper's own onboard Linux, with a companion-machine
fallback.

- **Robot:** Pepper, NAOqi OS **2.5.10.7**
- **Unit tag:** `AP990237I00Y62100482`
- **Status:** planning + step-1 feasibility (not yet built)

---

## 1. Goals

1. Improve language recognition (move away from NAOqi's fixed-vocabulary
   `ALSpeechRecognition` word-spotting toward real speech-to-text).
2. Make Pepper's responses less programmatic — understand a question and reply
   in natural language.
3. Do this with a **local, open-source LLM/SLM** — no cloud, no API cost,
   offline-capable — running on Pepper's OS where feasible.

---

## 2. Hardware constraints (the binding limit)

NAOqi **2.5.10.7** places this unit in the **Pepper 1.x generation**, whose
onboard computer is:

| Spec | Value | Consequence for LLMs |
|---|---|---|
| CPU | Intel Atom **E3845**, 4 cores @ ~1.9 GHz | Weak; **no AVX/AVX2**, only SSE4.2 → slow inference path |
| RAM | **4 GB**, shared with NAOqi/OS | ~2 GB realistically free → sub-2B models only |
| GPU | none | CPU-only inference |
| Storage | ~32 GB flash | Fine for a few small models |
| OS | Gentoo-based Linux (NAOqi OS) | Old glibc/toolchain → **static binaries** are safest |
| On-robot Python | **2.7 only** | Cannot host the modern AI stack directly → keep AI as separate binaries |

**Realistic expectation:** single-digit tokens/sec, multi-second replies. A
tightly-scoped persona + short replies is what makes a sub-2B model feel good.

### Two structural facts that shape the design

- **Virtual robot can't do this.** The Choregraphe virtual robot has no audio
  input, no speech recognition, and no TTS. This capability requires the
  **physical** Pepper. (This overrides the repo's "virtual-robot-only" rule for
  this feature.)
- **Python 2.7 vs. modern AI.** NAOqi's SDK is Python 2.7; modern inference
  tooling is C/C++ or Python 3. Solution: run the AI as **standalone C++
  binaries** (llama.cpp / whisper.cpp) and have a thin Python 2.7 NAOqi behavior
  orchestrate them over `localhost` HTTP. No pip installs on the robot.

---

## 3. Confirm the hardware (run first, on the robot)

SSH in (`ssh nao@<pepper-ip>`) and run these read-only checks. The output
decides model size.

```sh
# MOST DECISIVE: kernel arch + userland bitness. NAOqi OS has historically
# shipped a 32-bit (i686) userland — this determines the ENTIRE build target.
uname -a
uname -m            # x86_64 vs i686/i586  → 64-bit vs 32-bit build
getconf LONG_BIT    # 64 or 32
uname -r            # kernel version → affects whether a modern static binary runs

# DECISIVE: AVX/AVX2 vs. only SSE4.2
grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E 'avx|avx2|sse4_2'

# CPU model and core count
grep -m1 'model name' /proc/cpuinfo
nproc

# RAM: total vs. actually available (the real budget)
grep -E 'MemTotal|MemAvailable' /proc/meminfo

# Storage headroom + is /home/nao writable and NOT noexec?
df -h /
mount | grep -E ' /home| / '        # look for 'ro' or 'noexec' flags
touch /home/nao/_wtest && echo "writable" && rm -f /home/nao/_wtest

# Confirm on-robot Python
python --version
```

Interpretation (verify BEFORE building anything — §7.0 gates the build on this):
- **`uname -m`**: `x86_64` → the 64-bit recipe below applies. `i686`/`i586` →
  **32-bit build required** (change `-march=silvermont` to an i686 target and add
  `-m32`); the 64-bit binary will not run.
- **`uname -r`**: a very old kernel (Linux 3.x) may reject a binary built with a
  modern toolchain → use the Aldebaran cross-toolchain (CTC) instead (§7.0).
- **`mount`**: if `/home/nao` is `noexec` or the rootfs is `ro`, find a
  writable+exec path or remount before deploying.
- **No `avx`, only `sse4_2`** → confirmed slow path → **start with a 0.5B model**.
- `MemAvailable` under ~1.5 GB → cap model size accordingly.
- `python` should report 2.7.x.

> **These are unverified assumptions until the above output comes back.** The
> recipe in §7 is written for a **64-bit x86** userland on a kernel new enough to
> run an Alpine-built static binary. If either is false, use the CTC path (§7.0).

---

## 4. Architecture (on-robot)

Keep all AI as standalone binaries; the Python 2.7 behavior only orchestrates.

```
NAOqi behavior (Python 2.7)  — thin orchestrator
   1. ALAudioRecorder      → capture.wav          (push-to-talk via tablet button)
   2. whisper.cpp binary   → transcript           (STT, added in a later phase)
   3. HTTP POST to llama-server on 127.0.0.1:8080 → reply text
   4. ALTextToSpeech.say(reply) + gestures/LEDs
   (loop for multi-turn; behavior holds conversation history)
```

- `llama-server` (from llama.cpp) serves an OpenAI-compatible endpoint on
  `localhost`. Python 2.7 calls it with plain `urllib2` — no extra packages.
- STT and TTS only exist on the **physical** robot.
- Run STT and LLM **sequentially** (never concurrently) to fit the RAM/CPU
  budget — push-to-talk makes this natural.

---

## 5. Model candidates (open-source, on-robot)

4-bit quantized (Q4_K_M), sub-2B, strong instruction-followers:

| Model | Size (Q4) | Notes |
|---|---|---|
| **Qwen2.5-0.5B-Instruct** | ~0.4 GB | Fastest; best starting point on this CPU |
| **Llama 3.2 1B Instruct** | ~0.8 GB | Good balance |
| **Qwen2.5-1.5B-Instruct** | ~1.0 GB | Best quality that still fits (tighter on RAM) |
| **SmolLM2 1.7B** | ~1.1 GB | Purpose-built for on-device |

Start at **0.5B**, measure, then try 1B/1.5B only if tokens/sec and free RAM
allow.

---

## 6. Phased plan

1. **Feasibility (this doc, §7).** Build static `llama.cpp`, run a 0.5B model on
   the actual robot, measure tokens/sec and RAM. Go/no-go for on-robot.
2. **Text loop (§8).** Type text → local LLM → `ALTextToSpeech.say()`. Validates
   persona quality with the fewest moving parts.
3. **On-device STT.** `whisper.cpp` tiny.en (or Vosk) + push-to-talk button.
4. **Memory + embodiment.** Multi-turn context, listening/thinking LED cues,
   gestures during speech.

---

## 7. STEP 1 — Feasibility test (detailed recipe)

Goal: prove a 0.5B model runs on Pepper at a usable speed. Done on your **dev
machine** (build) + the **robot** (run). Build machine needs **Docker**
(Docker Desktop on Windows works; run the commands from Git Bash or WSL).

### 7.0 Compatibility gate — do this FIRST (cheap probe before the big build)

Do not build llama.cpp until a trivial binary, compiled the same way, actually
runs on the robot. This costs ~2 minutes and de-risks arch/kernel/noexec all at
once.

```sh
# On the dev machine — build a static hello-world matching the §3 arch result.
# 64-bit example (Alpine); for i686, use an i686 toolchain / -m32 instead.
printf '#include <stdio.h>\nint main(){puts("hello from pepper");return 0;}\n' > hello.c
docker run --rm -v "$PWD":/out alpine:3.19 sh -c \
  'apk add --no-cache build-base >/dev/null && \
   gcc -static -march=silvermont -O2 hello.c -o /out/hello'
file hello                                   # expect "statically linked"

scp hello nao@<pepper-ip>:/home/nao/
ssh nao@<pepper-ip> 'chmod +x /home/nao/hello && /home/nao/hello'
```

- Prints `hello from pepper` → the toolchain, arch, kernel, and exec permissions
  are all good; proceed to §7.1.
- `Illegal instruction` → wrong `-march`/instruction set.
- `cannot execute binary file` → **wrong architecture** (32- vs 64-bit mismatch).
- `Permission denied` after `chmod +x` → `noexec` mount; pick another path.
- Fails despite `file` showing static → likely **kernel too old for the modern
  toolchain** → use the CTC path below.

### 7.0b The guaranteed-compatible alternative: Aldebaran CTC

If the static probe won't run (old kernel, or you want the officially-supported
route), build with Aldebaran's **cross-compilation toolchain (CTC) for NAOqi
2.5**, which targets the robot's exact libc/kernel/arch. It's more setup than the
Docker shortcut but is the known-good way to produce robot binaries. Point
llama.cpp's CMake at the CTC as the cross toolchain; keep the same AVX-off /
SSE4.2 CPU flags from §7.1.

### 7.1 Build a static llama.cpp (SSE4.2, no AVX) — 64-bit x86 path

> Assumes §7.0 passed on a 64-bit userland. For i686, swap the arch flags for an
> i686 target (`-m32`); if §7.0 failed, use the CTC (§7.0b) instead.

A fully static (musl) build avoids the robot's old glibc entirely. AVX is
disabled and the target is `silvermont` (the E3845 microarch).

```sh
# Run from an empty working dir on your dev machine (Git Bash / WSL).
docker run --rm -v "$PWD":/out alpine:3.19 sh -c '
set -e
apk add --no-cache build-base cmake git linux-headers
git clone --depth 1 https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DLLAMA_CURL=OFF \
  -DGGML_NATIVE=OFF \
  -DGGML_AVX=OFF -DGGML_AVX2=OFF -DGGML_AVX512=OFF -DGGML_FMA=OFF -DGGML_F16C=OFF \
  -DGGML_SSE42=ON \
  -DCMAKE_EXE_LINKER_FLAGS="-static" \
  -DCMAKE_C_FLAGS="-march=silvermont" \
  -DCMAKE_CXX_FLAGS="-march=silvermont"
cmake --build build -j"$(nproc)" --target llama-cli llama-bench llama-server
cp build/bin/llama-cli build/bin/llama-bench build/bin/llama-server /out/
'
```

> On PowerShell, replace `"$PWD"` with `"${PWD}"`.
> CPU-feature flag names occasionally change across llama.cpp versions — an
> unknown `-D` flag is warned-and-ignored, not fatal. `-march=silvermont` is the
> load-bearing part.

Confirm it's static:

```sh
file llama-cli    # expect: "statically linked"
```

**If the musl static build gives trouble** (rare, usually the server): fall back
to a glibc `-static` build inside `ubuntu:16.04` (old glibc matches the robot),
same CMake flags.

### 7.2 Get the model

```sh
# Verify the exact filename on the HF page before downloading.
curl -L -o qwen2.5-0.5b-instruct-q4_k_m.gguf \
  https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf
```

### 7.3 Deploy to Pepper

```sh
PEPPER_IP=<pepper-ip>
ssh nao@$PEPPER_IP 'mkdir -p /home/nao/llm'
scp llama-cli llama-bench llama-server qwen2.5-0.5b-instruct-q4_k_m.gguf nao@$PEPPER_IP:/home/nao/llm/
ssh nao@$PEPPER_IP 'chmod +x /home/nao/llm/llama-*'
```

### 7.4 Benchmark tokens/sec (on the robot)

```sh
cd /home/nao/llm
./llama-bench -m qwen2.5-0.5b-instruct-q4_k_m.gguf -t 4 -p 64 -n 64
```

Read the **`tg` (text-generation) tokens/sec** — that's what sets reply latency.
While it runs, in a second SSH session watch RAM: `watch -n1 free -m`.

### 7.5 Eyeball response quality

```sh
./llama-cli -m qwen2.5-0.5b-instruct-q4_k_m.gguf -t 4 -c 1024 -n 80 \
  -p "You are Pepper, a friendly robot. Reply in one short sentence.\nUser: Who are you?\nPepper:"
```

(Or `-cnv -sys "You are Pepper, a friendly robot..."` for interactive chat mode.)

### 7.6 Go / No-go criteria

| `tg` tokens/sec | Free RAM headroom | Decision |
|---|---|---|
| **≥ ~4–5** | comfortable | Proceed on-robot; optionally test 1B/1.5B |
| ~2–4 | ok | Usable with short replies + push-to-talk; stay at 0.5B |
| **< ~2** | tight | Move the LLM to a companion box (see §10 fallback) |

---

## 8. STEP 2 — Text conversation loop

### 8.1 Start the local server (on the robot)

```sh
cd /home/nao/llm
./llama-server -m qwen2.5-0.5b-instruct-q4_k_m.gguf \
  -t 4 -c 2048 --host 127.0.0.1 --port 8080 &
```

Keep `-c` (context) small to save RAM. Later, run this under a supervisor so it
starts with the robot.

### 8.2 Minimal Python 2.7 NAOqi behavior

Runs **on the robot**. Calls the local server's OpenAI-compatible endpoint (the
server applies the model's chat template for you) and speaks the reply.

```python
# -*- coding: utf-8 -*-
# pepper_llm_say.py  — run on the robot: python pepper_llm_say.py
import json
import urllib2
from naoqi import ALProxy

ROBOT_IP = "127.0.0.1"   # script runs on the robot
PORT = 9559
LLM_URL = "http://127.0.0.1:8080/v1/chat/completions"

SYSTEM = ("You are Pepper, a friendly humanoid robot. "
          "Answer in one or two short, natural sentences.")

def ask_llm(user_text):
    body = json.dumps({
        "messages": [
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": user_text},
        ],
        "max_tokens": 80,
        "temperature": 0.7,
    })
    req = urllib2.Request(LLM_URL, body, {"Content-Type": "application/json"})
    resp = urllib2.urlopen(req, timeout=60)   # blocks; fine for push-to-talk
    data = json.loads(resp.read())
    return data["choices"][0]["message"]["content"].strip()

def main():
    tts = ALProxy("ALTextToSpeech", ROBOT_IP, PORT)  # works on physical robot
    try:
        reply = ask_llm("Hello Pepper, who are you?")
    except Exception as e:
        print "LLM error:", e
        tts.say("Sorry, my brain is offline right now.")
        return
    print "LLM:", reply
    tts.say(reply)

if __name__ == "__main__":
    main()
```

Notes:
- Multi-turn later: keep a running `messages` list in the behavior and append
  each user/assistant turn (the server is stateless).
- `urllib2` ships with Python 2.7 — no installs needed on the robot.
- `ALTextToSpeech.say()` is a real-robot-only capability (dead on the virtual
  robot).

---

## 9. Speech-to-text (later phase)

The heavier half. Options, cleanest first:
- **Push-to-talk + sequential** processing (tablet button starts recording; STT
  runs, then the LLM runs — never at the same time).
- **Engine:** `whisper.cpp` **tiny.en** (best quality/size on-device) or
  **Vosk** (~40 MB, lighter, lower accuracy). Both run as standalone processes,
  so the Python 2.7 constraint doesn't bite.
- Capture on the robot with `ALAudioRecorder` → WAV → feed to the STT binary →
  transcript into the §8 loop.

---

## 10. Fallback (keeps it local + open-source)

If §7.6 shows on-robot inference is too slow, run the **same open-source model**
— or a better one like **Phi-3.5-mini** or **Qwen2.5-3B** — on a **companion
Linux box on the same network** (`llama-server` on that box, robot points at its
IP instead of `127.0.0.1`). Still fully self-hosted, offline, no cloud, no API
cost — just not literally on the Atom. Treat as the safety net, not the start.

---

## 11. Open questions / decisions

- **Hardware confirmation** — paste the §3 command output (especially the CPU
  `flags` line and `MemAvailable`) to finalize model size.
- **Persona scope** — greeter, topic guide, or open-ended companion? A narrow
  scope is the single biggest lever for making a sub-2B model feel non-robotic;
  it shapes the system prompt and model choice.
- **Interaction trigger** — tablet button (recommended, reliable on 2.5) vs.
  wake word (unreliable on this generation).

---

## 12. References

- NAOqi 2.5 docs root: `http://doc.aldebaran.com/2-5/`
- llama.cpp: `https://github.com/ggml-org/llama.cpp`
- whisper.cpp: `https://github.com/ggml-org/whisper.cpp`
- Qwen2.5 GGUF models: Hugging Face `Qwen/Qwen2.5-0.5B-Instruct-GGUF`
- Constraints: NAOqi 2.5.10.7 (Python 2.7), Pepper 1.x (Atom E3845, 4 GB, no AVX)
