---
name: remotion-voice-video
description: Produce a finished MP4 video with two agents working together. The Remotion agent builds motion graphics in code (React, Remotion 4), and the voice agent generates a narration track with a local text-to-speech model (Qwen3-TTS) that drives the timing. Use when the user wants a video made from a brief or a script, asks for motion graphics or a promo or explainer video, wants a voiceover added, needs a video rendered to a file, or wants an existing video file restyled, retimed or given narration. Works on Windows, macOS and Linux.
version: 1.1.0
---

# Remotion + voice agent: video file out

This skill runs two agents on one video:

- **Remotion agent:** designs the scenes, animation, typography and sound placement in code, then renders the video to a file.
- **Voice agent:** writes the narration audio from the script with a local text-to-speech model, and reports the exact length of each line. The Remotion agent uses those lengths to time everything.

The skill contains no personal project files. Everything is generated from the user's brief.

---

## Part 1. One-time setup

### 1.1 Requirements

| Item | Windows | macOS | Linux |
| --- | --- | --- | --- |
| Node.js 18+ and npm | nodejs.org installer | `brew install node` | your package manager, or nodejs.org |
| Python 3.12 | python.org or `winget install Python.Python.3.12` | `brew install python@3.12` | `sudo apt install python3.12 python3.12-venv` (or conda) |
| FFmpeg | `winget install Gyan.FFmpeg` | `brew install ffmpeg` | `sudo apt install ffmpeg` |
| SoX (audio, needed by Qwen3-TTS) | `winget install ChrisBagwell.SoX` | `brew install sox` | `sudo apt install sox libsox-fmt-all` |
| GPU for speed | NVIDIA, about 8 GB VRAM | optional (CPU works, much slower) | NVIDIA, about 8 GB VRAM (CPU works, much slower) |

Check each tool after installing:

```bash
node -v && npm -v && python3.12 --version && ffmpeg -version | head -1 && sox --version
```

On Windows use `python` instead of `python3.12` if that is the command that works.

**Platform notes**
- **Linux:** needs glibc 2.35 or newer. Alpine and NixOS are not supported by Remotion.
- **macOS:** needs macOS 15 or newer for Remotion. Apple Silicon works for Remotion. For Qwen3-TTS, use the CPU setting in Part 1.3. This is slower but works.
- **Windows:** x64 only.

### 1.2 Install Qwen3-TTS (the voice)

Create a dedicated Python environment, so the voice dependencies stay separate from everything else.

**Windows (PowerShell):**
```powershell
py -3.12 -m venv qwen-venv
.\qwen-venv\Scripts\Activate.ps1
python -m pip install -U pip
pip install -U qwen-tts
```

**macOS and Linux (bash/zsh):**
```bash
python3.12 -m venv ~/qwen-venv
source ~/qwen-venv/bin/activate
python -m pip install -U pip
pip install -U qwen-tts
```

Conda works too: `conda create -n qwen3-tts python=3.12 -y`, then `conda activate qwen3-tts`, then `pip install -U qwen-tts`.

Optional, Linux with NVIDIA only, for lower GPU memory use:
```bash
pip install -U flash-attn --no-build-isolation
```
Do not install it on Mac or Windows. The model runs without it.

### 1.3 Check that the voice works

The model downloads on first use. It is several GB, so allow time.

Quick test with the bundled web demo:
```bash
qwen-tts-demo
```
Open the address it prints, type a sentence, and check that you get audio.

Minimal Python test. Save it anywhere, then run it with the environment's `python`:

```python
import torch, soundfile as sf
from qwen_tts import Qwen3TTSModel

# GPU (NVIDIA): device "cuda:0", dtype torch.bfloat16
# CPU (Mac, or no GPU): device "cpu", dtype torch.float32
device = "cuda:0" if torch.cuda.is_available() else "cpu"
dtype = torch.bfloat16 if device.startswith("cuda") else torch.float32

model = Qwen3TTSModel.from_pretrained(
    "Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice",   # use "Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice" on low-memory machines
    device_map=device,
    dtype=dtype,
    attn_implementation="sdpa",
)

wavs, sr = model.generate_custom_voice(
    text="Hello, and welcome to our short demo.",
    language="English",
    speaker="Ryan",
    instruct="Calm, warm and confident narrator.",
)
sf.write("voice-test.wav", wavs[0], sr)
print("wrote voice-test.wav at", sr, "Hz")
```

**Choosing the model**
- `1.7B-CustomVoice`: best quality. Needs about 8 GB of GPU memory, or a lot of RAM on CPU.
- `0.6B-CustomVoice`: smaller and faster. Use it when the machine has less memory.

**Languages:** English, German, French, Spanish, Italian, Portuguese, Russian, Japanese, Korean and Chinese are listed as supported. Test any language you need before the project starts.

**Speakers:** the model includes named voices such as Ryan, Aiden, Serena and Vivian. Pick one per role (narrator, presenter, customer) and keep it the same across the whole video.

**Run one voice job at a time.** Two at once on one GPU slow both down badly, and can run out of memory.

### 1.4 Set up the Remotion project

Create the project in an empty folder:

```bash
npx create-video@latest --yes --blank video-project
cd video-project
npm install
```

Add the Remotion packages. Always use `npx remotion add`, so every package stays on the same version:

```bash
npx remotion add @remotion/transitions @remotion/media @remotion/shapes @remotion/google-fonts
```

Optional, for Remotion's official documentation skills:

```bash
npx skills add remotion-dev/skills
```

Download the headless browser that Remotion uses to render frames:

```bash
npx remotion browser ensure
```

---

## Part 2. Inputs to collect from the user

Ask only what you cannot infer:

1. **Brief:** the message, the audience and the call to action.
2. **Script:** the narration in full. If none exists, write one from the brief and get approval first.
3. **Format:** 16:9 (1920×1080) or 9:16 (1080×1920), frame rate (30 fps default), and target length in seconds.
4. **Language:** the narration language and the on-screen language. Confirm the words to avoid.
5. **Brand:** colours (hex), fonts, logo file (if any), and any forbidden words.
6. **Optional source video:** if the user supplied a video file, ask whether to use it as a visual, a reference or the basis for a restyle.

---

## Part 3. Workflow

### Step 1: Plan the timeline
Split the script into scenes. Every scene has a start and an end in seconds. Leave the timing open until the voice is generated.

### Step 2: Voice agent
Generate one audio file per narration line, using the setup from Part 1.3. Name each file after its line id, for example `line-01.wav`.

- Choose the speaker and language for each role. Generate one line first and listen to it.
- Measure each file's duration with `ffprobe`, and keep the durations in a manifest:

```json
{
  "lines": [
    { "id": "line-01", "text": "…", "file": "voice/line-01.wav", "start": 0.4, "duration": 2.1, "end": 2.5 }
  ]
}
```

- Set `start` for each line. Add a short gap between lines, usually 0.25 s.
- If a line runs too long for its slot, shorten the text rather than speeding up the audio.

### Step 3: Remotion agent
Build the video from the timeline and the manifest.

- Put each scene in its own component. Wrap each one in a `<Sequence>` with `premountFor={fps}`.
- Drive all motion from the frame: `useCurrentFrame()`, `interpolate(..., { extrapolateLeft: "clamp", extrapolateRight: "clamp" })` and `spring()`. Do not use CSS transitions or animations, and do not use `Math.random()`. Use `random("seed")` instead.
- Write timing in seconds, converted with `Math.round(seconds * fps)`.
- Sync visuals to the voice: show a subtitle or a highlight when its line starts, using the manifest times.
- Keep the scene text in one object per language, so that changing the language is one change.
- Add the audio with `<Audio>` from `@remotion/media`, and load files with `staticFile()` from `public/`.
- Register each video as a `<Composition>` with `id`, `fps`, `width`, `height` and `durationInFrames`.

### Step 4: Preview
```bash
npx remotion studio --no-open --port=3002
```
Open `http://localhost:3002/<composition-id>` and check the timing, the text and the transitions. Use a port that is free on the machine.

### Step 5: Quality check before rendering
- Run the type check and linter (`npx tsc`, `npx eslint src`).
- Render a few still frames, at each scene start and at each transition, and look at them:
  ```bash
  npx remotion still <composition-id> out/check-<time>.jpg --frame=<frame>
  ```
- Check for text cut off, text overlapping, wrong numbers or dates, and any forbidden word.

### Step 6: Render the file
```bash
npx remotion render <composition-id> out/<name>-raw.mp4
```
Render to a `-raw` name first. Then master the audio (Step 7).

### Step 7: Master the audio
Target −14 LUFS integrated with a −2 dBTP true-peak ceiling, which suits web and social platforms. Use two-pass loudness normalization with the video stream copied.

Pass 1, measure:
```bash
ffmpeg -hide_banner -i out/<name>-raw.mp4 -af "loudnorm=I=-14:TP=-2:LRA=11:print_format=json" -f null -
```

Read `input_i`, `input_tp`, `input_lra`, `input_thresh` and `target_offset` from that output. Pass 2, apply them:

```bash
ffmpeg -y -i out/<name>-raw.mp4 -c:v copy -af "loudnorm=I=-14:TP=-2:LRA=11:measured_I=<input_i>:measured_TP=<input_tp>:measured_LRA=<input_lra>:measured_thresh=<input_thresh>:offset=<target_offset>:linear=true,aresample=48000" -c:a aac -b:a 320k -shortest -movflags +faststart out/<name>-final.mp4
```

On Windows PowerShell, put each command on one line and drop the `\` line breaks.

Confirm the result:
```bash
ffmpeg -hide_banner -i out/<name>-final.mp4 -af ebur128=peak=true -f null -
```
Integrated loudness should be about −14 LUFS, and the peak should be at or below −1.5 dBFS.

### Step 8: Verify and deliver
- Check the final file with `ffprobe`: width, height, frame rate, duration (within 0.1 s of the target), H.264 video and AAC audio.
- Send the `-final.mp4` file to the user. Include the narration script as a text file if the user needs it for review.

---

## Part 4. Rules

1. **Forbidden words:** never write or speak any word the user bans. Search the script and all generated text before rendering.
2. **Facts:** do not invent prices, claims, dates or contact details. Use only what the user provides.
3. **Consent:** only use people's names, voices or likenesses the user has permission to use. Tell the user that the voices are synthetic.
4. **Outputs:** keep raw renders and final files separate. Never render directly over a final file.
5. **Credentials:** do not put keys, passwords or personal data in code or in the output.
6. **One voice job at a time** on a single machine.
7. **Voice quality:** the voices are not all native speakers of every language. Listen to the German, French or other non-English narration before it is published, and state the limits to the user.

---

## Part 5. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `qwen_tts` not found | Wrong environment is active | Activate the environment from Part 1.2, then `pip install -U qwen-tts` |
| Error about `sox` | System SoX is missing or not on PATH | Install SoX (Part 1.1), open a new terminal |
| Very slow voice generation | Running on CPU | Use the 0.6B model, or a machine with an NVIDIA GPU |
| Out of GPU memory | 1.7B model too large, or two jobs running | Use the 0.6B model, and run one job at a time |
| Model download stalls | Slow or blocked connection | Retry, or download the model files first with `huggingface-cli download` |
| Studio says the port is in use | Another Studio is running | Use another `--port` |
| Remotion versions differ | A package was installed with plain npm | `npx remotion versions`, then `npx remotion add` |
| Audio is silent in the preview | Audio file is missing or the path is wrong | Check `public/` and use `staticFile()` |
| Render stalls on the first run | Headless browser is still downloading | Run `npx remotion browser ensure` first |
| Loudness is off, or the final is "dynamic" | The peaks are too high for linear gain | Lower the loudest sound effects, or the voice gain |
| Narration runs long | The text is too long for the slot | Shorten the text and regenerate that line |
