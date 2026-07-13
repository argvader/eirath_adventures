---
name: translate-deepgram
description: >-
  Turn a session.deepgram.json into the transcript.md the Eirath Adventures summarizer
  expects, applying the speaker map from sessions-raw/<DATE>/speaker-map.json. Use when
  the user says "translate deepgram", "convert the deepgram json", "make the transcript",
  or similar. Requires a speaker map — run the build-speaker-mapping skill first.
---

# Translate the Deepgram JSON into a transcript

Applies a confirmed speaker mapping to Deepgram's `utterances` and writes the
`MM:SS <Label>: <text>` transcript that **summarize-session** reads.

## Steps

### 1. Load the speaker map

Find the session: the date the user named, else the newest `sessions-raw/<DATE>/`. The map
lives at `sessions-raw/<DATE>/speaker-map.json`.

**If it isn't there, stop.** Tell the user to run the **build-speaker-mapping** skill
first — the numbers Deepgram assigns change with every conversion, so a map from another
session is worthless and a guessed one silently mislabels the whole transcript. Do not
fabricate one.

If it is there, check it survived any hand-editing, and show it so the user can see what's
about to be applied:

```bash
jq empty sessions-raw/<DATE>/speaker-map.json && cat sessions-raw/<DATE>/speaker-map.json
```

A parse error means a bad manual edit (usually a trailing comma) — report it and offer to
fix.

### 2. Run the conversion

Locate the `session.deepgram.json` (repo root, or inside the dated folder), then:

```bash
jq -r --argjson map "$(cat sessions-raw/<DATE>/speaker-map.json)" '
  def p2: tostring | if length < 2 then "0" + . else . end;
  .results.utterances[]
  | (.start|floor) as $t
  | "\(($t/60|floor)|p2):\(($t%60)|p2) \($map[(.speaker|tostring)] // "Speaker \(.speaker)"): \(.transcript)"
' session.deepgram.json > sessions-raw/<DATE>/transcript.md
```

If `sessions-raw/<DATE>/transcript.md` already exists, confirm with the user before
overwriting it — it may carry hand-corrections.

### 3. Check the output

Don't assume it worked; look:

```bash
wc -l sessions-raw/<DATE>/transcript.md
head -5 sessions-raw/<DATE>/transcript.md
grep -c '^[0-9][0-9]*:[0-9][0-9] Speaker [0-9]' sessions-raw/<DATE>/transcript.md
sed -E 's/^[0-9]+:[0-9]+ ([^:]+):.*/\1/' sessions-raw/<DATE>/transcript.md | sort | uniq -c | sort -rn
```

- The **per-label line counts** should look like a session at the table: the DM well
  ahead, every player present. A missing or wildly over-represented name means the map is
  wrong, not the transcript.
- A nonzero **`Speaker <N>`** count means that number wasn't in the map. Report it —
  don't quietly leave it in the transcript the summarizer will read. Offer to re-run
  **build-speaker-mapping** to name them.
- Show the user the first few lines so they can eyeball the format.

### 4. Next step

With `sessions-raw/<DATE>/transcript.md` in place (plus any optional `dm-notes.md` /
`<player>-notes.md`), the session is ready for the **summarize-session** skill.
