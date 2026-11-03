# watch-video

A Claude Code skill. Turns one video link (YouTube, Instagram, TikTok, or Facebook) into a timestamped transcript and summary, so Claude can discuss it or follow along with you.

## Install

```bash
git clone https://github.com/aidanlachlan/watch-video.git ~/.claude/skills/watch-video
```

Or paste this repo's link into Claude Code and ask it to install the skill.

Update later with:

```bash
cd ~/.claude/skills/watch-video && git pull
```

## Requirements

- yt-dlp and Deno 2.0+ (captions and metadata)
- FFmpeg and whisper.cpp (only for videos without usable captions)
- Whisper model `ggml-small.en.bin` (about 488 MB, downloaded on first use with permission)
