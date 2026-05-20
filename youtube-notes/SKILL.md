---
name: youtube-notes
description: Take structured notes on any YouTube video. Fetches captions via yt-dlp, extracts key points with clickable timestamps, identifies concepts, action items, and resources mentioned — saved as a markdown note.
version: 1.0.0
metadata:
  openclaw:
    emoji: 📝
    requires:
      bins: [yt-dlp, jq]
    install:
      - kind: brew
        formula: yt-dlp
        bins: [yt-dlp]
      - kind: brew
        formula: jq
        bins: [jq]
    envVars:
      - name: YOUTUBE_NOTES_DIR
        required: false
        description: Directory where notes are saved. Defaults to ~/youtube-notes.
      - name: YOUTUBE_NOTES_MODE
        required: false
        description: "Note style: 'lecture' (concepts + timestamps), 'tutorial' (steps + commands), 'talk' (arguments + quotes). Defaults to auto-detect."
---

# youtube-notes

Take structured notes on any YouTube video. Given a URL, it downloads captions, extracts key points linked to timestamps, and saves a markdown note you can navigate like a table of contents.

---

## Trigger

User sends a YouTube URL with a note-taking intent. Examples:
- `notes on https://youtube.com/watch?v=...`
- `youtube-notes https://youtu.be/...`
- `take notes on this video: https://...`
- `summarize this youtube video: https://...`
- `what does this video cover? https://...`

Capture the URL from the message. Validate it is a YouTube URL (`youtube.com/watch`, `youtu.be/`, `youtube.com/shorts/`, or `youtube.com/live/`).

---

## Execution Steps

### Step 1 — Fetch Video Metadata

```bash
mkdir -p /tmp/youtube-notes
yt-dlp --dump-json --no-download "<URL>" > /tmp/youtube-notes/meta.json
```

Extract fields needed for the note:

```bash
jq -r '{
  id: .id,
  title: .title,
  channel: .uploader,
  duration: .duration,
  upload_date: .upload_date,
  description: (.description | .[0:500]),
  chapters: .chapters,
  tags: .tags[0:10],
  view_count: .view_count
}' /tmp/youtube-notes/meta.json
```

Store `id` — it is needed to build clickable timestamp URLs in the format:
`https://www.youtube.com/watch?v=<id>&t=<seconds>`

---

### Step 2 — Download Captions

Prefer manual captions over auto-generated; prefer English.

```bash
yt-dlp \
  --skip-download \
  --write-sub \
  --write-auto-sub \
  --sub-lang "en,en-orig,en-US" \
  --sub-format "vtt" \
  --output "/tmp/youtube-notes/%(id)s.%(ext)s" \
  "<URL>"
```

Locate the downloaded `.vtt` file:
```bash
VTT_FILE=$(ls /tmp/youtube-notes/*.vtt 2>/dev/null | head -1)
```

If no `.vtt` file was produced, captions are unavailable. Inform the user:
> "This video has no available captions. I'll generate notes from the title, description, and chapter markers only."
> Proceed to Step 4 using only metadata.

---

### Step 3 — Parse Captions into Timestamped Segments

Convert the VTT file into a clean timestamped transcript. Each line: `[HH:MM:SS] text`.

```bash
python3 - <<'EOF'
import re, sys

vtt_path = "/tmp/youtube-notes/transcript.txt"
with open(sys.argv[1]) as f:
    content = f.read()

# Extract timestamp + cue text blocks
blocks = re.split(r'\n\n+', content)
seen = set()
lines = []

for block in blocks:
    rows = block.strip().splitlines()
    # Find the timestamp line
    ts_line = next((r for r in rows if '-->' in r), None)
    if not ts_line:
        continue
    start = ts_line.split('-->')[0].strip()
    # Convert HH:MM:SS.mmm → HH:MM:SS
    start = re.sub(r'\.\d+$', '', start)
    text_rows = [r for r in rows if '-->' not in r and not r.strip().isdigit() and r.strip()]
    text = ' '.join(text_rows).strip()
    if text and text not in seen:
        seen.add(text)
        lines.append(f"[{start}] {text}")

with open(vtt_path, 'w') as f:
    f.write('\n'.join(lines))
print(f"Wrote {len(lines)} lines to {vtt_path}")
EOF
python3 /tmp/youtube-notes/parse_vtt.py "$VTT_FILE"
```

The result at `/tmp/youtube-notes/transcript.txt` is the input for summarization.

---

### Step 4 — Detect Note Mode

Use `YOUTUBE_NOTES_MODE` if set. Otherwise auto-detect from metadata:

| Signal | Mode |
|--------|------|
| Title contains "how to", "tutorial", "step by step", "build", "install", "setup" | `tutorial` |
| Title contains "lecture", "course", "explained", "introduction to", "deep dive" | `lecture` |
| Channel is a conference or event (TED, GOTO, Strange Loop, etc.) | `talk` |
| Duration > 45 min and has chapters | `lecture` |
| Default | `talk` |

Each mode shapes the output sections (see Step 5).

---

### Step 5 — Generate Notes

Apply the prompt template from `prompts/extract.md` with the transcript, metadata, video ID, and detected mode.

Produce the following sections based on mode:

#### All modes include:
- **Overview** (3–4 sentences): What the video is about, who it's for, and the core argument or outcome.
- **Key Points with Timestamps**: Each point is a concise insight or claim, linked to where it appears in the video.
- **Resources Mentioned**: URLs, books, tools, papers, or people referenced in the video.

#### Mode: `lecture`
Additional sections:
- **Concept Index**: Each major concept introduced, with a one-line definition and its timestamp.
- **Prerequisites**: What prior knowledge is assumed.

#### Mode: `tutorial`
Additional sections:
- **Steps Summary**: Numbered list of the main steps demonstrated.
- **Commands & Code**: Any shell commands, code snippets, or config values shown. Include the timestamp where each appeared.
- **Prerequisites**: Tools or setup required before following along.

#### Mode: `talk`
Additional sections:
- **Central Argument**: The speaker's main thesis in 1–2 sentences.
- **Notable Quotes**: 2–4 verbatim quotes with timestamps.
- **Counterarguments Addressed**: Points the speaker anticipated or rebutted.

---

### Step 6 — Build Timestamp Links

For every timestamp `[HH:MM:SS]` in the output, convert to a YouTube deep-link:

```
HH:MM:SS → total seconds
link = https://www.youtube.com/watch?v=<VIDEO_ID>&t=<SECONDS>
```

Format in markdown as: `[[HH:MM:SS]](link)` so each timestamp is clickable.

Python helper:
```python
def ts_to_seconds(ts):
    parts = ts.split(':')
    if len(parts) == 3:
        h, m, s = parts
        return int(h)*3600 + int(m)*60 + int(s)
    elif len(parts) == 2:
        m, s = parts
        return int(m)*60 + int(s)
    return 0
```

---

### Step 7 — Write the Note

Resolve output path:
```bash
NOTES_DIR="${YOUTUBE_NOTES_DIR:-$HOME/youtube-notes}"
mkdir -p "$NOTES_DIR"
DATE=$(date +%Y-%m-%d)
TITLE_SLUG=$(echo "<TITLE>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-\|-$//g' | cut -c1-60)
OUTPUT_FILE="$NOTES_DIR/${DATE}-${TITLE_SLUG}.md"
```

Write the note using the template below:

---

```markdown
# <Title>

> **URL**: [Watch on YouTube](<URL>)
> **Channel**: <channel>
> **Duration**: <HH:MM:SS>
> **Published**: <YYYY-MM-DD>
> **Mode**: <lecture | tutorial | talk>
> **Notes taken**: <today's date>

---

## Overview

<3–4 sentence overview>

---

## Key Points

- [[00:01:23]](https://youtube.com/watch?v=ID&t=83) <key point>
- [[00:05:47]](https://youtube.com/watch?v=ID&t=347) <key point>
- [[00:12:10]](https://youtube.com/watch?v=ID&t=730) <key point>
...

---

## <Mode-specific section(s)>

...

---

## Resources Mentioned

- [<name>](<url or search term>) — <brief context>
...

---

## Chapters

| Chapter | Start | Link |
|---------|-------|------|
| <Chapter title> | 00:00:00 | [[▶]](https://youtube.com/watch?v=ID&t=0) |
...

*(section omitted if video has no chapters)*

---

*Generated by youtube-notes skill · [youtube-notes](https://github.com/openclaw/skills)*
```

---

### Step 8 — Respond to the User

Send a confirmation message:

> 📝 **Notes saved!**
>
> **"<Title>"**
> <Channel> · <duration> · Mode: `<mode>`
>
> **Overview**: <first sentence of overview>
>
> <N> key points with timestamps · Saved to: `<OUTPUT_FILE>`
>
> Want me to read out the key points, jump to a specific section, or search the notes?

---

## Error Handling

| Situation | Action |
|-----------|--------|
| Not a YouTube URL | Reply: "This skill only supports YouTube URLs. For other video sources, try `podcast-recap`." |
| `yt-dlp` not installed | Reply: "Install yt-dlp first: `brew install yt-dlp`" |
| No captions available | Use title + description + chapters only; flag in note with ⚠️ |
| Video is private or age-restricted | Report the yt-dlp error; suggest the user check access |
| Video is a live stream (ongoing) | Reply: "Live streams cannot be noted until after broadcast ends." |
| Transcript is under 100 words | Warn: "Transcript is very short — notes may be incomplete." |
| Output directory not writable | Fall back to `~/Downloads/youtube-notes/`; inform user |

---

## Cleanup

After writing the note, remove temp files:
```bash
rm -rf /tmp/youtube-notes/
```
