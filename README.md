# 🌳 wtree

A single-file Bash helper for working with git worktrees in a bare-repo layout.

Instead of one checkout you switch back and forth in, you get a directory of sibling
worktrees — one per branch — that all share a single bare clone. `wtree` handles the
bookkeeping: creating them with the right upstream, listing their state, fast-forwarding
them, sweeping the dead ones, and dropping an editor or a coding agent into any of them.

```
$ wtree ls
🌳 /Users/me/work/acme

  NAME            BRANCH          STATE  SYNC
  ──────────────────────────────────────────────────────
* main            main            clean  up to date
  feature/login   feature/login   dirty  ↑2
  spike-caching   spike-caching   clean  ↓5
  old-experiment  old-experiment  clean  upstream gone

  * current   ↑ ahead   ↓ behind
```

## Install

`wtree` is one script with no dependencies beyond `git` and Bash. Put it on your `PATH`:

```bash
curl -o ~/.local/bin/wtree https://raw.githubusercontent.com/ready2know/wtree/main/wtree
chmod +x ~/.local/bin/wtree
```

Add the shell hook so `wtree cd` can actually move your shell:

```bash
eval "$(wtree shell-init)"   # in ~/.zshrc or ~/.bashrc
```

(A child process can't change its parent's directory, so `cd` is the one command that
needs a shell function wrapping it. Everything else works without the hook.)

Optional: `herdr` and `jq`, only for the `herdr-agent` command.

## Layout

`wtree init` clones into `.bare` and points a `.git` file at it, so the root directory is
the repo and every subdirectory is a worktree:

```
acme/
├── .bare/            the bare clone — all git objects live here
├── .git              file containing "gitdir: ./.bare"
├── main/             worktree on branch main
├── feature/login/    worktree on branch feature/login
└── spike-caching/    worktree on branch spike-caching
```

Worktree names map straight to directory names and branch names. `feature/login` becomes
a nested directory and a slashed branch. Absolute paths are also accepted anywhere a
worktree name is expected.

## Getting started

```bash
mkdir acme && cd acme
wtree init acme-corp/backend      # a full git URL works too
wtree add main                    # check out the default branch
wtree add feature/login           # tracks origin/feature/login if it exists,
                                  # otherwise branches off main
wtree cd feature/login
```

`init` refuses to run in a non-empty directory (the `wtree` script itself is allowed to
sit there), which keeps the layout above intact.

## Commands

| Command | What it does |
| --- | --- |
| `wtree init <owner>/<repo>` | Bare-clone into the current empty directory and wire up remote tracking |
| `wtree add <worktree>` | Create a worktree. Checks out `origin/<worktree>` if it exists, otherwise branches off main/master |
| `wtree remove <worktree>` | Remove a worktree and delete its local branch |
| `wtree ls` \| `list` | Table of worktrees with branch, dirty state and ahead/behind counts |
| `wtree sync [<worktree>]` | Fetch, then fast-forward one worktree — or all of them |
| `wtree sweep` \| `clean` | Remove worktrees whose branch is merged or whose upstream is gone |
| `wtree path [<worktree>]` | Print a worktree's path, or the repo root |
| `wtree cd <worktree>` | Jump to a worktree (needs the shell hook) |
| `wtree editor [<worktree>]` | Open a worktree in your IDE |
| `wtree run <worktree> <cmd...>` | Run a command with the worktree as its working directory |
| `wtree agent <worktree> [args...]` | Start a coding agent inside a worktree |
| `wtree herdr-agent <worktree>` | Start a named herdr agent on a worktree |
| `wtree prune` | Prune stale worktree metadata |
| `wtree shell-init` | Print the shell hook |
| `wtree help` | Full usage |

### add

```
wtree add <worktree> [-c <branch>] [-p]
  -c, --custom-branch <branch>   Base the new branch on <branch> instead of main/master
  -p, --print-path               Print only the worktree path on stdout
```

Three cases, in order: if the branch already exists locally it is checked out (and wired
up to `origin/<branch>` if it wasn't tracking anything — `clone --bare` copies remote
heads in without tracking info, which would otherwise leave `ls`/`sync`/`sweep` blind);
if only a remote branch exists it is tracked; otherwise a new branch is created off the
default branch with `--no-track`.

`-p` keeps stdout to just the path, so it composes:

```bash
cd "$(wtree add feature/login -p)"
```

### remove

```
wtree remove <worktree> [-f]
  -f, --force   Remove even if dirty, and delete an unmerged branch
```

Without `-f`, a dirty worktree or an unmerged branch is kept and reported rather than
thrown away.

### ls

Shows every worktree with its branch, whether the working tree is dirty, and how it sits
against its upstream:

| SYNC | Meaning |
| --- | --- |
| `up to date` | Level with upstream |
| `↑n` / `↓n` | Ahead / behind by n commits |
| `↑n ↓m` | Diverged |
| `no upstream` | Local-only branch |
| `upstream gone` | The remote branch it tracked was deleted |
| `detached` | Detached HEAD |
| `missing` | Directory is gone — run `wtree prune` |

Extra arguments are passed straight to `git worktree list`, so `wtree ls --porcelain`
still gives you the raw thing.

### sync

```
wtree sync [<worktree>] [-r]
  -r, --rebase   Rebase local commits instead of skipping diverged branches
```

Fetches with `--prune`, then fast-forwards each worktree. Only clean fast-forwards happen
by default — a diverged branch is reported and skipped, never merged. With `-r` diverged
branches are rebased instead, and a rebase that hits conflicts is aborted rather than
left half-applied.

### sweep

```
wtree sweep [-n] [-y] [-f]
  -n, --dry-run   Only show what would go
  -y, --yes       Skip the confirmation
  -f, --force     Include dirty worktrees and unmerged commits
```

Removes worktrees whose branch is merged into the default branch, or whose upstream was
deleted. It asks before doing anything (and refuses in a non-interactive shell unless you
pass `-y`). Never touches the default branch, the worktree you're standing in, or — unless
forced — anything dirty or holding commits that aren't in the default branch. Anything
skipped is listed with the reason.

### run

```bash
wtree run main npm install
wtree run main "npm ci && npm test"    # a single quoted arg goes through the shell
```

The command replaces the script via `exec`, so its exit code and terminal are yours. The
`▶️` header goes to stderr, leaving stdout exactly what the command produced.

### agent

```
wtree agent <worktree> [--model <m>] [--effort <e>] [args...]
```

Runs your coding agent with the worktree as its working directory. `--model` and
`--effort` must come before the worktree name; everything after it is handed to the agent
untouched. Defaults are applied only if you didn't pass the same flag yourself.

### herdr-agent

```
wtree herdr-agent <worktree> [-n <name>] [-k <kind>] [--down|--right] [--focus]
```

Starts a named agent on a worktree inside herdr, the terminal multiplexer.
With `--down`/`--right` it splits the current pane and starts the agent there. Without a
direction it looks for an idle, agent-free pane in the current tab, and falls back to
taking over the current pane. Requires `herdr` and `jq`, and only works from inside a
herdr pane.

## Environment variables

| Variable | Default | Used by |
| --- | --- | --- |
| `WTREE_EDITOR` | `webstorm` | `editor` — may carry flags, e.g. `WTREE_EDITOR="code -n"` |
| `WTREE_AGENT` | `claude` | `agent` — may carry flags, e.g. `WTREE_AGENT="claude --dangerously-skip-permissions"` |
| `WTREE_MODEL` | `opus` | `agent`, `herdr-agent` — set to `""` to pass no `--model` flag |
| `WTREE_EFFORT` | `xhigh` | `agent`, `herdr-agent` — set to `""` to pass no `--effort` flag |
