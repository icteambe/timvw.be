---
title: "Never Lose an AI Coding Session Again"
date: 2026-02-16
draft: false
tags: ["tmux", "ai", "coding-agents", "claude-code", "opencode", "codex", "developer-tools", "productivity"]
categories: ["developer-tools"]
---

We're at an inflection point for software development. AI coding assistants -- Claude Code, OpenCode, Codex CLI -- have crossed the threshold from novelty to genuine productivity multiplier. Developers who lean into these tools are shipping products faster and at higher quality, not because the AI writes perfect code, but because it handles the grunt work while you focus on architecture, decisions, and review.

If you're running these tools seriously, you're probably running multiple of them in parallel: one per project, one per task. [tmux](https://github.com/tmux/tmux) is the natural home for this workflow -- split panes, named sessions, scriptable, works over SSH. If you're new to tmux, I wrote about [training wheels to ease the learning curve](/2026/02/12/tmux-training-wheels-a-zellij-inspired-shortcut-hints-bar/), and [wt](/2026/02/13/wt-a-better-way-to-manage-git-worktrees/) makes managing parallel worktrees painless.

With [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) and [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum), your pane layouts survive restarts. But there's a gap.

## The Problem

When tmux restarts -- reboot, crash, `kill-server` -- tmux-resurrect faithfully recreates your windows and panes. But the AI sessions that were running inside them are gone. Your Claude conversation about the refactoring plan, the OpenCode session mid-way through a feature, the Codex session that had full context on your codebase -- all lost. You're left re-launching each assistant manually and hoping you remember which session ID belongs to which project.

With three to five assistants spread across projects, this gets old fast.

## The Solution: tmux-assistant-resurrect

[tmux-assistant-resurrect](https://github.com/timvw/tmux-assistant-resurrect) is a tmux plugin that automatically saves and restores AI coding assistant sessions across tmux restarts. It hooks into tmux-resurrect's save/restore cycle so you don't have to think about it.

```
SAVE (every 5 min via continuum, or prefix + Ctrl-s)
  tmux-resurrect saves pane layouts
    -> detects running assistants by process inspection
    -> extracts session IDs via native hooks/plugins
    -> writes assistant-sessions.json

RESTORE (on tmux start, or prefix + Ctrl-r)
  tmux-resurrect restores pane layouts
    -> reads saved session IDs
    -> resumes each assistant with the correct flag:
         claude --resume <session-id>
         opencode -s <session-id>
         codex resume <session-id>
```

Three tools are supported today:

| Tool | Resume command |
|------|---------------|
| [Claude Code](https://github.com/anthropics/claude-code) | `claude --resume <id>` |
| [OpenCode](https://github.com/opencode-ai/opencode) | `opencode -s <id>` |
| [Codex CLI](https://github.com/openai/codex) | `codex resume <id>` |

## How It Works

A few technical choices that matter:

- **No wrapper scripts.** The plugin uses each tool's native hook or plugin system (Claude's `SessionStart` hook, OpenCode's plugin API, Codex's JSONL file) to track session IDs. Your workflow stays unchanged.
- **Direct process inspection.** Detection takes a single `ps` snapshot and matches child processes of each pane's shell against known binary names. No heuristics, no screen scraping -- fast and deterministic.
- **Two-guard restore safety.** Before sending a resume command to a pane, the plugin checks that (1) the pane is running a shell, and (2) no assistant is already running there. This prevents typing into TUIs or double-launching.
- **POSIX-safe quoting.** All values sent via `tmux send-keys` are safely quoted for bash, zsh, fish, and other shells.

## Installation

Install [TPM](https://github.com/tmux-plugins/tpm) if you haven't already, then add to your `~/.tmux.conf`:

```bash
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @plugin 'timvw/tmux-assistant-resurrect'

# Initialize TPM (must be last line)
run '~/.tmux/plugins/tpm/tpm'
```

Inside tmux, press `prefix + I`. TPM installs the plugin and automatically configures Claude hooks, the OpenCode plugin, and continuum's auto-save interval.

## Try It

1. **Launch** a few assistants in separate tmux panes, as you normally would.
2. **Save** with `prefix + Ctrl-s`. Inspect what was captured:

```bash
cat ~/.tmux/resurrect/assistant-sessions.json | jq .
```

3. **Kill tmux** to simulate a reboot: `tmux kill-server`
4. **Start tmux** again and **restore** with `prefix + Ctrl-r`.
5. **Verify** -- each assistant should resume its previous conversation in the correct directory.

Check the restore log if you want to see what happened:

```bash
cat ~/.tmux/resurrect/assistant-restore.log
```

## Built Over a Weekend

I built this over a weekend, and the project itself was entirely vibecoded -- designed and implemented through conversation with AI coding assistants. It has 100+ automated tests running in Docker with real CLI binaries (`@anthropic-ai/claude-code`, `opencode-ai`, `@openai/codex`). No mocks, no API keys needed for testing.

That said, it has limited real-world usage so far. Expect rough edges. It's [MIT licensed](https://github.com/timvw/tmux-assistant-resurrect/blob/main/LICENSE) and contributions are welcome.

## What's Next

Saving and restoring sessions is half the problem. When you have multiple agents running in parallel, they frequently get blocked waiting for human input -- confirmation dialogs, permission prompts, or just waiting at their input prompt. I'm working on a supervisor tool that detects blocked agents and helps you unblock them. More on that soon.

## Resources

- [tmux-assistant-resurrect on GitHub](https://github.com/timvw/tmux-assistant-resurrect)
- [Tmux Training Wheels](/2026/02/12/tmux-training-wheels-a-zellij-inspired-shortcut-hints-bar/) -- getting started with tmux
- [wt: A Better Way to Manage Git Worktrees](/2026/02/13/wt-a-better-way-to-manage-git-worktrees/) -- parallel worktrees for parallel agents
- [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) -- persist tmux layouts
- [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum) -- auto-save and auto-restore
