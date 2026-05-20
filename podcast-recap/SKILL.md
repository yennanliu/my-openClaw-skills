---
name: podcast-recap
description: Transcribe and summarize any podcast episode or YouTube video. Given a URL (YouTube, RSS feed, or direct audio), it fetches the transcript, extracts key takeaways with timestamps, and saves a structured markdown note.
version: 1.0.0
metadata:
  openclaw:
    emoji: 🎙️
    homepage: https://github.com/openclaw/skills
    requires:
      bins: [yt-dlp, ffmpeg]
    anyBins: [whisper, python3]
    install:
      - kind: brew
        formula: yt-dlp
        bins: [yt-dlp]
      - kind: brew
        formula: ffmpeg
        bins: [ffmpeg]
    envVars:
      - name: PODCAST_NOTES_DIR
        required: false
        description: Directory where recap notes are saved. Defaults to ~/podcast-notes.
---

# podcast-recap

Summarize a podcast episode or YouTube video from a URL. Produces a structured markdown note with key takeaways, chapter summaries, and notable quotes — saved locally for future reference.

---

## Trigger

The user sends a message containing a URL and a request to recap or summarize it. Examples:
- "recap this podcast: https://..."
- "summarize this episode: https://..."
- "podcast-recap https://..."
- "take notes on https://..."

---

## Execution Steps

### Step 1 — Detect Input Type

Inspect the URL to determine its type:

- **YouTube**: URL contains `youtube.com/watch`, `youtu.be/`, or `youtube.com/shorts/`
- **RSS feed**: URL ends in `.xml`, `.rss`, contains `/feed`, or returns `application/rss+xml` or `application/atom+xml` on HEAD request
- **Direct audio**: URL ends in `.mp3`, `.m4a`, `.ogg`, `.wav`, `.opus`
- **Podcast episode page**: Any other URL — try to find a media file link embedded in the page

---

### Step 2 — Get the Transcript

#### 2A: YouTube URL

Use `yt-dlp` to extract auto-generated or manual captions:

```bash
yt-dlp \
  --skip-download \
  --write-auto-sub \
  --write-sub \
  --sub-lang en \
  --sub-format vtt \
  --output "/tmp/podcast-recap/%(title)s.%(ext)s" \
  "<URL>"
```

If captions are unavailable, fall back to downloading the audio and running Whisper (Step 2C).

Parse the `.vtt` file into plain text with timestamps:

```bash
# Strip VTT metadata and de-duplicate overlapping lines
grep -v '^WEBVTT\|^NOTE\|^[0-9].*-->' /tmp/podcast-recap/*.vtt \
  | sed '/^$/d' \
  | awk '!seen[$0]++' \
  > /tmp/podcast-recap/transcript.txt
```

Also capture episode metadata (title, channel, duration, upload date):

```bash
yt-dlp --dump-json --no-download "<URL>" > /tmp/podcast-recap/meta.json
```

Extract with jq:
```bash
jq -r '{title, uploader, upload_date, duration, description}' /tmp/podcast-recap/meta.json
```

#### 2B: RSS Feed URL

1. Fetch the feed XML:
   ```bash
   curl -sL "<URL>" -o /tmp/podcast-recap/feed.xml
   ```

2. Extract the latest episode's enclosure URL (the audio file):
   ```bash
   grep -oP '(?<=url=")[^"]+\.(mp3|m4a|ogg|opus)' /tmp/podcast-recap/feed.xml | head -1
   ```
   Or, if the user specified an episode number/title, find the matching `<item>` block.

3. Download the audio and proceed to Step 2C.

#### 2C: Direct Audio or Audio Fallback

Download the audio file:
```bash
curl -sL "<AUDIO_URL>" -o /tmp/podcast-recap/episode.mp3
```

Transcribe with Whisper (if installed):
```bash
whisper /tmp/podcast-recap/episode.mp3 \
  --model small \
  --output_format txt \
  --output_dir /tmp/podcast-recap/
```

If Whisper is not installed, instruct the user:
> "Whisper is not installed. Install it with: `pip install openai-whisper`. I can still summarize from the episode description if you'd like."
> Then proceed using only the episode title and description.

---

### Step 3 — Summarize

Read the full transcript from `/tmp/podcast-recap/transcript.txt`.

Apply the summarization template in `prompts/summarize.md`.

Generate the following sections:

1. **TL;DR** (2–3 sentences): The single most important insight from the episode.
2. **Key Takeaways** (5–8 bullets): Distilled, actionable or notable points. Use the speaker's own words where impactful.
3. **Chapter Summaries**: If the episode has chapters (from YouTube or timestamps in the transcript), produce a one-paragraph summary per chapter. Otherwise, divide into ~4 logical sections.
4. **Notable Quotes**: 2–4 verbatim quotes worth remembering, each with a brief note on why it matters.
5. **Topics & Tags**: 5–10 keywords or topics covered (for searchability).
6. **My Take** (optional): If the user has asked for opinion or framing, add a brief editorial note.

---

### Step 4 — Resolve Output Path

Determine the output directory:
```bash
NOTES_DIR="${PODCAST_NOTES_DIR:-$HOME/podcast-notes}"
mkdir -p "$NOTES_DIR"
```

Generate a filename from the episode title:
```bash
# Sanitize title: lowercase, replace spaces/special chars with hyphens
TITLE_SLUG=$(echo "<EPISODE_TITLE>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-\|-$//g')
DATE=$(date +%Y-%m-%d)
OUTPUT_FILE="$NOTES_DIR/${DATE}-${TITLE_SLUG}.md"
```

---

### Step 5 — Write the Note

Write the structured markdown note to `$OUTPUT_FILE` using the template below.

```markdown
# <Episode Title>

> **Source**: <URL>
> **Published**: <Upload date or feed date>
> **Show / Channel**: <Podcast name or YouTube channel>
> **Duration**: <HH:MM:SS>
> **Recap generated**: <today's date>

---

## TL;DR

<2–3 sentence summary>

---

## Key Takeaways

- <takeaway 1>
- <takeaway 2>
- <takeaway 3>
- <takeaway 4>
- <takeaway 5>

---

## Chapter Summaries

### <Chapter 1 Title or "Part 1">
<paragraph>

### <Chapter 2 Title or "Part 2">
<paragraph>

...

---

## Notable Quotes

> "<quote>"
— *<Speaker name or "Host">*, <approximate timestamp>

> "<quote>"
— *<Speaker name>*, <approximate timestamp>

---

## Topics & Tags

`<tag1>` `<tag2>` `<tag3>` `<tag4>` `<tag5>`

---

*Generated by podcast-recap skill*
```

---

### Step 6 — Respond to the User

Send a message confirming the recap was saved:

> 🎙️ **Podcast recap saved!**
>
> **"<Episode Title>"**
> Show: <Podcast / Channel> · Duration: <duration>
>
> **TL;DR**: <the TL;DR text>
>
> Saved to: `<OUTPUT_FILE>`
>
> Want me to read out the key takeaways or dive into any section?

---

## Error Handling

| Situation | Response |
|-----------|----------|
| `yt-dlp` not installed | Tell the user: "Run `brew install yt-dlp` to enable YouTube support." |
| No captions available and Whisper not installed | Summarize from title + description only; flag the limitation |
| RSS feed has no enclosure URL | Ask the user to provide the direct audio URL |
| Transcript is very short (<200 words) | Warn the user: "Transcript appears incomplete — summary may be limited." |
| Network error fetching URL | Report the error and suggest the user check the URL |
| Output directory not writable | Fall back to `~/Downloads/podcast-notes/` and inform the user |

---

## Notes

- Temporary files are written to `/tmp/podcast-recap/` and can be cleaned up after the note is saved with `rm -rf /tmp/podcast-recap/`.
- The skill works best with YouTube videos that have auto-captions enabled. For RSS feeds, Whisper provides high-quality transcripts for audio under ~60 minutes using the `small` model without GPU.
- If the episode is longer than 2 hours, use `--model tiny` for Whisper to keep transcription time reasonable.
- The `PODCAST_NOTES_DIR` env var allows routing notes to an Obsidian vault or any synced folder.
