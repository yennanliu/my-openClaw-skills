# youtube-notes

An OpenClaw skill that takes structured, timestamp-linked notes on any YouTube video — lectures, tutorials, talks, and interviews.

## What It Does

1. Accepts a YouTube URL
2. Downloads captions via `yt-dlp` (manual or auto-generated)
3. Extracts key points, concepts, steps, or quotes — each linked to a clickable timestamp
4. Auto-detects the video type (lecture / tutorial / talk) and adjusts note structure
5. Saves a navigable markdown note locally

## Usage

Send any of these in chat:

```
notes on https://youtube.com/watch?v=dQw4w9WgXcQ
youtube-notes https://youtu.be/dQw4w9WgXcQ
take notes on this video: https://youtube.com/watch?v=...
summarize this youtube video: https://...
what does this video cover? https://youtu.be/...
```

## Note Modes

The skill auto-detects the video type and tailors the output:

| Mode | Triggered by | Extra sections |
|------|-------------|----------------|
| `lecture` | "lecture", "explained", "course", long video with chapters | Concept Index, Prerequisites |
| `tutorial` | "how to", "build", "install", "setup" | Steps Summary, Commands & Code, Prerequisites |
| `talk` | Conference talk, TED, opinion, interview | Central Argument, Notable Quotes, Counterarguments |

Override auto-detection by setting `YOUTUBE_NOTES_MODE=lecture` (or `tutorial` / `talk`).

## Dependencies

| Tool | Required | Purpose | Install |
|------|----------|---------|---------|
| `yt-dlp` | Yes | Download captions and video metadata | `brew install yt-dlp` |
| `jq` | Yes | Parse video metadata JSON | `brew install jq` |
| `python3` | Yes | Parse VTT caption files | Pre-installed on macOS |

## Configuration

```bash
# Where notes are saved (default: ~/youtube-notes)
export YOUTUBE_NOTES_DIR="$HOME/Documents/Obsidian/YouTube"

# Force a note mode instead of auto-detecting
export YOUTUBE_NOTES_MODE="lecture"   # lecture | tutorial | talk
```

## Output Format

Notes are saved as `YYYY-MM-DD-video-title.md` and contain:

- **Overview** — 3–4 sentence summary of the video
- **Key Points** — 8–15 bullets each with a clickable timestamp link
- **Mode-specific sections** — concepts, steps/code, or argument + quotes
- **Resources Mentioned** — books, tools, repos, papers named in the video
- **Chapters table** — if the video has chapters

Every timestamp links directly to that moment in the video:
`[[00:14:22]](https://youtube.com/watch?v=ID&t=862)`

## Comparison with `podcast-recap`

| | `youtube-notes` | `podcast-recap` |
|--|--|--|
| Source | YouTube only | YouTube, RSS feeds, audio files |
| Focus | Timestamp-linked navigation | Takeaways and quotes |
| Transcription | yt-dlp captions only | yt-dlp + Whisper fallback |
| Best for | Lectures, tutorials, talks | Podcast episodes, long interviews |

## File Structure

```
youtube-notes/
├── SKILL.md           # Skill definition and execution instructions
├── README.md          # This file
└── prompts/
    └── extract.md     # Extraction prompt template and mode reference
```
