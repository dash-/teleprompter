# Voice-Tracking Teleprompter

## Context

We generate QA video scripts in a markdown format with blockquoted dialogue (`> "text"`) and bulleted action cues (`- do this`). Recording these requires reading dialogue aloud while performing on-screen actions — currently there's no tool that shows both side-by-side and tracks where you are by listening to your voice. The goal is a single-file web app that parses our script format, displays it as a teleprompter, and auto-advances based on syllable matching via the Web Speech API.

## Architecture

Single `index.html` file — HTML + CSS + vanilla JS. No build step, no dependencies. Open in browser, paste/load a script, start recording.

Location: `~/labs/teleprompter/index.html`

## Script Parsing

Parse the existing markdown format into segments:

```
## Step N: Title        → segment boundary
> "dialogue text"       → what to say (voice-tracked)
- action item           → what to do (displayed as cue, not tracked)
---                     → visual separator (ignored)
```

Each segment contains:
- `title` — the step heading
- `dialogue` — concatenated blockquote lines (stripped of `>` and quotes)
- `actions` — array of bullet items
- `syllableCount` — pre-computed from dialogue text

The Intro and Outro sections are segments too.

## UI Layout

```
┌─────────────────────────────────────────────────┐
│  Step 2: Admin launch immediately        [2/4]  │  ← segment title + progress
├───────────────────────────┬─────────────────────┤
│                           │                     │
│  Same thing from the      │  • Sign out and     │  ← dialogue (large, readable)
│  admin side. I'll sign    │    sign back in as   │     + actions (secondary)
│  out and back in as the   │    QA admin user    │
│  QA admin user. ▌         │  • Navigate to      │  ← cursor shows current word
│                           │    /home?ff=designs  │
│  Navigate to /home with   │  • Select QA Tests  │
│  the designs flag,        │                     │
│  select QA Tests again.   │                     │
│                           │                     │
├───────────────────────────┴─────────────────────┤
│  ▲ Step 1: Complete flow   ▼ Step 3: Save fail  │  ← prev/next preview
└─────────────────────────────────────────────────┘
```

- **Left pane (70%)**: Dialogue text, large font. Words already spoken are dimmed. Current word is highlighted. Upcoming words are full brightness.
- **Right pane (30%)**: Action cues in a smaller font, always fully visible for the current segment.
- **Top bar**: Segment title, segment number (e.g., 2/4), mic status indicator.
- **Bottom bar**: Prev/next segment preview (one-line truncated).

## Voice Tracking — Syllable Counting

**Core idea**: Don't match *what* was said, just *how much* — measured in syllables.

1. **Pre-compute**: For each segment's dialogue, split into words and count syllables per word. Store a cumulative syllable array so we know "word N starts at syllable M."

2. **Listen**: Web Speech API in continuous mode. As `onresult` fires with recognized words, count their syllables and add to a running total.

3. **Advance cursor**: Map `totalSpokenSyllables` back to the cumulative array to find the current word index. Highlight that word, dim everything before it.

4. **Auto-advance segment**: When spoken syllables reach ~90% of the segment's total, auto-advance to the next segment (with a brief visual transition).

**Syllable counting heuristic**:
```js
function countSyllables(word) {
  word = word.toLowerCase().replace(/[^a-z]/g, '')
  if (word.length <= 2) return 1
  word = word.replace(/e$/, '')  // silent e
  const vowelGroups = word.match(/[aeiouy]+/g)
  return Math.max(1, vowelGroups ? vowelGroups.length : 1)
}
```

This gives: "dragon" → 2, "drag" → 1, "in" → 1. Close enough — exact accuracy doesn't matter because keyboard shortcuts cover any drift.

## Keyboard Controls

| Key | Action |
|-----|--------|
| `→` | Advance cursor one word forward |
| `←` | Move cursor one word backward |
| `↓` | Jump to next segment |
| `↑` | Jump to previous segment |
| `Space` | Pause/resume voice tracking |

Arrow keys adjust the cursor position directly. This is the escape hatch — if voice tracking drifts, tap right arrow a few times to catch up, or left to go back.

## State Machine

```
[Load Script] → READY → (Space) → TRACKING → (Space) → PAUSED → (Space) → TRACKING
                                      ↓                    ↓
                                  (auto-advance or ↓)   (↓ or ↑)
                                      ↓                    ↓
                                  NEXT SEGMENT ←──────────┘
                                      ↓
                                  (last segment + 90% syllables)
                                      ↓
                                    DONE
```

States:
- **READY**: Script loaded, waiting to start. Shows first segment.
- **TRACKING**: Mic active, syllables being counted, cursor advancing.
- **PAUSED**: Mic off, cursor frozen. Arrow keys still work.
- **DONE**: All segments complete. Show summary.

## Script Loading

### CLI (primary)

```bash
~/labs/teleprompter/teleprompt ~/labs/scripts/cor-233.md
```

The `teleprompt` shell script:
1. Reads the markdown file
2. Base64-encodes it
3. Opens `index.html` in the default browser with the content as a URL hash fragment: `file:///...index.html#base64:<encoded>`

The HTML checks `location.hash` on load — if it starts with `#base64:`, it decodes and auto-loads the script, skipping the paste/file-picker screen entirely.

### Browser fallback

If opened without a hash, show a landing screen with:
1. **Paste**: A textarea for pasting markdown.
2. **File input**: Standard file picker for `.md` files.

After loading (either path), parse and switch to READY state showing the first segment.

## Files

- `~/labs/teleprompter/index.html` — the app
- `~/labs/teleprompter/teleprompt` — CLI launcher (shell script, chmod +x)

## Verification

1. Run `~/labs/teleprompter/teleprompt ~/labs/scripts/cor-233.md` — browser opens with script pre-loaded
2. Verify parsing: 5 segments (Intro + Steps 1-3 + Outro), dialogue and actions split correctly
3. Verify keyboard navigation: arrows move word/segment as expected
4. Grant mic permission, press Space, speak — verify cursor advances roughly in sync
5. Press Space to pause, verify cursor freezes, press Space to resume
6. Test drift recovery: press → a few times to manually advance past voice position
7. Open `index.html` directly (no args) — paste screen should appear as fallback
