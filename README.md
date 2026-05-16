# brew-autoupgrade

A small Homebrew command that keeps your packages upgraded in the background, behaves itself on a laptop, and gets out of the way.

```bash
brew tap nktvit/autoupgrade
brew autoupgrade setup   # or: brew au setup
```

That's it. The wizard runs in under a minute and you're done.

---

## Why I built it

I wanted `brew upgrade` to happen on its own — quietly, overnight, without nagging me, without piling up if I'd been away, and without becoming yet another thing running on my Mac. Most "auto-upgrade" recipes I'd seen were either a cron job that fires once at 3am (great unless your lid is closed at 3am) or a long-lived Ruby/Python daemon. Neither felt right for a laptop.

So this is what came out of it: a single bash script, one `launchd` agent, plain-text config, no daemon.

## The travelling-laptop problem (the part I'm proud of)

If you cron something to run at 3am and your laptop spent the last week in a backpack, you have a choice: either you missed every single fire, or your machine wakes up and tries to run seven upgrades in a row. Both are bad.

`brew-autoupgrade` doesn't do either. Here's the trick:

- `launchd` is asked to **poll every 4 hours** (`StartInterval`), not to fire at a wall-clock time. A relative timer keeps ticking against elapsed time, so the next opportunity is always coming up — there's no fixed slot to miss.
- Every time it wakes up, the script asks itself one question: *how long since the last successful run?* That single timestamp is the entire state — there is no queue, no backlog, no "pending" anything.
- Under **23 hours** since last run → skip. (Six wakeups in a day all see this and bail in milliseconds.)
- Over 23h **and** inside your time window → run.
- Outside your window **but more than 48h** since last run → run anyway, because clearly the window keeps getting missed. Logged as `CATCH-UP`.

The combined effect: open the lid after a week away and you get **one** upgrade in the next 4-hour window, not seven. Schedule respected when it can be, broken sensibly when it can't.

## What it feels like

Setup is a four-question wizard:

1. **All packages, or selected?** If you have more than 50 things installed it nudges you toward selected mode with a `💡` hint about battery and breakage.
2. **Pick the packages.** If you have `fzf` it launches a multi-select with a `brew info` preview pane. If you don't, it offers to install fzf for you; decline and it falls back to a numbered list.
3. **Time window?** Defaults to `02:00-06:00`. Anything goes — `23:00-05:00` and other midnight-spanning ranges work fine.
4. **Start the service now?** Yes by default.

> **One thing to know about windows:** the service polls every 4 hours and decides on each wakeup whether to upgrade. That means a window narrower than ~4 hours might not catch a tick on any given day — you'd fall through to the 48h catch-up instead. If you want truly daily upgrades, give it a 4-hour window or wider (the default `02:00-06:00` is sized for exactly this reason).

Done. You get a little summary:

```
✅ Setup complete!
   Mode:     Selected packages (12 packages)
   Schedule: 02:00 - 06:00
   Service:  Running
```

Re-run `brew autoupgrade status` any time for the same readout plus when the last run happened.

## Commands

Both `brew autoupgrade` and the shorter `brew au` work for every command.

```bash
brew au setup              # the wizard
brew au status             # service / allowlist / schedule / last run
brew au list               # show your allowlist
brew au add ripgrep        # validates it's actually installed first
brew au remove ripgrep
brew au schedule --window 02:00-06:00
brew au schedule --clear
brew au run --selected     # upgrade now (still honours the lock + dedup)
brew au start --selected   # turn the background service on
brew au stop               # turn it off
```

`start` and `run` always need an explicit `--all` or `--selected` — no magic default. The wizard threads the right one through for you.

## The light stuff under the hood

The whole thing is ~850 lines of bash, no Ruby, no compiled binaries, no resident process. `launchd` owns the lifecycle and the script fires and exits. Some of the patterns that keep it small and well-behaved:

- **mkdir-as-lock with a PID liveness check.** Two `launchd` fires can't overlap, and a previous run that crashed mid-way doesn't leave the script wedged forever — a dead PID in the lock dir is detected and cleaned up.
- **Cheap internet probe** before doing anything. A 5-second `curl` against `1.1.1.1` — captive-portal wifi at a hotel or no signal on a plane means the script exits silently. Nothing in the log, no error, no half-finished `brew update`.
- **5-minute watchdog on `brew update`.** A flaky CDN can't wedge the run.
- **Per-package isolation in selected mode.** One broken cask doesn't poison the rest — they're upgraded one at a time and the summary tells you `succeeded / failed`.
- **Log rotation at 5 MB.** One generation, no `logrotate` dependency, no growing log file.
- **`RunAtLoad=false`** in the plist so loading the service doesn't kick off an immediate upgrade — the 4-hour timer is the only trigger.
- **First-run wizard only on a real TTY.** The launchd invocation never gets stuck waiting on stdin.
- **Plist points at the canonical script path** even when you invoked setup via the `brew au` alias — the script follows symlinks before writing the path in.
- **It upgrades itself.** Every run does a `brew update`, which `git pull`s every tap including this one — so the script keeps itself current as a side effect. The running process safely finishes on the old inode; the new version takes effect on the next 4-hour fire. When that happens you'll see a `Self-upgrade: 1.0.0 → 1.0.1 (active on next run).` line in the log.

## Files

| Path | What's in it |
|---|---|
| `~/.config/brew-autoupgrade/allowlist.txt` | Your selected packages, one per line |
| `~/.config/brew-autoupgrade/schedule.conf` | Two `KEY=VALUE` lines |
| `~/.config/brew-autoupgrade/last-run` | A single epoch timestamp — the entire state machine |
| `~/Library/Logs/brew-autoupgrade.log` | Timestamped run log |
| `~/Library/LaunchAgents/com.brew-autoupgrade.plist` | Generated on `start`, deleted on `stop` |

All plain text. Edit the allowlist by hand if you want — the script doesn't care.

## Watching it

```bash
tail -f ~/Library/Logs/brew-autoupgrade.log
```

Each run is bracketed with `═══` separators. Skips are explicit and tell you why:

```
[2026-05-15 03:14:01] ═══════════════════════════════════════════════════
[2026-05-15 03:14:01] Starting upgrade run (mode: selected)
[2026-05-15 03:14:18] Upgrading 'node'...
[2026-05-15 03:14:42] Upgrading 'ripgrep'...
[2026-05-15 03:14:55] Summary: 12/12 succeeded, 0 failed.
[2026-05-15 03:14:55] ═══════════════════════════════════════════════════
```

Other things you'll see in there:

```
SKIP: Last run was 4h ago (minimum interval: 23h).
SKIP: No internet connection detected.
SKIP: Outside upgrade time window.
CATCH-UP: Outside time window but last run was 52h ago (>48h). Proceeding.
```

## Uninstall

```bash
brew au stop
brew untap nktvit/autoupgrade
rm -rf ~/.config/brew-autoupgrade   # optional — kills your config too
```

## Requirements

macOS, Homebrew, bash 4+. Everything else (`launchctl`, `curl`, `PlistBuddy`) ships with the OS. `fzf` is optional and the wizard will offer to install it.

## License

MIT — see [LICENSE](LICENSE).
