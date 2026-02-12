# Tmux Training Wheels: A Zellij-Inspired Shortcut Hints Bar

I've been using tmux since around 2010, after years of working with GNU Screen. Tmux itself has been around since [2007](https://github.com/tmux/tmux), and over the years its shortcuts have become second nature to me. But I've noticed that when colleagues see my terminal setup and want to try tmux themselves, the keyboard shortcuts are the first barrier. They end up with a cheat sheet pinned to their monitor or open in a browser tab, constantly context-switching away from the terminal.

Recently I gave [Zellij](https://zellij.dev/) a try and immediately noticed its always-visible shortcut hints bar at the bottom of the screen. It's a small thing, but it removes the need for an external reference entirely. I thought: why not bring that same idea to tmux?

---

## The Idea: Training Wheels for Tmux

The concept is simple: use tmux's second status line to display a color-coded bar showing the most common shortcuts, grouped by category:

- **Green** for session operations (detach, list, new, rename)
- **Blue** for window operations (new, next, prev, list, rename)
- **Magenta** for pane operations (split, zoom, kill, navigate)

Think of it as training wheels. Newcomers keep them on while learning, and once the shortcuts become muscle memory, they toggle them off with a single keybinding.

![Tmux with Zellij-style shortcut hints bar](tmux-status-lines.png)

*The hints bar shows color-coded shortcuts grouped by category: session (green), window (blue), and pane (magenta).*

---

## Building the Hints Bar

Tmux supports multiple status lines via `set -g status 2`. The first line is the default status bar showing windows. The second line is where the hints bar lives.

Here's the key configuration:

```
# Enable two status lines
set -g status 2
set -g status-interval 1

# Zellij-style shortcut hints on the second status line
set -g status-format[1] '...'  # (see full config below)
```

The format string uses tmux's style tags to create a pill-like effect where each shortcut key stands out against its action description:

- `#[bg=green,fg=black] d` -- the key in a colored pill
- `#[bg=colour235,fg=white] Detach` -- the action in neutral colors
- `|` separators between shortcuts, `||` between categories

The `colour235` dark grey background gives the bar its own visual identity, clearly separate from the main status line.

---

## Adding a Toggle

For experienced users, the hints bar is unnecessary screen real estate. One line gives you a toggle:

```
# Toggle hints bar with prefix + h
bind-key h if -F '#{==:#{status},2}' 'set -g status on' 'set -g status 2'
```

This checks the current state: if the status shows two lines, collapse to one. Otherwise, expand back to two. Press `Ctrl+b h` to toggle.

---

## The Complete Configuration

The full `~/.tmux.conf` is available as a [GitHub Gist](https://gist.github.com/timvw/79fd84dea6cf1c9c0d36b817f5f39144). Copy it, adjust to taste, and reload with:

```
tmux source-file ~/.tmux.conf
```

---

## Making It Your Own

This configuration is a starting point. A few ideas:

- **Add more shortcuts**: Copy-mode bindings, custom keybindings you've defined
- **Change colors**: Match your terminal theme
- **Reorganize**: Group by frequency of use instead of tmux's conceptual model
- **Custom session keybinding**: The `bind-key a` line adds `prefix + a` to create a new named session -- much faster than the default workflow

---

## Conclusion

Tmux has been around since 2007 and remains one of the most powerful terminal multiplexers available. But its keyboard-driven interface has a real learning curve, especially for people coming from GUI-based terminals. By borrowing Zellij's idea of an always-visible hints bar, newcomers get a built-in reference right where they need it, without leaving the terminal.

The `prefix + h` toggle means the bar is never permanent. It's training wheels you remove when you're ready.

---

*Originally published on [timvw.be](https://timvw.be/2026/02/12/tmux-training-wheels-a-zellij-inspired-shortcut-hints-bar/).*

**Tags:** tmux, terminal, productivity, developer experience
