# GNU Screen cheat sheet

GNU Screen is a terminal multiplexer: it keeps shells and commands running inside persistent sessions, even when an SSH connection or terminal window closes. A Screen session can contain multiple windows, and each window can run its own shell or program.

> This sheet covers GNU Screen on Linux, not the unrelated `screen` display/capture features found in other tools. Check your installed version with `screen --version`.

## 1. Quick start

```bash
# Create a named session
screen -S maintenance

# Detach without stopping anything: press Ctrl+A, then D

# List sessions
screen -ls

# Reattach later
screen -r maintenance

# Detach it from another terminal and attach it here
screen -d -r maintenance
```

Screen's default command prefix is `Ctrl+A`, written as `C-a`. Shortcuts are normally sequential: press and release `Ctrl+A`, then press the command key.

| Shortcut | Action |
|---|---|
| `C-a d` | Detach from the session |
| `C-a c` | Create a new window |
| `C-a n` / `C-a p` | Next / previous window |
| `C-a 0` … `C-a 9` | Jump to window number |
| `C-a "` | Show the interactive window list |
| `C-a A` | Rename the current window |
| `C-a k` | Kill the current window after confirmation |
| `C-a ?` | Show Screen's key help |
| `C-a :` | Open Screen's command prompt |

## 2. Installation and startup

```bash
# Ubuntu or Debian
sudo apt update
sudo apt install screen

# Fedora
sudo dnf install screen

# Arch Linux
sudo pacman -S screen

# Verify the installation
screen --version
```

Useful ways to start Screen:

```bash
# Start an unnamed interactive session
screen

# Start a named interactive session
screen -S updates

# Start a session running a particular command
screen -S monitor htop

# Start detached in the background
screen -dmS worker

# Start a detached command; the session ends when the command ends
screen -dmS backup bash -lc '/path/to/backup.sh'

# Use a different configuration file
screen -c /path/to/project.screenrc -S project
```

Always name sessions that you intend to revisit. `screen -S mysql-import` is far easier to recognise than a generated name such as `28413.pts-2.fractal`.

## 3. Everyday commands

### Outside Screen

```bash
screen -ls                    # list your sessions
screen -r NAME                # attach to a detached session
screen -d NAME                # detach a session from its current terminal
screen -d -r NAME             # detach elsewhere, then attach here
screen -x NAME                # share an attached session
screen -wipe                  # remove dead session records
screen -S NAME -X quit        # terminate an entire named session
```

Use the shortest unique part of a session identifier if you do not want to type the whole `PID.name` shown by `screen -ls`.

### Inside Screen

| Shortcut | Action |
|---|---|
| `C-a c` | New window |
| `C-a n` / `C-a Space` | Next window |
| `C-a p` / `C-a Backspace` | Previous window |
| `C-a C-a` | Toggle between current and previous window |
| `C-a "` | Choose a window from a list |
| `C-a '` | Prompt for a window name or number |
| `C-a A` | Rename current window |
| `C-a w` | Briefly display the window list |
| `C-a k` | Kill current window |
| `C-a d` | Detach safely |
| `C-a x` | Lock the Screen display |
| `C-a ?` | Show bindings |

Closing a shell with `exit` closes that window. When the final window closes, the Screen session ends.

## 4. Sessions, windows and regions

| Object | Meaning | Typical command |
|---|---|---|
| Session | The persistent Screen container | `screen -S NAME` |
| Window | A virtual terminal inside the session | `C-a c` |
| Region | A visible split showing a window | `C-a S` or `C-a \|` |

A split creates another viewing region; it does **not** create a new shell. After splitting, switch into the new region with `C-a Tab`, then choose an existing window or create one with `C-a c`.

### Split-screen shortcuts

| Shortcut | Action |
|---|---|
| `C-a S` | Split horizontally |
| `C-a \|` | Split vertically |
| `C-a Tab` | Move focus to the next region |
| `C-a X` | Close the current region |
| `C-a Q` | Close every region except the current one |
| `C-a :resize 20` | Resize current region to 20 lines |
| `C-a :resize +5` | Grow current region by five lines |
| `C-a :fit` | Fit the window to the current region |

## 5. Advanced usage

### Scrollback, copy and paste

```text
C-a [       enter copy/scrollback mode
Arrow keys  move around
Page Up/Down scroll
Space       begin selection
Space       finish selection and copy
Esc         leave copy mode
C-a ]       paste Screen's copy buffer
```

In vi-style copy mode, `h`, `j`, `k`, `l`, `0`, `$`, `w`, `b`, `g` and `G` also navigate. Screen's paste buffer is separate from the desktop clipboard.

Increase the available history interactively:

```text
C-a :scrollback 20000
```

### Logging and hard copies

```text
C-a H                  toggle logging for the current window
C-a :logfile FILE      select a logfile
C-a :hardcopy FILE     write the visible screen to a file
C-a :hardcopy -h FILE  include the scrollback buffer
```

From outside Screen:

```bash
# Start with logging enabled
screen -L -Logfile "$HOME/screen-maintenance.log" -S maintenance

# Capture window 0 and its scrollback
screen -S maintenance -p 0 -X hardcopy -h "$HOME/maintenance.txt"
```

Logs can contain passwords, tokens, customer data or other terminal output. Protect and remove them appropriately.

### Monitoring windows

| Shortcut | Action |
|---|---|
| `C-a M` | Toggle activity monitoring for the current window |
| `C-a _` | Toggle silence monitoring |
| `C-a m` | Repeat the last displayed message |
| `C-a t` | Show system and load information |

### Send Screen commands remotely

`-X` sends a Screen command to a running session:

```bash
# Create and name a new window
screen -S maintenance -X screen -t logs

# Change the session name
screen -S maintenance -X sessionname server-work

# Set window 0's title
screen -S maintenance -p 0 -X title shell

# Ask the complete session to exit
screen -S maintenance -X quit
```

You can inject text into a window, but use this cautiously—it behaves like typing and does not know whether the target program is ready:

```bash
screen -S maintenance -p 0 -X stuff $'uptime\n'
```

### Share a session

```bash
# A second terminal belonging to the same Unix account
screen -x maintenance
```

True multi-user sharing uses Screen's `multiuser` and ACL commands and normally requires a correctly installed setuid Screen binary. It expands access to your terminal and should only be configured deliberately on a trusted host.

## 6. SSH and long-running work

A reliable remote workflow is:

```bash
ssh server
screen -S upgrade

# Run the work, then press C-a d before disconnecting
exit

# Later
ssh server
screen -r upgrade
```

If SSH drops unexpectedly, Screen normally preserves the session. Reconnect and run:

```bash
screen -ls
screen -d -r upgrade
```

Useful patterns:

```bash
# Long import
screen -S import bash -lc 'php artisan import:catalogue'

# Interactive maintenance session with a memorable name
screen -S typesense-maintenance

# Detached backup with output logging
screen -L -Logfile "$HOME/backup.log" -dmS backup \
  bash -lc '/path/to/backup.sh'
```

Screen protects a process from a terminal or SSH disconnection, but not from a server reboot, crash or out-of-memory kill. Use `systemd`, Docker restart policies or another service manager for permanent services.

## 7. Configuration files and data

| Path | Purpose |
|---|---|
| `/etc/screenrc` | System-wide configuration |
| `~/.screenrc` | Your normal per-user configuration |
| Path passed to `screen -c` | Alternative configuration for one invocation |
| `/run/screen/` or `/var/run/screen/` | Runtime session sockets, organised by user |
| `screenlog.0`, `screenlog.1`, etc. | Default log names when logging is enabled |
| `hardcopy.0`, `hardcopy.1`, etc. | Default hard-copy filenames |

Screen reads the system configuration first and then your `.screenrc`. Commands entered through `C-a :` generally use the same syntax as `.screenrc` commands.

Reload your user configuration without restarting the session:

```text
C-a :source $HOME/.screenrc
```

To find the exact runtime directory on your system:

```bash
screen -ls
ls -ld /run/screen /var/run/screen 2>/dev/null
```

Do not manually delete live Screen sockets. Use `screen -wipe` for stale records.

## 8. Practical `.screenrc`

```text
# ~/.screenrc

# Skip the licence/startup message
startup_message off

# Preserve sessions if the terminal disappears
autodetach on

# Keep more scrollback per window
defscrollback 20000

# Prefer a visual message instead of an audible bell
vbell on
vbell_msg "Activity in another Screen window"

# Enable 256-colour applications on modern Linux systems
term screen-256color

# Put useful session and window information on the last line
hardstatus alwayslastline
hardstatus string '%{= kG}[%H] %{= kw}%-Lw%{= kW}%n*%f %t%{-}%+Lw %=%Y-%m-%d %c'

# Name log files by window number and title
logfile $HOME/screenlog.%n-%t
logfile flush 1

# Press C-a R to reload this file
bind R source $HOME/.screenrc
```

If `screen-256color` is missing from the target machine's terminfo database, remove the `term` line or use `term screen`.

### Change the command prefix

`Ctrl+A` clashes with Bash/Readline's “start of line” shortcut. To send a literal `Ctrl+A` to the application inside Screen, press `C-a a`. Alternatively, change Screen's prefix in `.screenrc`:

```text
# Use Ctrl+B as prefix; the second character sends a literal Ctrl+B
escape ^Bb
```

After this change, every shortcut in this sheet begins with `Ctrl+B` instead of `Ctrl+A`.

## 9. Exit and clean up safely

| Goal | Method |
|---|---|
| Leave work running | `C-a d` |
| Close one window cleanly | Run `exit` in that shell |
| Force-close one window | `C-a k`, then confirm |
| End the entire session from inside | Exit every window, or use `C-a :quit` |
| End a named session externally | `screen -S NAME -X quit` |
| Remove stale socket records | `screen -wipe` |

Before terminating a session, check what is running:

```bash
screen -ls
screen -r NAME
```

`screen -S NAME -X quit` stops every process attached to that Screen session. It is not the same as detaching and should be treated as destructive.

## 10. Troubleshooting

```bash
# Confirm version and binary
screen --version
command -v screen

# List sessions and their state
screen -ls

# Check whether the current shell is already inside Screen
printf '%s\n' "$STY"

# Remove records belonging to processes that no longer exist
screen -wipe
```

### “There is no screen to be resumed”

Run `screen -ls` and use the displayed identifier. The session may be attached rather than detached; use:

```bash
screen -d -r SESSION
```

### More than one matching session

Specify the full identifier shown by `screen -ls`, for example:

```bash
screen -r 28413.maintenance
```

### Nested Screen warning

If `$STY` is non-empty, you are already inside Screen. Usually, create another window with `C-a c` instead of starting a nested session. Detach first if you actually want to attach to a different session.

### Colours or terminal applications look wrong

```bash
printf '%s\n' "$TERM"
infocmp screen-256color >/dev/null
```

Inside Screen, `$TERM` will normally be `screen` or `screen-256color`. Remove an unsupported `term screen-256color` setting and reconnect. Run `reset` if a crashed program leaves the terminal garbled.

### Detach shortcut appears not to work

Press the keys sequentially: hold `Ctrl` and press `A`, release both, then press `D`. If the prefix was changed in `.screenrc`, use the configured prefix instead.

## 11. Five things worth remembering first

```text
screen -S NAME       create a named session
screen -ls           list sessions
screen -r NAME       reattach
C-a d                detach and leave work running
C-a ?                show all current key bindings
```

## Official references

- [GNU Screen project](https://www.gnu.org/software/screen/)
- [GNU Screen manual](https://www.gnu.org/software/screen/manual/screen.html)
- [GNU Screen source and releases](https://git.savannah.gnu.org/cgit/screen.git/)
- Your installed manual: `man screen`

