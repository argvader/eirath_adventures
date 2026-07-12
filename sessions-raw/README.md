# Raw Session Materials

Drop the inputs for each session into a dated folder here, then run the
summarizer (see `../SESSION_SUMMARIZER.md`). The summarizer reads the **newest**
date folder as the latest session.

```
sessions-raw/
  2026-06-04/
    transcript.md      # full session transcript (Deepgram diarized output)
    dm-notes.md        # GM session notes and prep (optional)
    <player>-notes.md  # individual player notes, e.g. braedon-notes.md (optional)
```

These files are **inputs only** — they are not published to the site (only
`docs/` is built by MkDocs).
