---
title: "Tmux Training Wheels: A Zellij-Inspired Shortcut Hints Bar"
date: 2026-02-12T10:00:00+01:00
draft: false
tags: ["tmux", "terminal", "productivity", "developer-experience"]
categories: ["developer-tools"]
---

I've been using tmux since around 2010, after years of working with GNU Screen. Tmux itself has been around since [2007](https://github.com/tmux/tmux), and over the years its shortcuts have become second nature to me. But I've noticed that when colleagues see my terminal setup and want to try tmux themselves, the keyboard shortcuts are the first barrier. They end up with a cheat sheet pinned to their monitor or open in a browser tab, constantly context-switching away from the terminal.

Recently I gave [Zellij](https://zellij.dev/) a try and immediately noticed its always-visible shortcut hints bar at the bottom of the screen. It's a small thing, but it removes the need for an external reference entirely. I thought: why not bring that same idea to tmux?

## The Idea: Training Wheels for Tmux

The concept is simple: use tmux's second status line to display a color-coded bar showing the most common shortcuts, grouped by category:

- **Green** for session operations (detach, list, new, rename)
- **Blue** for window operations (new, next, prev, list, rename)
- **Magenta** for pane operations (split, zoom, kill, navigate)

Think of it as training wheels. Newcomers keep them on while learning, and once the shortcuts become muscle memory, they toggle them off with a single keybinding.

![Tmux with Zellij-style shortcut hints bar](/images/2026/02/12/tmux-status-lines.png)

## Building the Hints Bar

Tmux supports multiple status lines via `set -g status 2`. The first line (`status-format[0]`) is the default status bar showing windows. The second line (`status-format[1]`) is where the hints bar lives.

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
- `|` separators visually group related shortcuts within a category
- `||` double separators mark the boundary between categories

### The Format Pattern

Each shortcut follows the same pattern:

```text
#[bg=<color>,fg=white] <key> #[bg=colour235,fg=white] <action>
```

This creates a pill-like effect where the key stands out against the action description, making it easy to scan.

## Adding a Toggle Keybinding

For experienced users, the hints bar is unnecessary screen real estate. A toggle keybinding lets you hide it:

```bash
# Toggle hints bar with prefix + h
bind-key h if -F '#{==:#{status},2}' 'set -g status on' 'set -g status 2'
```

This uses tmux's conditional format: if the status is currently `2` (both lines visible), switch to `on` (single line). Otherwise, switch back to `2`. Press `Ctrl+b h` to toggle.

## The Complete Configuration

The full `~/.tmux.conf` is available as a [GitHub Gist](https://gist.github.com/timvw/79fd84dea6cf1c9c0d36b817f5f39144). Copy it, adjust to taste, and reload with:

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

Tmux has been around since 2007 and remains one of the most powerful terminal multiplexers available. But its keyboard-driven interface has a real learning curve, especially for people coming from GUI-based terminals. By borrowing Zellij's idea of an always-visible hints bar, newcomers get a built-in reference right where they need it, without leaving the terminal.

The `prefix + h` toggle means the bar is never permanent. It's training wheels you remove when you're ready.

## Resources

- [Complete tmux.conf (GitHub Gist)](https://gist.github.com/timvw/79fd84dea6cf1c9c0d36b817f5f39144)
- [Tmux manual - status line](https://man.openbsd.org/tmux#status-format)
- [Zellij terminal multiplexer](https://zellij.dev/)
- [Tmux on GitHub](https://github.com/tmux/tmux)
