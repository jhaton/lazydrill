# Build `drill`: a Bubble Tea TUI that gamifies a daily interview coding drill

## Who this is for and what it has to enforce

I'm a senior data platform engineer preparing for interview coding rounds. My weakness isn't the problems; it's freezing when observed and going silent when stuck. The drill that fixes that is: 25 minutes a weekday, easy problems, **talk out loud the entire time including when stuck**, hard stop at the timer whether or not the problem is finished. The app's job is to make that routine stick for eight weeks by making it feel like a game, and to refuse to let me cheat the routine (skip the narration, run long, grind mediums before I've earned them).

Style reference: I built `lazybq`, a BigQuery TUI in the lazygit mold — numbered bordered panes, vim keys, a status bar with mode and `<space> menu  ? help`, muted palette, dense but legible. `drill` should feel like a sibling: same conventions, same keybinding grammar, same restraint. No emoji-soup, no ASCII fireworks, no motivational copy. The "game" is in the mechanics and the numbers, not the decoration.

## Stack and constraints

- Go 1.22+, `charmbracelet/bubbletea`, `bubbles` (timer, list, textinput, viewport, progress, table), `lipgloss`. No other UI deps.
- Single static binary. State in `$XDG_DATA_HOME/drill/` (default `~/.local/share/drill/`) as a single SQLite file via `modernc.org/sqlite` (pure Go, no cgo). Problem bank as YAML files in `$XDG_CONFIG_HOME/drill/problems/`, with an embedded starter set.
- Opens my `$EDITOR` for the actual coding (the app is the coach and scoreboard, not an IDE). Each session gets a scratch file at `~/.local/share/drill/sessions/<date>-<slug>.<ext>`.
- Works in an 80x24 terminal; uses more if available.
- Must be usable entirely without the mouse. Every pane reachable by its number key; `j/k` move, `enter` selects, `esc` backs out, `q` quits with confirmation if a session is live, `?` shows keys for the current pane.

## Layout (lazygit-style, numbered panes)

```
┌[1] Today──────────────┐┌[4] Session─────────────────────────────────────┐
│ Thu 29 Oct · streak 6 ││ Deduplicate events by (user_id, event_id),     │
│ ▸ SQL · window dedup  ││ keeping the latest by ingested_at.             │
│   Python · log parser ││                                                 │
│   Design · 8TB ingest ││ Talk the whole time. Stuck is fine. Silent     │
│                       ││ isn't.                                          │
├[2] Week───────────────┤│                                                 │
│ M ■ T ■ W ■ T ▢ F ▢   ││ ⏱ 18:42 remaining   ▮▮▮▮▮▮▮▮▮▮▮▮▮▯▯▯▯▯▯▯      │
│ SQL 1/2 Py 1/2 Des 0/1││ narration check-ins: 3/4 answered              │
├[3] Ledger─────────────┤│ stuck: 2m10s (2 marks)                          │
│ 10-28 ✓ Py   22:10 +31││                                                 │
│ 10-27 ✓ SQL  25:00 +28││ [e] open editor  [s] mark stuck  [n] note       │
│ 10-26 ✗ Des  25:00 +12││ [enter] done     [x] abandon                    │
└───────────────────────┘└─────────────────────────────────────────────────┘
 NORMAL  streak 6 · best 11 · xp 1,240 · lvl 7        <space> menu  ? help
```

- **[1] Today**: the queue for today, generated from the weekly mix. Top item is "up next."
- **[2] Week**: five-day strip with done/missed, plus the mix quota (2 SQL, 2 Python, 1 Design per week by default; configurable).
- **[3] Ledger**: recent sessions, one line each: date, outcome, category, time used, XP earned.
- **[4] Session**: the live drill. Before start it shows the problem and a `[enter] start` prompt. During, it shows the prompt, timer, narration check-in count, stuck marks, and keys. After, it shows the debrief form.
- Status bar: mode, streak, best streak, XP, level, menu/help hint.

## Session mechanics (this is the game)

1. **Start** (`enter` on a queued problem): writes the scratch file with the problem header as a comment, starts a 25:00 countdown (configurable per category; Design defaults to 20:00), opens `$EDITOR` on the scratch file in a subprocess (`tea.ExecProcess`). When the editor exits, return to the session view with the timer still running; `e` reopens.
2. **Narration check-ins**: at randomized intervals (every 4–7 minutes), the session pane shows a one-line prompt — "What are you doing right now? (one sentence)" — with a text input and a 45-second sub-timer. Answering it, with anything non-empty, counts. Ignoring it counts as a miss. Four to five per session. This is the proxy for "talking out loud" that the app can actually measure; make the prompt visually unmissable (invert the pane border color) but never block the editor.
3. **Stuck marks** (`s`): records a timestamp. Pressing `s` again ends the stuck interval. Stuck time is displayed, never penalized. Marking stuck and *then* answering the next check-in earns a bonus (this is the behavior I'm training: keep talking while stuck).
4. **Notes** (`n`): free-text note appended to the session, for things like "forgot how window frames work."
5. **Hard stop**: at 00:00 the timer flashes, the session is frozen, and the debrief opens. There is no "five more minutes." If `$EDITOR` is open at zero, the session still ends when the editor closes; time past zero is recorded as overrun and costs XP.
6. **Done** (`enter`) before the timer: ends early. Finishing early is fine; XP is not time-based.
7. **Abandon** (`x`, confirm): records the session as abandoned. Still counts toward the streak (showing up is what the streak measures), earns minimal XP.
8. **Debrief** (required to close a session): three fields, each one line. "Did you finish? (y/n)", "What was the hardest moment?", "Self-rating 1–5 for narration." Then a summary of XP earned with a breakdown.

## Scoring

Keep it legible; I should be able to predict my score. Suggested:

- Showing up: +10. Completing the timer or finishing: +10.
- Each narration check-in answered: +3. All answered: +5 bonus.
- Each stuck mark followed by an answered check-in: +4.
- Finished the problem: +5 (small on purpose; finishing is not the goal).
- Overrun past the hard stop: −2 per minute, floor at 0 for the session.
- Streak multiplier: ×1.0 at 1–4 days, ×1.25 at 5–9, ×1.5 at 10+. Weekends don't break streaks; a missed weekday does. One "freeze" per calendar month (`space` menu → use freeze) protects a missed day.
- Levels: XP thresholds growing ~1.4× per level. Levels unlock difficulty, not features.

## Difficulty gating

The problem bank tags each problem `easy | medium | hard`. Mediums are locked until level 5 **and** the current two-week narration average is ≥ 4/5 self-rating with ≥ 80% check-ins answered. Hards are locked until level 9. The point: I'm not allowed to grind difficulty until the narration habit is in place. Show the lock reason plainly in the problem list.

## Problem bank

YAML, one file per problem:

```yaml
id: sql-window-dedup
category: sql          # sql | python | design
difficulty: easy
title: Deduplicate events, keep latest
minutes: 25
prompt: |
  Table events(user_id, event_id, ingested_at, payload). Return one row per
  (user_id, event_id), keeping the row with the greatest ingested_at.
  Then: how does your answer change if ingested_at can tie?
ext: sql               # scratch file extension
followups:
  - What if the table is 10 TB and partitioned by ingested_at date?
tags: [window, dedup]
```

Ship ~30 starters: 12 SQL (window functions, dedup, sessionization, SCD2, gaps-and-islands, funnel), 12 Python (log parsing, stream aggregation, rate limiter, LRU, interval merge, top-k), 6 Design prompts (ingest N TB/day from Kafka to a warehouse; exactly-once vs at-least-once; backfill strategy; schema evolution; a shadow rollout plan; a partition-swap publisher that can't lie about failure). Design sessions have no editor; the scratch file is a `.md` for notes and the check-ins ask "what component are you on?"

Selection for the day: fill the week's quota round-robin across categories, prefer problems not seen in 30 days, never two from the same tag in a row.

## CLI subcommands (for scripting and the weekly scorecard)

- `drill` — the TUI.
- `drill today` — print today's queue, plain text.
- `drill log [--week]` — ledger as a table; `--json` for machine output.
- `drill stats` — streak, XP, level, 14-day narration average, check-in rate, per-category counts. This is what I paste into my weekly scorecard.
- `drill add <file.yaml>` — add a problem to the bank.
- `drill freeze` — use this month's streak freeze.

## Data model (SQLite)

`sessions(id, date, problem_id, category, started_at, ended_at, outcome {finished,timeout,abandoned}, overrun_sec, stuck_sec, stuck_marks, checkins_shown, checkins_answered, finished_problem bool, narration_rating, hardest_moment, xp)`; `checkins(session_id, shown_at, answered_at, text)`; `notes(session_id, at, text)`; `meta(key, value)` for streak, freeze-used-month, etc. Streak and level are derived on read, not stored, so a bug can't corrupt them.

## Non-goals

No accounts, no sync, no network calls, no leaderboard, no AI grading of solutions, no code execution. Don't judge my code; judge whether I showed up, talked, and stopped on time.

## Deliverables

- `cmd/drill/main.go`, `internal/{tui,model,bank,store,score}` with the Bubble Tea model split per pane and a root model that routes messages.
- `go test ./...` covering: scoring (table-driven, every rule), streak derivation across month boundaries and freezes, difficulty gating, daily selection determinism given a seed.
- A `Makefile` with `build`, `test`, `install`.
- A README with the layout screenshot, the scoring table, and the keybinding table. Keep the README's tone like the app's: plain.

Start by proposing the pane models and message types, then implement the session state machine with tests before any styling.
