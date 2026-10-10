---
title: A status line for the context budget
description: Building a Claude Code status line around what ends a session (auto-compaction) and what makes it expensive (an expired prompt cache, the quotas).
date: 2026-10-10
---

A long Claude Code session doesn't end when the model runs out of ideas. It ends in one of two ways: auto-compaction rewrites the conversation into a summary and the details are gone, or you hit a quota and wait for the reset. On the way there, one more thing quietly makes it expensive: the prompt cache expiring between two messages.

None of the three was visible where I look all day, so I built a status line that shows them. Two lines, refreshed after every message:

```text
⌛1h06m TTL 42m │ Opus 3/5 │ 90.0k/167k 53%
5h ███│░░░░░░░  30% ok →60% 🔄2h30m │ 7d █████│░░░░  50% ok →75% 🔄2d07h │ ⎇ feat/my-branch* #412✓
```

It's a Bash script reading the JSON payload Claude Code pipes into the status line command. Here is what each part does, and why it's computed the way it is.

## Context: count against the trigger, not the window

The obvious counter is tokens used against the model's context window. It's also the wrong one. Compaction fires well before the window is full, and compaction is what ends the session's memory, so that's the ceiling worth showing.

Where exactly does it fire? I looked at 624 auto-compactions in my own transcripts, all under a 175k `autoCompactWindow`: the median fired at 143k, about 32k below the setting. So the counter is scored against `autoCompactWindow` minus a 33k reserve. In the example above, `90.0k/167k` is a 200k window less that reserve.

The colour follows that limit: green up to half, yellow to three quarters, orange to nine tenths, then red, and `⚠ COMPACTING` once it's reached.

## Prompt cache: a countdown, and the price of missing it

Every message resends the whole conversation. With a warm cache, the prefix is read at a fraction of the input price; once the cache has expired, the next message processes everything again at full price. Coming back from lunch to a two-hour session costs a lot more than the short reply you're about to type suggests.

The payload says whether caching is on, when the cached prefix expires and the TTL in use (five minutes or an hour). The status line turns that into `TTL 42m`, coloured by the share of the TTL left: green while more than half remains, yellow down to a fifth, then orange. Once it has expired, it shows `EXP 🔄180k`: the number of tokens the next message will pay for in full. It puts a number on the choice between sending the message now and starting a fresh session from a handover.

## Quotas: pace, not just percentage

`30%` of the 5-hour quota tells you nothing on its own. Thirty percent ten minutes into the window is a problem; thirty percent four hours in is a quiet afternoon. What matters is how usage compares with the time elapsed.

Each window (5 hours and 7 days) gets a bar that fills with usage, and a `│` tick where even spending would put you by now. Fill left of the tick is green; fill past it means you're ahead of pace and takes the verdict's colour.

The verdict extrapolates the average burn rate to the reset: usage divided by the share of the window elapsed.

- `ok →60%`: at this rate, the window ends at 60%.
- `tight →95%`: it lands between 90 and 100%.
- `out 1h20m`: the limit comes before the reset, in that long.
- `max`: the limit is reached.

A projection made two minutes into a window is noise, so none shows before a tenth of the window has passed. `🔄2h30m` is the time left before the reset.

## The rest

The first line also carries the session's age (`⌛1h06m`), the model and the effort level as a position on the five-step scale, in the colours `/effort` uses (`3/5` for high), plus `⚡` in fast mode. The second line ends with git: branch, `*` when tracked files changed, the worktree, and the PR number with `✓` when approved or `✗` when changes are requested.

Two engineering choices kept it fast and robust:

- **One `jq` call per refresh.** Every field comes out of the payload in a single pass. A status line runs after every message; spawning a process per field adds up.
- **Lazy segments.** `git status` and the segments other plugins add are the slow part. Each refresh prints the value the previous refresh computed and recomputes it in the background, so the line never waits on git. A run still going when the next refresh comes isn't started twice, so a hung `git` can't pile up.

It runs under macOS's Bash 3.2, and a bad setting never blanks the line: an invalid value falls back to its default and a leading `⚠ ignored:` segment names it.

## Where it went next

The script shipped as the [`statusline` plugin](https://github.com/FunDrivenDev/claude-plugins/tree/main/plugins/statusline) in [FunDrivenDev/claude-plugins](https://github.com/FunDrivenDev/claude-plugins). It's now deprecated: the same readings moved into the `session-panel` plugin, a panel drawn by Claude Code itself rather than a shell script, where the quota bars can unfold and the handover can start from a button. The ideas didn't change: count context against the compaction trigger, show the cache as a countdown with a price, and judge quotas by pace.
