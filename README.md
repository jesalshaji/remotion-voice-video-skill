# Remotion Voice Video

A Claude Code skill that turns a brief or a script into a finished MP4 with narration.

Two agents work together:

- **Voice agent:** reads the script aloud with a local text-to-speech model ([Qwen3-TTS](https://huggingface.co/Qwen)), one audio file per line, and measures how long each line runs.
- **Remotion agent:** builds the motion graphics in code with [Remotion 4](https://www.remotion.dev/) (React), times every scene to the narration, renders the video, and masters the audio to −14 LUFS for web and social.

The voice runs on your own machine. No cloud TTS account or API key is needed.

## What you can ask for

- "Make a 30-second explainer video about our new app, 16:9, with a calm male narrator."
- "Turn this script into a vertical 9:16 promo with captions."
- "Add a voiceover to this video and retime the scenes to fit."
- "Render the same video in English and German."

Claude asks for anything it can't work out on its own: the script, the format, brand colours and fonts, and any words to avoid.

## Requirements

| | |
| --- | --- |
| Node.js 18+ and npm | for Remotion |
| Python 3.12 | for Qwen3-TTS |
| FFmpeg | for rendering and audio mastering |
| SoX | needed by Qwen3-TTS |
| NVIDIA GPU, about 8 GB VRAM | recommended. CPU works, much slower |

It works on Windows (x64), macOS 15+, and Linux (glibc 2.35+). [SKILL.md](SKILL.md) has the install commands for each system.

## Install the skill

Clone this repo into your Claude Code skills folder.

**macOS and Linux:**
```bash
git clone https://github.com/jesalshaji/remotion-voice-video-skill.git ~/.claude/skills/remotion-voice-video
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/jesalshaji/remotion-voice-video-skill.git "$env:USERPROFILE\.claude\skills\remotion-voice-video"
```

To use it in a single project only, clone it into that project's `.claude/skills/remotion-voice-video` folder instead.

Then restart Claude Code. The skill loads on its own when you ask for a video, or you can call it by name with `/remotion-voice-video`.

## First run

Claude goes through the one-time setup in [SKILL.md](SKILL.md), Part 1, with you:

1. Install the tools listed above.
2. Create a Python environment and install Qwen3-TTS. The model downloads on first use and is several GB.
3. Create a Remotion project and download its headless browser.

Later videos reuse this setup.

## How a video gets made

1. **Plan:** split the script into scenes.
2. **Voice:** generate each narration line and record its length in a timing manifest.
3. **Build:** write the Remotion scenes and sync captions and motion to the manifest.
4. **Preview:** check the video in Remotion Studio.
5. **Check:** run the type check and lint, then look at still frames at every scene change.
6. **Render:** write a raw MP4.
7. **Master:** normalize the loudness to −14 LUFS, −2 dBTP true peak.
8. **Deliver:** check the final file's size, frame rate and duration, then hand it over.

You get `out/<name>-final.mp4`, and the raw render is kept next to it.

## Good to know

- **Synthetic voices.** All narration is AI-generated. Tell your audience where needed, and only use names or likenesses you have permission for.
- **Non-English narration.** The voices sound most natural in English. Listen to German, French and other languages before you publish.
- **No invented facts.** The skill uses only the prices, dates and claims you give it.
- **One voice job at a time.** Running two on one GPU slows both down and can run out of memory.

[SKILL.md](SKILL.md), Part 5, covers troubleshooting.

## Files

| File | Purpose |
| --- | --- |
| [SKILL.md](SKILL.md) | The skill: setup, workflow, rules and troubleshooting |
| README.md | This page |
