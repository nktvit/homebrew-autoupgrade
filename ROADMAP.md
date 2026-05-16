# Roadmap

A running list of things to fix, polish, or build. Loosely prioritised.

## 1.0.1 — shipped

- [x] **`wait` + `set -e` reachability bug** — the "continue with stale data" branch was unreachable because `set -e` killed the script on `wait` of a failed `brew update`. Fixed by reaping the process via `if ! wait`.
- [x] **`brew au` alias** — added `cmd/brew-au` symlink so the short form works for every subcommand.
- [x] **Symlink-aware `SCRIPT_PATH`** — plist always points at the canonical script even when invoked via the alias.
- [x] **Self-upgrade detection** — log a line when `brew update` pulls a newer version of this script into the tap.

## 1.0.2 — shipped (wizard polish)

- [x] **Default schedule is now time window** instead of "Anytime" — most users want overnight upgrades.
- [x] **"Anytime" reworded** to "Any time of day — upgrades may run while you're working", which is what no-restriction actually means.
- [x] **`[↵]` hints on every wizard prompt** so it's obvious what pressing Enter will do.
- [x] **Optional initial upgrade as final wizard step** — fixes the awkward "Last run: Never" right after setup; streams output to the terminal.
- [x] **"Useful commands" hint uses the `brew au` alias** since that's what you'll actually type.

## 1.0.3 — shipped (alias-everywhere + smart defaults)

- [x] **Every user-facing message uses `brew au`** instead of `brew autoupgrade`. The long form still works; it's just no longer what we suggest.
- [x] **`brew au run` and `brew au start` no longer require a flag** — they default to the mode you chose during `setup` (persisted to `~/.config/brew-autoupgrade/mode`), falling back to `--selected` if no choice has been recorded yet.
- [x] **Legacy auto-migration** — existing installs without a `mode` file get one written on their first run, derived from whatever the launchd plist passed.

## 1.0.4 — shipped

- [x] **`status` now shows when the script was last updated** plus the running version (e.g. `Updated: 2026-05-16 22:17:47 (v1.0.4)`). Derived from the script file's mtime, which changes whenever `brew update` pulls a new tap revision.

## 1.0.5 — shipped

- [x] **`status` shows what packages were upgraded on the last run.** Implemented via `brew list --versions` snapshots before & after the upgrade phase, joined on package name to detect actual version changes. Stored in `~/.config/brew-autoupgrade/last-changes`. Truncated to 10 lines in `status` with a "... and N more" footer for big upgrade runs.

## 1.0.6 — shipped (interactive add/remove + small refactor)

- [x] **`brew au add` / `brew au remove` accept multiple packages** and report per-package outcome (`✓ added / - already present / ✗ not installed`, summary line).
- [x] **`brew au add` / `brew au remove` with no args open an fzf multi-select picker** (the same one used by the wizard). For `add`, the candidate list is "installed packages not already in your allowlist"; for `remove`, it's the current allowlist.
- [x] **Extracted helpers** for reuse: `count_lines`, `read_window`, `installed_packages`, `pick_packages`. Collapsed ~25 lines of duplication and locked the picker behind a single function so future flows can call it too.
- [x] **Bash 3.2 portability** — replaced `declare -A` / `mapfile` with portable alternatives. The script now runs on stock macOS `/bin/bash` (3.2.57) without needing brew's bash 4+.

## 1.0.7 — small bug-fix release

- [ ] **Tighten the `--window` regex.** Currently accepts `25:99-29:00`; should be `(?:[01][0-9]|2[0-3]):[0-5][0-9]` on both sides.
- [ ] **Normalise zero-padded hours in the setup wizard.** Right now typing `1:00-4:00` is stored as-is and `status` shows `01:00 - 4:00`. Either reject or pad to `01:00-04:00`.
- [ ] **Reject `--window 03:00-03:00`** (start == end produces an unreachable window).
- [ ] **Error on conflicting flags.** `brew au run --all --selected` silently takes the last flag; should fail loudly.
- [ ] **`gum` integration as optional prettier wizard.** Detect at runtime, use for `confirm`/`choose`/`input` if installed, fall back to current text prompts otherwise. Same pattern as fzf.

## 1.1 — polish

- [ ] **Cache `brew list | wc -l`** for `smart_hint`. Every `status` and `help` invocation currently waits 1-2s on a `brew list` call. Cache to `~/.config/brew-autoupgrade/.pkg-count` with a 24h TTL.
- [ ] **Auto-prune the allowlist** when a package is uninstalled. Either prune silently before a run, or surface a one-time message on `status`: *"node is in your allowlist but no longer installed — `brew au remove node`?"*.
- [ ] **Atomic config writes** for `schedule.conf` and `allowlist.txt`. Write to a tempfile + `mv` so an interrupted write can't leave a half-file.
- [ ] **Add `/opt/homebrew/sbin` to the plist `PATH`** for the rare formula that lives there.
- [ ] **`status` shows recent run health,** not just timestamp. Grep the last `═══` block in the log and surface `Last result: 12/12 succeeded` or `Last result: 2 failed (firefox, node)`.
- [ ] **Dry-run mode.** `brew au run --selected --dry-run` prints what would be upgraded without doing it.

## 1.2 — reliability

- [ ] **Multi-target internet probe.** Fall back from `1.1.1.1` to `8.8.8.8` to `captive.apple.com` before giving up. Corporate firewalls sometimes block Cloudflare specifically.
- [ ] **Atomic stop.** Roll back plist deletion if `launchctl bootout` fails, so we don't end up with a loaded service whose config file is gone.
- [ ] **Window-fit validation.** When user sets a `--window` narrower than `PLIST_INTERVAL` (4h), print a warning that they'll likely fall through to the 48h catch-up rather than running daily.
- [ ] **Lock-acquisition race during stale recovery.** Very narrow window where two stale-lock recoverers can both `rm -rf` and one of them ends up with no lock. Mostly benign because `mkdir` is atomic, but worth tightening.

## 1.3 — features

- [ ] **Notifications.** Optional `osascript -e 'display notification ...'` on a successful or failed run, gated by a `~/.config/brew-autoupgrade/notifications` flag.
- [ ] **Cleanup pass.** Run `brew cleanup` after a successful upgrade, behind a `--cleanup` flag or wizard option.
- [ ] **Failure backoff.** If a package fails N runs in a row, automatically mute it for a few days and surface in `status`.
- [ ] **`history` subcommand.** Reconstruct the last N runs from the log file.

## Long-term / maybe-never

- [ ] **Re-implement as a Homebrew "service" instead of a raw plist.** `brew services` is the conventional way; it'd lose some control over the plist but be more discoverable.
- [ ] **Linuxbrew support.** Replace `stat -f%z` and `launchctl` with portable equivalents; use `systemd --user` or cron on Linux.
- [ ] **Test suite.** Bats-based tests for the time-window math, the dedup/catch-up ladder, and the lock recovery.
