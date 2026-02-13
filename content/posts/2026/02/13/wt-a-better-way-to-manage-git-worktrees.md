---
title: "wt: A Better Way to Manage Git Worktrees"
date: 2026-02-13
draft: false
tags: ["git", "worktrees", "cli", "golang", "developer-tools", "ai", "coding-agents"]
categories: ["developer-tools"]
---

If you work on multiple branches simultaneously, you've probably discovered [git worktrees](https://git-scm.com/docs/git-worktree). They let you check out multiple branches at the same time in separate directories, without the overhead of cloning the repository again. The problem is that the built-in commands are tedious to use in practice. I built [wt](https://github.com/timvw/wt) to fix that.

## The Problem with Vanilla Git Worktrees

Using `git worktree` directly means you have to:

- Manually specify full paths every time you create a worktree
- Remember where you put each worktree on disk
- Manually `cd` into the new directory after creating one
- Come up with your own naming and organizational conventions
- Clean up stale worktrees by hand when branches get merged

It's powerful, but the workflow friction adds up quickly. After a while I found myself avoiding worktrees altogether simply because the ceremony around them was too high.

## Why Worktrees Matter Even More Now

With the rise of AI-assisted development, worktrees have gone from "nice to have" to practically a hard requirement. When you have multiple coding agents working in parallel — each tackling a different task or feature — they each need their own isolated checkout of the repository. You can't have two agents working on the same working directory without them stepping on each other's changes.

And it's not just about a single repository. Most real-world tasks span multiple repos: updating a shared library and the application that consumes it, changing an API and the frontend that calls it, modifying infrastructure and the service that runs on it. Each agent (or human) working on a task needs a separate worktree in *each* of those repositories.

`wt` handles this with a custom pattern that groups worktrees by task instead of by repo:

```toml
# ~/.config/wt/config.toml
strategy = "custom"
pattern = "{.worktreeRoot}/{.branch}/{.repo.Name}"
```

Use the same branch name across repositories:

```bash
cd ~/src/shared-lib
wt create feat/PROJ-123

cd ~/src/main-app
wt create feat/PROJ-123
```

And everything for that task ends up in one place:

```
~/dev/worktrees/
  feat/PROJ-123/
    shared-lib/
    main-app/
```

This makes it trivial to spin up (and tear down) an isolated workspace for each parallel stream of work. If managing worktrees manually was already tedious for one developer in one repo, imagine doing it across multiple repos for three or four parallel agents. That was the final push I needed to build proper tooling around it.

## What wt Does

`wt` wraps git worktree with an opinionated, ergonomic interface. It handles the path management, shell navigation, and lifecycle for you. The core idea is simple: you tell `wt` the branch name and it takes care of the rest.

### Creating and Navigating Worktrees

With shell integration enabled (`wt init`), creating a new worktree and navigating to it is a single command:

```bash
wt create feat/add-oauth
# You're now inside ~/dev/worktrees/my-repo/feat/add-oauth
```

Want to check out an existing remote branch? Same deal:

```bash
wt checkout fix/timeout-bug
```

Running `wt co` without arguments presents an interactive selection menu, so you can pick a branch with arrow keys instead of typing the full name.

### Checking Out Pull Requests and Merge Requests

This is one of the features I use most. Need to review a GitHub PR locally?

```bash
wt pr 42
```

That resolves the PR's branch name via the `gh` CLI and creates a worktree for it. The same works for GitLab merge requests:

```bash
wt mr 15
```

### Cleanup

When branches get merged, their worktrees become stale. Instead of tracking them down manually:

```bash
wt cleanup
```

This removes worktrees for branches that have already been merged into your main branch.

## Configurable Organization

By default, `wt` places worktrees under `~/dev/worktrees/<repo>/<branch>`. But the placement strategy is fully configurable. There are seven built-in strategies, and you can define custom patterns using Go template syntax:

```toml
# ~/.config/wt/config.toml
strategy = "custom"
pattern = "{.worktreeRoot}/{.repo.Owner}/{.repo.Name}/{.branch}"
```

Template variables include `{.repo.Name}`, `{.repo.Owner}`, `{.repo.Host}`, `{.branch}`, and even `{.env.VARNAME}` for environment variable interpolation. The last one is useful for grouping worktrees by ticket number across multiple repositories.

## Hooks

`wt` supports pre- and post-hooks for create, checkout, and remove operations. A common use case is copying environment files or installing dependencies when a new worktree is created:

```toml
[hooks]
post_create = ["cp ../.env .env", "npm install"]
```

Pre-hooks abort the operation on failure. Post-hooks only warn, so they don't block your workflow.

## Installation

The quickest way to install on macOS:

```bash
brew install timvw/tap/wt
```

After installing, set up shell integration:

```bash
wt init
```

This configures your shell (Bash, Zsh, or PowerShell) so that `wt` automatically navigates you into new worktrees.

Packages are also available for Linux (deb, rpm, AUR), Windows (Scoop, WinGet), or you can install from source with `go install github.com/timvw/wt@latest`.

## Inspiration

The original inspiration for `wt` came from [Phil Haack](https://haacked.com/)'s [`tree-me`](https://github.com/haacked/tree-me) bash script, which introduced the idea of wrapping worktree management behind a simpler interface. I liked the concept but wanted something cross-platform, with richer features like PR/MR checkout and configurable placement strategies, so I rewrote it in Go.

## Wrapping Up

`wt` has become a core part of my daily development workflow — both for my own work and as the backbone for running parallel AI coding agents. The small friction reduction of not having to think about paths and navigation adds up over dozens of branch switches per week. If you use git worktrees (or have been wanting to try them), give it a spin.

The source code is on GitHub: [github.com/timvw/wt](https://github.com/timvw/wt)
