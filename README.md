# brew-autoupgrade

Automatically upgrade your Homebrew packages — all of them or just the ones you care about — with a native macOS background service.

## Features

- **Two upgrade modes**: upgrade everything (`--all`) or only your curated list (`--selected`)
- **Interactive setup wizard**: get running in under a minute with `fzf`-powered package selection
- **Time window scheduling**: restrict upgrades to specific hours (e.g., overnight)
- **Smart catch-up**: if your Mac was asleep or offline during the upgrade window, it catches up at the next opportunity
- **Reliable background service**: uses macOS `launchd` with built-in deduplication — no duplicate runs, no stacking
- **Per-package error handling**: one failed package won't block the rest
- **Automatic log rotation**: keeps logs bounded at ~5MB

## Installation

```bash
brew tap nktvit/autoupgrade
```

That's it. The command `brew autoupgrade` is now available.

## Quick Start

Run the interactive setup wizard:

```bash
brew autoupgrade setup
```

The wizard walks you through:
1. Choosing upgrade mode (all vs. selected)
2. Picking packages with an interactive `fzf` selector
3. Setting an optional time window
4. Starting the background service

## Commands

### Setup & Status

```bash
brew autoupgrade setup       # Run/re-run the setup wizard
brew autoupgrade status      # Show service status, schedule, and allowlist info
brew autoupgrade version     # Show version
brew autoupgrade help        # Show help
```

### Package Allowlist

Manage which packages get auto-upgraded in `--selected` mode:

```bash
brew autoupgrade add node          # Add a package
brew autoupgrade add google-chrome # Casks work too
brew autoupgrade remove node       # Remove a package
brew autoupgrade list              # Show all selected packages
```

Packages must be installed before they can be added to the allowlist.

### Schedule

Control when upgrades are allowed to run:

```bash
brew autoupgrade schedule                       # Show current schedule
brew autoupgrade schedule --window 02:00-06:00  # Only upgrade between 2-6 AM
brew autoupgrade schedule --window 23:00-05:00  # Midnight-spanning windows work too
brew autoupgrade schedule --clear               # Remove time restriction
```

**Catch-up behavior**: If your Mac misses every upgrade window for 48+ hours (sleep, travel, no internet), the next check will run upgrades regardless of the time window.

### Run Upgrades

Manually trigger an upgrade:

```bash
brew autoupgrade run --all       # Upgrade all packages now
brew autoupgrade run --selected  # Upgrade only allowlisted packages now
```

### Background Service

Start or stop the automatic background service:

```bash
brew autoupgrade start --all       # Auto-upgrade everything daily
brew autoupgrade start --selected  # Auto-upgrade only selected packages daily
brew autoupgrade stop              # Stop the background service
```

The service checks every 4 hours and runs upgrades at most once per day. This frequent checking ensures it reliably finds your configured time window, even after sleep/wake cycles.

## How It Works

### Upgrade Flow

When `run` is invoked (manually or by `launchd`), it follows these steps:

1. **Lock** — acquires an exclusive lock to prevent concurrent runs
2. **Internet check** — exits silently if offline (no error in logs)
3. **Deduplication** — skips if last run was less than 23 hours ago
4. **Time window** — checks if current time is within the configured window (or if catch-up is needed)
5. **`brew update`** — fetches latest package info (5-minute timeout)
6. **Upgrade** — runs `brew upgrade` for all or selected packages
7. **Timestamp** — records the run time for deduplication

### All vs. Selected Mode

| | `--all` | `--selected` |
|---|---|---|
| **Scope** | Every installed formula and cask | Only packages in your allowlist |
| **Best for** | Small installations, hands-off users | Large installations, developers |
| **Risk** | May upgrade things you depend on at specific versions | Only touches what you explicitly chose |
| **Speed** | Depends on total packages | Typically much faster |

**Smart hint**: If you have 50+ packages installed, `brew autoupgrade` will recommend `--selected` mode.

## Files & Locations

| Path | Purpose |
|---|---|
| `~/.config/brew-autoupgrade/allowlist.txt` | Selected packages (one per line) |
| `~/.config/brew-autoupgrade/schedule.conf` | Time window configuration |
| `~/.config/brew-autoupgrade/last-run` | Timestamp of last successful run |
| `~/Library/Logs/brew-autoupgrade.log` | Upgrade logs with timestamps |
| `~/Library/LaunchAgents/com.brew-autoupgrade.plist` | launchd service definition |

## Viewing Logs

```bash
# Tail the log file
tail -f ~/Library/Logs/brew-autoupgrade.log

# Search for errors
grep ERROR ~/Library/Logs/brew-autoupgrade.log
```

Logs are automatically rotated when they exceed 5MB.

## Uninstallation

```bash
brew autoupgrade stop               # Stop the background service
brew untap nktvit/autoupgrade       # Remove the tap
rm -rf ~/.config/brew-autoupgrade   # Remove configuration (optional)
```

## License

MIT — see [LICENSE](LICENSE) for details.
