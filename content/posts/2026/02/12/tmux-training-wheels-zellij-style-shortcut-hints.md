---
title: "Tmux Training Wheels: A Zellij-Inspired Shortcut Hints Bar"
date: 2026-02-12T10:00:00+01:00
draft: false
tags: ["tmux", "terminal", "productivity", "developer-experience"]
categories: ["developer-tools"]
---

I recently switched to tmux as my terminal multiplexer. The problem? I kept forgetting the shortcuts. Every time I needed to split a pane or switch sessions, I'd alt-tab to a browser with a cheat sheet open. It completely broke my flow.

Then I remembered [Zellij](https://zellij.dev/) and its always-visible shortcut hints bar at the bottom of the screen. No memorization needed - the available actions are right there. What if I could bring that same idea to tmux?

## The Idea: Training Wheels for Tmux

The concept is simple: use tmux's second status line to display a color-coded bar showing the most common shortcuts, grouped by category:

- **Green** for session operations (detach, list, new, rename)
- **Blue** for window operations (new, next, prev, list, rename)
- **Magenta** for pane operations (split, zoom, kill, navigate)

Think of it as training wheels. You keep them on while you're learning, and once the shortcuts become muscle memory, you toggle them off with a single keybinding.

![Tmux with Zellij-style hints bar visible](/images/2026/02/12/tmux-hints-visible.gif)

## Building the Hints Bar

Tmux supports multiple status lines via `set -g status 2`. The first line (`status-format[0]`) is the default status bar showing windows. The second line (`status-format[1]`) is where our hints bar lives.

Here's the configuration:

```bash
# Enable two status lines
set -g status 2
set -g status-interval 1

# Zellij-style shortcut hints on the second status line
set -g status-format[1] '#[bg=colour235,fg=white] #[fg=green,bold]SESSION #[bg=green,fg=black] ^b #[bg=colour235,fg=colour245]+ #[bg=green,fg=black] d #[bg=colour235,fg=white] Detach #[fg=colour240]| #[bg=green,fg=black] s #[bg=colour235,fg=white] List #[fg=colour240]| #[bg=green,fg=black] a #[bg=colour235,fg=white] New #[fg=colour240]| #[bg=green,fg=black] $ #[bg=colour235,fg=white] Rename #[fg=colour240]| #[bg=green,fg=black] ( #[bg=colour235,fg=white] Prev #[fg=colour240]| #[bg=green,fg=black] ) #[bg=colour235,fg=white] Next #[fg=colour240]|| #[fg=blue,bold]WINDOW #[bg=blue,fg=white] ^b #[bg=colour235,fg=colour245]+ #[bg=blue,fg=white] c #[bg=colour235,fg=white] New #[fg=colour240]| #[bg=blue,fg=white] n #[bg=colour235,fg=white] Next #[fg=colour240]| #[bg=blue,fg=white] p #[bg=colour235,fg=white] Prev #[fg=colour240]| #[bg=blue,fg=white] w #[bg=colour235,fg=white] List #[fg=colour240]| #[bg=blue,fg=white] , #[bg=colour235,fg=white] Rename #[fg=colour240]|| #[fg=magenta,bold]PANE #[bg=magenta,fg=white] ^b #[bg=colour235,fg=colour245]+ #[bg=magenta,fg=white] % #[bg=colour235,fg=white] VSplit #[fg=colour240]| #[bg=magenta,fg=white] " #[bg=colour235,fg=white] HSplit #[fg=colour240]| #[bg=magenta,fg=white] z #[bg=colour235,fg=white] Zoom #[fg=colour240]| #[bg=magenta,fg=white] x #[bg=colour235,fg=white] Kill #[fg=colour240]| #[bg=magenta,fg=white] o #[bg=colour235,fg=white] Next #[fg=colour240]| #[bg=magenta,fg=white] arrows #[bg=colour235,fg=white] Navigate'
```

Let me break down what's happening:

- `#[bg=colour235]` sets the background to a dark grey, giving the bar its own visual identity separate from the main status line
- Each category label (SESSION, WINDOW, PANE) is displayed in bold with its category color
- The shortcut keys are shown with colored backgrounds, followed by their action names in white
- `|` separators visually group related shortcuts
- `||` double separators mark the boundary between categories

### The Format Pattern

Each shortcut follows the same pattern:

```text
#[bg=<color>,fg=white] <key> #[bg=colour235,fg=white] <action>
```

This creates a pill-like effect where the key stands out against the action description, making it easy to scan.

## Adding a Toggle Keybinding

Once you've internalized the shortcuts, the hints bar becomes unnecessary screen real estate. A toggle keybinding lets you hide it:

```bash
# Toggle hints bar with prefix + h
bind-key h if -F '#{==:#{status},2}' 'set -g status on' 'set -g status 2'
```

This uses tmux's conditional format: if the status is currently `2` (both lines visible), switch to `on` (single line). Otherwise, switch back to `2`.

![Toggling the hints bar with prefix + h](/images/2026/02/12/tmux-hints-toggle.gif)

## The Complete Configuration

Here's the full `~/.tmux.conf` with the hints bar, toggle, and a few other useful settings:

```bash
set -g mouse on
set -g history-limit 100000
bind-key -T copy-mode-vi y send -X copy-pipe-and-cancel "pbcopy"
set -g set-clipboard on

set -g renumber-windows on

set -g status-left '#S '
set -g status-left-length 100

# Custom keybindings
bind-key a command-prompt -p "New session name:" { new-session -d -s "%%" ; switch-client -t "%%" }
bind-key h if -F '#{==:#{status},2}' 'set -g status on' 'set -g status 2'

# --- Zellij-style shortcut hints bar ---
set -g status 2
set -g status-interval 1
set -g status-format[1] '#[bg=colour235,fg=white] #[fg=green,bold]SESSION #[bg=green,fg=black] ^b #[bg=colour235,fg=colour245]+ #[bg=green,fg=black] d #[bg=colour235,fg=white] Detach #[fg=colour240]| #[bg=green,fg=black] s #[bg=colour235,fg=white] List #[fg=colour240]| #[bg=green,fg=black] a #[bg=colour235,fg=white] New #[fg=colour240]| #[bg=green,fg=black] $ #[bg=colour235,fg=white] Rename #[fg=colour240]| #[bg=green,fg=black] ( #[bg=colour235,fg=white] Prev #[fg=colour240]| #[bg=green,fg=black] ) #[bg=colour235,fg=white] Next #[fg=colour240]|| #[fg=blue,bold]WINDOW #[bg=blue,fg=white] ^b #[bg=colour235,fg=colour245]+ #[bg=blue,fg=white] c #[bg=colour235,fg=white] New #[fg=colour240]| #[bg=blue,fg=white] n #[bg=colour235,fg=white] Next #[fg=colour240]| #[bg=blue,fg=white] p #[bg=colour235,fg=white] Prev #[fg=colour240]| #[bg=blue,fg=white] w #[bg=colour235,fg=white] List #[fg=colour240]| #[bg=blue,fg=white] , #[bg=colour235,fg=white] Rename #[fg=colour240]|| #[fg=magenta,bold]PANE #[bg=magenta,fg=white] ^b #[bg=colour235,fg=colour245]+ #[bg=magenta,fg=white] % #[bg=colour235,fg=white] VSplit #[fg=colour240]| #[bg=magenta,fg=white] " #[bg=colour235,fg=white] HSplit #[fg=colour240]| #[bg=magenta,fg=white] z #[bg=colour235,fg=white] Zoom #[fg=colour240]| #[bg=magenta,fg=white] x #[bg=colour235,fg=white] Kill #[fg=colour240]| #[bg=magenta,fg=white] o #[bg=colour235,fg=white] Next #[fg=colour240]| #[bg=magenta,fg=white] arrows #[bg=colour235,fg=white] Navigate'
```

After saving, reload it with:

```bash
tmux source-file ~/.tmux.conf
```

## Making It Your Own

This configuration is a starting point. A few ideas for customization:

- **Add more shortcuts**: Include copy-mode bindings or custom keybindings you've defined
- **Change colors**: Pick colors that match your terminal theme
- **Reorganize categories**: Group by frequency of use instead of tmux's conceptual model
- **Add a custom session keybinding**: The `bind-key a` line above adds `prefix + a` to create a new named session, which is much faster than the default `prefix + :` followed by typing `new-session`

## Conclusion

Tmux is powerful, but its keyboard-driven interface has a real learning curve. By borrowing Zellij's idea of an always-visible hints bar, you get the best of both worlds: tmux's flexibility and maturity, with a built-in reference that stays out of your way once you no longer need it.

The `prefix + h` toggle means you're never stuck with the bar permanently - it's training wheels you remove when you're ready.

## Resources

- [Tmux manual - status line](https://man.openbsd.org/tmux#status-format)
- [Zellij terminal multiplexer](https://zellij.dev/)
- [My dotfiles](https://github.com/timvw/dotfiles)
