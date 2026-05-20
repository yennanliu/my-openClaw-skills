# podcast-recap

An OpenClaw skill that transcribes and summarizes podcast episodes and YouTube videos, saving structured markdown notes locally.

## What It Does

1. Accepts a YouTube URL, podcast RSS feed, or direct audio file URL
2. Fetches the transcript (via yt-dlp captions or Whisper transcription)
3. Summarizes into TL;DR, key takeaways, chapter summaries, and notable quotes
4. Saves a markdown note to your notes directory

## Usage

Send any of these in chat:

```
recap this podcast: https://youtube.com/watch?v=...
summarize this episode: https://feeds.example.com/podcast.rss
podcast-recap https://example.com/episode.mp3
take notes on https://youtu.be/...
```

## Dependencies

| Tool | Required | Purpose | Install |
|------|----------|---------|---------|
| `yt-dlp` | Yes (for YouTube) | Download captions and metadata | `brew install yt-dlp` |
| `ffmpeg` | Yes (for audio) | Audio processing for Whisper | `brew install ffmpeg` |
| `whisper` | Optional | Local audio transcription | `pip install openai-whisper` |

## Configuration

Set `PODCAST_NOTES_DIR` in your environment to control where notes are saved:

```bash
# In your shell profile or OpenClaw env config
export PODCAST_NOTES_DIR="$HOME/Documents/Obsidian/Podcasts"
```

Default output directory: `~/podcast-notes/`

## Output Format

Notes are saved as `YYYY-MM-DD-episode-title.md` and contain:

- **TL;DR** — 2–3 sentence summary of the core insight
- **Key Takeaways** — 5–8 distilled bullets
- **Chapter Summaries** — one paragraph per section or chapter
- **Notable Quotes** — 2–4 verbatim quotes with context
- **Topics & Tags** — searchable keywords

## Supported Input Types

| Input | Transcript Method |
|-------|------------------|
| YouTube URL | yt-dlp auto/manual captions |
| YouTube (no captions) | yt-dlp audio → Whisper |
| RSS feed | Latest episode audio → Whisper |
| Direct `.mp3` / `.m4a` | Whisper |
| Episode page URL | Finds embedded audio → Whisper |

## File Structure

```
podcast-recap/
├── SKILL.md           # Skill definition and execution instructions
├── README.md          # This file
└── prompts/
    └── summarize.md   # Summarization prompt template
```
