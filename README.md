# Voice eval recorder

Self-contained HTML tool for producing the `(audio, transcript, expected_parse)` triples that feed voice-pipeline training and accuracy evaluation for sports scoring apps.

No build step. No backend. Runs entirely in your browser. All state lives in IndexedDB until you hit Export.

## Quick start

```bash
# Serve this directory over http (Web Speech needs http://, not file://):
npx serve .
# → opens http://localhost:3000 (or whatever port serve picks)
```

Open the URL in **Chrome** (preferred) or **Safari**. Firefox does not support the Web Speech API and the recorder will say so in the mic badge.

> Tip: granting microphone permission once persists for the localhost origin. Subsequent sessions skip the prompt.

### iPhone use

The hosted GitHub Pages URL works directly on iPhone Safari (HTTPS satisfies the Web Speech requirement). Add it to your home screen for one-tap launch at games. All recordings stay on the device until you export.

## Tabs

The recorder has four tabs, each fully self-instructing — open the tab and the on-screen "How this tab works" panel walks you through it. Quick reference:

### 1. Quick test (start here)

Verifies the recorder works end-to-end on this device in ~30 seconds. Four checks: mic permission, speech-to-text, IndexedDB write, export shape. Nothing is permanently saved — the test row is deleted at the end.

Use it before every game to confirm the device is recorder-ready.

### 2. Practice with prompts

Walks you through 50 fixed scenarios (see `prompts.json`). Each prompt shows:
- the game-state context (`idle`, `awaiting_assist`, etc.)
- a brief description (*"Kai (#23) makes a 2-pointer, no assist"*)
- the expected parse (collapsed by default — open it to verify before saving)

For each prompt: tap **Start**, speak naturally, tap **Stop**, fix the transcript if Web Speech got a word wrong, tap **Save & Next**.

Designed to take ~20–30 minutes for all 50 prompts. Best for guaranteed coverage of edge cases real games might miss. Each prompt's expected parse is pre-defined — labelling is automatic.

### 3. Capture a game

The recorder watches a real game (or a recording / video) with this tool open in a second tab on the same device. Workflow:

1. **During the game** — for each utterance you want to capture:
   - Tap **Start utterance**, speak, tap **Stop**.
   - Each utterance is timestamped and saved to the captured list.
2. **After the game** — work through the captured list:
   - The first captured utterance opens automatically in the **Label expected parse** card.
   - Tap an event button (`2PT make`, `STL`, etc.), enter the jersey, pick the side.
   - Tap **Save & next**.
3. Continue until every captured utterance is labelled. Skip a row if you can't tell what was said.

Best for capturing real gym acoustics + real speech disfluencies. Plan ~10 minutes of post-game labelling for a 60-minute game.

You can mix tabs — record from a real game in **Capture**, top up to 200 entries in **Practice** to cover any corner cases the real game didn't hit.

### 4. Export

When you have ≥200 saved rows:

1. Tap **Download .jsonl** — your phone saves `voice_parse_eval.jsonl` to Downloads.
2. **Rename it now** in the Files app to something dated like `voice_2026-05-15_vs-cougars.jsonl`. Otherwise the next game's download collides.
3. AirDrop / email / iCloud the file to your laptop within the day. iOS Safari can evict IndexedDB after ~7 days of no PWA visits.
4. Tap **Clear all saved rows** so the next game starts clean.

## Output schema

Each line in the exported JSONL is a JSON object:

```json
{
  "transcript": "kai three",
  "context": "idle",
  "expected_parse": {
    "event_type": "3PT_MADE",
    "player_jersey": "23"
  },
  "wall_clock_ms": 1719000000000,
  "mode": "prompted",
  "prompt_id": "p001"
}
```

Fields:
- **`transcript`** (string) — the (possibly-edited) Web Speech transcript.
- **`context`** (string) — one of `idle | awaiting_assist | awaiting_foul_type | awaiting_sub_in | awaiting_fast_break | awaiting_second_chance`. Capture rows always save `idle` — you can hand-edit later if needed.
- **`expected_parse`** (object | array) — the parse a correct voice parser should return. Schema:
  - Single event: `{ event_type, player_jersey?, metadata? }`
  - Multi-event: array of single events
  - Non-event utterance: `{ intent: "no_assist" | "skip" | "undo" | "no_event" | "ambiguous" | "low_confidence", candidates? }`
- **`wall_clock_ms`** (number) — when the utterance was captured. Useful for downstream timing analysis.
- **`mode`** (`"prompted" | "live"`) — which tab produced the row. (Names kept stable for backwards compatibility with existing exports — the on-screen labels read "Practice" and "Capture".)
- **`prompt_id`** (string, optional) — the `prompts.json` row id when `mode === "prompted"`. Lets you cross-reference the original scenario.

## Files in this dir

| File | Purpose |
|---|---|
| `index.html` | The standalone recorder. Open in a browser (served over http or https). |
| `prompts.json` | 50 Practice prompts covering simple events, follow-up contexts, multi-event, filler-laden, ambiguous. |
| `synthetic-seed.json` | 200-entry synthetic dataset. Use this if you don't have real-game data yet. Suitable as test fixture data anytime. |
| `README.md` | This file. |

## When to re-run the tool

| Trigger | Action |
|---|---|
| Voice parser accuracy regressing | Re-run, add 50 more Practice prompts targeting the failing patterns |
| Before swapping the parser implementation | Re-run during a real game with the current parser; archive the corpus for before/after comparison |
| Adding a new sport | Build a sport-specific `prompts.json` and run the tool against that sport's commentary |

## Troubleshooting

- **Mic badge shows "unsupported"** → switch to Chrome or Safari. Firefox's Web Speech support is too patchy.
- **Quick test fails on Speech-to-text** → speak within 5 seconds of the step turning amber. iOS Safari sometimes needs you to enable "Dictation" in Settings → General → Keyboard.
- **Recording starts but no transcript appears** → check OS microphone permissions for the browser. Chrome → Settings → Privacy → Site Settings → Microphone.
- **Saved rows disappeared between sessions** → the IndexedDB origin is `http://localhost:3000` (or whatever `npx serve` picked). If you started a different port last time, the data lives at that origin. Switch back to it, or use `localhost` consistently.
- **Export downloads an empty file** → no labelled rows yet. Captured (live) rows need to be reviewed and labelled before they appear in export.
