---
layout: post
title: "A Friendlier Tmux Cheat Sheet"
date: 2026-06-03
---

Tmux's defaults are powerful but awkward — `Ctrl-b` is a stretch, and the split keys (`%` and `"`) tell you nothing about which way the screen divides. This is the config I settled on, plus a cheat sheet for the bindings it creates.

## The mental model: session → window → pane

Before the keys, the hierarchy. Tmux nests three things:

```
Session   a whole workspace — survives the terminal closing
 └─ Window   a screen inside the session (like a browser tab)
     └─ Pane   a split region within a window
```

- **Session** — a named, persistent collection of windows. It lives on the tmux server even after you close the terminal. You *detach* and later *attach* with everything intact. This is the feature that matters: start a long job, detach, go home, reattach.
- **Window** — one screen inside a session. Only one is visible at a time; you switch between them like tabs.
- **Pane** — a subdivision of a window, so you can see multiple shells side by side at once.

The rule of thumb: **sessions** are for persistence, **windows** are for organizing separate tasks, **panes** are for seeing related things together.

Everything is driven by a **prefix key** first, then a command key. So "split pane" is *prefix* then `|` — not a single chord.

## The config

Drop this in `~/.tmux.conf`. The big changes: prefix moves to `Ctrl-a`, splits become `|` and `-`, panes navigate with vim keys, and the mouse just works.

```tmux
# ── Prefix: Ctrl-a instead of Ctrl-b ──
unbind C-b
set -g prefix C-a
bind C-a send-prefix          # press Ctrl-a twice to send a literal Ctrl-a

# ── Intuitive splits (and open in current path) ──
bind | split-window -h -c "#{pane_current_path}"   # prefix |  → vertical split
bind - split-window -v -c "#{pane_current_path}"   # prefix -  → horizontal split
unbind '"'
unbind %

# ── Move between panes with prefix + h/j/k/l (vim style) ──
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# ── Resize panes with prefix + Shift+H/J/K/L (repeatable) ──
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# ── Mouse support: click panes, drag borders, scroll ──
set -g mouse on

# ── Reload config with prefix + r ──
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# ── Quality of life ──
set -g base-index 1            # windows start at 1, not 0
setw -g pane-base-index 1      # panes too
set -g renumber-windows on     # renumber when one closes
set -g history-limit 10000     # bigger scrollback
set -sg escape-time 10         # snappier (helps vim/neovim)

# ── Vim-style copy mode ──
setw -g mode-keys vi
bind v copy-mode
bind -T copy-mode-vi v send -X begin-selection
bind -T copy-mode-vi y send -X copy-selection-and-cancel

# ── Subtle status bar ──
set -g status-style 'bg=#333333 fg=#ffffff'
set -g status-left-length 20
set -g status-right '%Y-%m-%d %H:%M '
```

A few notes on the choices:

- **`Ctrl-a` as prefix** is the GNU screen convention and an easier reach. The trade-off is that `Ctrl-a` (jump to start of line) now needs a double tap — `Ctrl-a a`.
- **`-c "#{pane_current_path}"`** makes new splits open in the *same directory* as the pane you split from. Small thing, huge quality-of-life win.
- **`-r` on the resize binds** means "repeatable" — after the prefix, you can keep tapping `H/J/K/L` to resize without re-pressing the prefix.

## Cheat sheet

In everything below, **`prefix` = `Ctrl-a`**.

### Sessions

| Action | Keys / Command |
|---|---|
| New named session | `tmux new -s name` |
| List sessions | `tmux ls` |
| Attach to last session | `tmux attach` |
| Attach to named session | `tmux attach -t name` |
| Detach from session | `prefix d` |
| Rename current session | `prefix $` |
| Switch / pick a session | `prefix s` |
| Kill a session | `tmux kill-session -t name` |

### Windows

| Action | Keys |
|---|---|
| New window | `prefix c` |
| Next / previous window | `prefix n` / `prefix p` |
| Jump to window N | `prefix 1` … `9` |
| Rename current window | `prefix ,` |
| List / pick a window | `prefix w` |
| Close current window | `prefix &` |

### Panes

| Action | Keys |
|---|---|
| Split vertical (side by side) | `prefix |` |
| Split horizontal (stacked) | `prefix -` |
| Move between panes | `prefix h / j / k / l` |
| Resize pane (repeatable) | `prefix H / J / K / L` |
| Cycle through panes | `prefix o` |
| Toggle pane zoom (fullscreen) | `prefix z` |
| Convert pane → its own window | `prefix !` |
| Show pane numbers (then press one) | `prefix q` |
| Close current pane | `prefix x` |

### Copy mode (scrollback)

| Action | Keys |
|---|---|
| Enter copy mode | `prefix v` |
| Move around | `h / j / k / l`, arrows, `Ctrl-u` / `Ctrl-d` |
| Start selection | `v` |
| Copy selection and exit | `y` |
| Search backward / forward | `?` / `/` |
| Quit copy mode | `q` |

The mouse also works for all of this — click to select a pane, drag a border to resize, scroll to enter copy mode.

### Misc

| Action | Keys |
|---|---|
| Reload config | `prefix r` |
| Command prompt | `prefix :` |
| List all key bindings | `prefix ?` |

## The three commands worth memorizing first

If you remember nothing else:

1. **`tmux new -s work`** — start a named session.
2. **`prefix d`** — detach. Close the terminal; the session keeps running.
3. **`tmux attach -t work`** — come back to exactly where you left off.

That detach/attach loop is the whole reason tmux exists. The panes and windows are convenience on top. Everything else on this page you can look up with `prefix ?` until it's in your fingers.
