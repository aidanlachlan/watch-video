---
name: watch-video
description: Turn one video link (YouTube, Instagram, TikTok, or Facebook) into a timestamped transcript and a short summary so Claude can discuss the video. Use this whenever the user pastes a single video link from any of those platforms and wants Claude to watch, read, summarize, take notes on, or follow along with it, even if they don't say "transcript". For deciding which of several videos to watch, use the video-triage skill. For analyzing a creator's style across several videos, use the creator-video-analysis skill.
---

# Watch Video

Goal: give Claude the full spoken content of ONE video, fast, with timestamps, so it can answer questions and follow along with the user.

## Rules

- One video per run. If the user gives several links, ask which one first.
- Never use browser cookies or log in to any platform (`--no-cookies --no-cookies-from-browser`). Some Instagram and Facebook posts are login-walled. If one fails, tell the user and stop.
- Captions first. Only download audio and run whisper if no captions exist or they fail the quality check in Step 2. Missing captions are common on Instagram, TikTok, and Facebook. Whisper is slow on older machines.
- If the transcript is nearly empty (music-only or on-screen text), grab frames (Step 4) without asking, since the visuals are the content.
- Do not download the full video unless the user asks for screenshots or frames.
- Save everything in `./videos/<platform>-<video-id>/` inside the current folder.

## Steps

### 1. Check the link and get metadata

```bash
yt-dlp --no-cookies --no-cookies-from-browser --skip-download \
  --write-info-json -o "videos/%(extractor_key)s-%(id)s/info" "<URL>"
```

Read `info.info.json` for title, channel, duration, upload date, and chapters (if any).

### 2. Try captions (fast path)

```bash
yt-dlp --no-cookies --no-cookies-from-browser --skip-download \
  --write-subs --write-auto-subs --sub-langs "en.*,en" --sub-format vtt \
  -o "videos/%(extractor_key)s-%(id)s/captions" "<URL>"
```

Prefer manual subtitles over auto-generated ones if both download.

Convert the VTT into `transcript.md`:
- One line per caption block, prefixed with `[mm:ss]`.
- Remove duplicate rolling lines (auto-captions repeat text across blocks).
- Strip VTT tags like `<c>` and inline timestamps.

**Caption quality check.** Before trusting captions, look for these signs they're bad:
- Very sparse for the runtime: under about 60 words per minute in a video where someone is talking
- Lots of `[Music]`, `[Applause]`, or gaps where speech clearly continues
- Garbled or nonsense phrases, or key names and technical terms mangled throughout
- Captions in a different language than the one spoken (auto-translated tracks)

Manual subtitles almost always pass. If auto-captions show two or more of these signs, treat them as missing and fall back to whisper. Tell the user captions were replaced and why.

### 3. Fallback: transcribe audio (if step 2 found no usable captions)

```bash
yt-dlp --no-cookies --no-cookies-from-browser -x --audio-format wav \
  -o "videos/%(extractor_key)s-%(id)s/audio.%(ext)s" "<URL>"
ffmpeg -y -i videos/<id>/audio.wav -ar 16000 -ac 1 -c:a pcm_s16le videos/<id>/audio16k.wav
whisper-cli -m ~/src/whisper.cpp/models/ggml-small.en.bin -f videos/<id>/audio16k.wav \
  -otxt -of videos/<id>/whisper
```

If `~/src/whisper.cpp/models/ggml-small.en.bin` is missing, tell the user and ask before downloading it (about 488 MB). Convert the whisper output into `transcript.md` with `[mm:ss]` timestamps.

### 4. Frames (if the user asks for visuals, or the transcript is under about 30 words)

Download the video at 720p or lower, then grab frames (every 3 seconds for videos under a minute, every 10 seconds otherwise) and view them, reading any on-screen text:

```bash
ffmpeg -y -i videos/<id>/video.mp4 -vf "fps=1/10,scale=960:-2" videos/<id>/frames/%03d.jpg
```

### 5. Write `summary.md`

Keep it short and scannable:
- **Video:** title, channel, length, link
- **Big idea:** one or two sentences
- **Sections:** use the video's chapters if it has them, otherwise split by topic, each with a start timestamp and a one-line summary
- **Action steps:** concrete things the viewer is told to do, in order, with timestamps
- **Tools or products mentioned**

Mark anything unclear in the captions as uncertain instead of guessing.

### 6. Report back

Tell the user where the files are, show `summary.md`, and say whether the content came from captions, whisper, or frames. Then answer questions using `transcript.md`, citing timestamps like `[03:45]`.

If the user is following along with the video, track the timestamp they say they're at and avoid summarizing ahead of it unless they ask.
