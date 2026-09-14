# Atuin v18 cheat sheet

Atuin replaces normal shell-history search with a searchable SQLite database. In addition to the command, it records the working directory, time, duration, exit status, host, user, and shell session. Sync is optional and end-to-end encrypted.

> Applies to Atuin 18.x. Features marked **18.13+** or **18.18+** need that patch release or newer. Check with `atuin --version`.

## 1. Quick start

```bash
# Check installation and diagnose shell integration
atuin --version
atuin doctor
atuin info

# Import the existing history for the current shell
atuin import auto

# Open the interactive search interface
atuin search -i
```

Atuin normally binds both `Ctrl+R` and the `Up` arrow. In the search interface:

| Key | Action |
|---|---|
| `Ctrl+R` | Cycle scope: global, host, session, directory, workspace |
| `Ctrl+S` | Cycle matching: fuzzy, prefix, full text, daemon-fuzzy |
| `Up` / `Down` or `Ctrl+P` / `Ctrl+N` | Move through results |
| `Enter` | Run the selected command (depending on `enter_accept`) |
| `Tab` | Put the command on the prompt so you can edit it |
| `Ctrl+Y` | Copy the selected command |
| `Ctrl+O` | Inspect runs, session and statistics for the command |
| `Esc` / `Ctrl+C` | Close search |
| `Ctrl+A`, then `D` | Delete every entry matching the selected command |
| `Ctrl+A`, then `C` | Switch to the selected command's session context |

The `Ctrl+A` shortcuts are two-step combinations: press and release `Ctrl+A`, then press the second key.

## 2. Shell setup

Add the appropriate line to the shell startup file, then start a new terminal:

```bash
# Bash: ~/.bashrc
eval "$(atuin init bash)"

# Zsh: ~/.zshrc
eval "$(atuin init zsh)"

# Fish: ~/.config/fish/config.fish
atuin init fish | source

# PowerShell: $PROFILE
atuin init powershell | Out-String | Invoke-Expression
```

Bash 18.18+ automatically loads its bundled `bash-preexec` if another supported preexec backend is not present. To opt out:

```bash
eval "$(ATUIN_NO_BUILTIN_PREEXEC=1 atuin init bash)"
```

Keep normal `Up`-arrow behaviour while retaining Atuin on `Ctrl+R`:

```bash
eval "$(atuin init bash --disable-up-arrow)"  # use zsh as appropriate
```

To disable only `Ctrl+R`, use `--disable-ctrl-r`. To let Atuin create no bindings, set `ATUIN_NOBIND=true` before `atuin init`.

## 3. Everyday commands

```bash
# Interactive search, optionally starting with a query
atuin search -i
atuin search -i docker

# Non-interactive search
atuin search docker
atuin search --limit 10 docker
atuin search --cwd . docker
atuin search --exit 0 git
atuin search --exclude-exit 0
atuin search --after "yesterday 3pm" docker
atuin search --before "2026-09-01" --after "2026-08-01" rsync

# Show history
atuin history list
atuin history list --cwd
atuin history list --session --human
atuin history list --cmd-only
atuin history list --reverse

# Usage statistics
atuin stats
atuin stats today
atuin stats week
atuin stats month
atuin stats year

# Diagnostics and locations
atuin doctor
atuin info
```

Useful output formatting:

```bash
atuin search --format '{time}  {duration}  {directory}  [{exit}]  {command}' docker
atuin history list --format '{time}\t{directory}\t{command}'
```

Search format fields include `{command}`, `{directory}`, `{duration}`, `{user}`, `{host}`, `{time}`, `{exit}`, and `{relativetime}`. `history list` supports the same core fields, though the exact set varies slightly by v18 patch release.

## 4. Search scopes and matching

### Filter modes: where to search

| Mode | Searches |
|---|---|
| `global` | All recorded history from every synced machine |
| `host` | This machine only |
| `session` | This shell session only |
| `directory` | The current directory only |
| `workspace` | Anywhere in the current Git repository; requires `workspaces = true` |
| `session-preload` | Current session plus global history from before it began |

Recommended setup: global search on `Ctrl+R`, current-directory history on `Up`.

```toml
filter_mode = "global"
filter_mode_shell_up_key_binding = "directory"
workspaces = true
```

### Search modes: how to match

| Mode | Behaviour |
|---|---|
| `fuzzy` | Flexible matching using fzf-style query syntax |
| `prefix` | Commands beginning with the query |
| `fulltext` | Query occurring anywhere in the command |
| `daemon-fuzzy` | Fast in-memory fuzzy search with tunable ranking; **18.13+** |

Fuzzy query syntax:

| Query | Meaning |
|---|---|
| `dcker` | Fuzzy match |
| `'docker compose` | Contains the exact phrase |
| `^sudo` | Starts with `sudo` |
| `.log$` | Ends with `.log` |
| `!password` | Does not contain `password` |
| `^git pull$` | Exact whole command |
| `^core go$ \| rb$ \| py$` | Starts with `core` and ends in `go`, `rb`, or `py` |

The `|` OR operator is not supported by `daemon-fuzzy` in v18.

## 5. Advanced command-line searches

```bash
# Successful Docker commands from the current directory
atuin search --cwd . --exit 0 docker

# Failed commands from the last seven days
atuin search --exclude-exit 0 --after "1 week ago"

# Commands outside a noisy directory
atuin search --exclude-cwd /var/log rsync

# Oldest matching results first
atuin search --reverse --after "1 month ago" apt

# Page through results
atuin search --limit 50 --offset 50 git

# Null-delimited output is safest for multiline commands
atuin history list --cmd-only --print0
```

Dates accept both normal and human-readable expressions such as `yesterday`, `last friday`, and `yesterday 3pm`. Set `dialect = "uk"` if you want ambiguous numeric dates interpreted in UK order.

## 6. Sync across machines

Sync is optional. The server stores encrypted data; the encryption key remains essential for adding or recovering another machine.

```bash
# Create an account; omit -p to enter the password without recording it
atuin register -u USERNAME -e EMAIL

# Display the encryption key—store it in a password manager; never share it
atuin key

# Log in on another machine; omit -p and -k for private prompts
atuin login -u USERNAME

# Synchronise now; -f forces a full sync if records appear missing
atuin sync
atuin sync -f

# Stop using the account on this machine
atuin logout
```

Deleting the server account is destructive but does not remove local data:

```bash
atuin account delete
```

Self-hosted clients set this before registering or logging in:

```toml
sync_address = "https://atuin.example.com"
```

## 7. Configuration files and data

Run `atuin info` for the authoritative paths on your computer. Normal Linux defaults are:

| Path | Purpose | Back up? |
|---|---|---|
| `~/.config/atuin/config.toml` | Client behaviour, UI, search and sync settings | Yes |
| `~/.config/atuin/server.toml` | Configuration for a self-hosted Atuin server | Only if self-hosting |
| `~/.local/share/atuin/history.db` | Local SQLite history database | Yes if history is not otherwise recoverable |
| `~/.local/share/atuin/key` | End-to-end encryption key | **Yes—store securely** |
| `~/.local/share/atuin/session` | Local session state | Usually no |
| `~/.bashrc`, `~/.zshrc`, etc. | Contains the `atuin init` shell hook | Yes, usually with dotfiles |

The base configuration directory can be overridden with `ATUIN_CONFIG_DIR`. Standard XDG environment variables can also change the normal config and data locations.

### Inspect or change configuration

These subcommands exist in newer v18 patch releases. If your build does not recognise them, edit `config.toml` directly.

```bash
atuin config get search_mode
atuin config get search_mode --resolved  # include defaults and overrides
atuin config get enter_accept --verbose  # configured versus effective value
atuin config print                       # print the whole file
atuin config print daemon
atuin config set search_mode fuzzy
atuin config set daemon.enabled true
```

`atuin config set` handles scalar values. Edit arrays and tables directly in TOML.

## 8. Practical `config.toml`

```toml
# ~/.config/atuin/config.toml

# UK-style ambiguous dates in `atuin stats`
dialect = "uk"

# Search all machines with Ctrl+R, but only this directory with Up
search_mode = "fuzzy"
filter_mode = "global"
search_mode_shell_up_key_binding = "fuzzy"
filter_mode_shell_up_key_binding = "directory"
workspaces = true

# Keep the UI compact; 0 means full-screen
style = "compact"
inline_height = 30
show_preview = true
max_preview_height = 4

# Safer: Enter inserts for editing instead of immediately executing
enter_accept = false
exit_mode = "return-query"
keymap_mode = "emacs"  # or auto, vim-normal, vim-insert

# Record failed commands as well as successful ones
store_failed = true

# Optional encrypted sync
auto_sync = true
sync_frequency = "10m"
sync_address = "https://api.atuin.sh"

# Never record matching commands or commands run in matching directories.
# These are regular expressions and are unanchored unless you add ^ or $.
history_filter = [
  "--password",
  "--token",
  "^export .*(_KEY|_TOKEN|_SECRET)=",
]

cwd_filter = [
  "^/path/to/private/project",
]
```

Treat `history_filter` as a safety net, not a complete secrets-management system. Atuin also has a built-in secrets filter, but it can only recognise known credential formats. For a one-off sensitive command, begin it with a space; Atuin honours shell `ignorespace` behaviour.

### Fast daemon-backed search (18.13+)

```toml
search_mode = "daemon-fuzzy"

[daemon]
enabled = true
autostart = true

[search]
# Larger values give that signal more influence
frequency_score_multiplier = 2.0
recency_score_multiplier = 1.0
frecency_score_multiplier = 1.0
```

From the command line, `daemon-fuzzy` falls back to ordinary fuzzy behaviour; its in-memory ranking is for the interactive interface.

## 9. Delete and clean up safely

Preview matches before deleting them:

```bash
# Preview
atuin search --after "1 week ago" secret-command

# Delete exactly those matches
atuin search --delete --after "1 week ago" secret-command
```

After adding `history_filter` or `cwd_filter`, remove older matching records:

```bash
atuin history prune --dry-run
atuin history prune
```

In the TUI, select an entry, press `Ctrl+O` to inspect it, then `Ctrl+D` to delete that single run. With sync enabled, history deletions propagate to other machines.

Avoid `atuin search --delete-it-all` unless you genuinely intend to erase all local history.

## 10. Troubleshooting

```bash
# Show version, paths and effective environment
atuin --version
atuin info

# Check whether shell hooks are active
atuin doctor
atuin doctor | grep preexec

# An interactive Bash/Zsh shell should include i in this output
echo "$-"

# Confirm the binary being executed
command -v atuin
```

If `shell.preexec` is `none`, verify that the terminal launches an interactive shell, sources the correct startup file, and runs `atuin init` during startup. After changing shell setup, open a fresh terminal rather than merely re-running the search command.

## 11. Five settings worth changing first

```toml
# Good general-purpose starting point
dialect = "uk"
filter_mode = "global"
filter_mode_shell_up_key_binding = "directory"
workspaces = true
enter_accept = false
```

## Official references

- [Atuin documentation](https://docs.atuin.sh/latest/)
- [Basic usage](https://docs.atuin.sh/latest/guide/basic-usage/)
- [Search reference](https://docs.atuin.sh/latest/reference/search/)
- [Configuration reference](https://docs.atuin.sh/latest/configuration/config/)
- [Key bindings](https://docs.atuin.sh/latest/configuration/key-binding/)
- [Sync guide](https://docs.atuin.sh/latest/guide/sync/)
- [Deleting history](https://docs.atuin.sh/latest/guide/delete-history/)

